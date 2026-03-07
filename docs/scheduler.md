# Scheduler の設計と実装

## 1. Scheduler とは何か

kube-scheduler は **「Pending 状態の Pod を監視し、最適な Node を選んで割り当てる」** 専用コンポーネントである。
スケジューリングの本質は「Pod の要件（CPU/メモリ/アフィニティ等）を満たす Node を絞り込み（Filter）、
その中から最も適切な Node を選ぶ（Score）」という2段階の処理だ。
Kubernetes 1.15 以降は **Scheduling Framework** と呼ばれるプラグインシステムが導入され、
スケジューリングの各ステップをプラグインとして実装・差し替えできる設計になっている。

---

## 2. 全体処理フロー

```
  Pending Pod
      │
      │  SchedulingQueue.Pop()（優先度順）
      v
  ┌────────────────────────────────────────────┐
  │          Scheduling Cycle（直列）           │
  │                                            │
  │  PreFilter  → 全 Node 共通の事前計算        │
  │  Filter     → 配置不可 Node を除外（並列）  │
  │  PostFilter → Filter 全失敗時（Preemption） │
  │  PreScore   → Score の事前計算             │
  │  Score      → 残 Node をスコアリング（並列）│
  │  NormalizeScore → スコアを 0-100 に正規化   │
  │  Reserve    → 選択 Node をキャッシュに仮予約│
  │  Permit     → 外部承認待ち（省略可）        │
  └─────────────────────┬──────────────────────┘
                        │ SuggestedHost 決定
                        v
  ┌────────────────────────────────────────────┐
  │          Binding Cycle（非同期 goroutine）  │
  │                                            │
  │  PreBind    → ボリュームバインドなど        │
  │  Bind       → Pod.spec.nodeName を書き込む  │
  │  PostBind   → 後処理（ログ・通知等）        │
  └────────────────────────────────────────────┘
```

**2サイクルに分かれている理由**:
Scheduling Cycle は直列（1 Pod ずつ）で実行されるが、
Bind（apiserver への書き込み）は時間がかかる。
Binding Cycle を非同期 goroutine に分離することで、
Bind 待ちの間も次の Pod のスケジューリングを進められる。

---

## 3. 主要コンポーネントの概念

### 3-1. SchedulingQueue：スケジューリング待ちキュー

通常の FIFO ではなく、**優先度付きキュー（Heap）** になっている。

**優先度付きキュー（Heap）とは**:
各要素に「優先度」を持たせ、**常に優先度が最も高い要素から取り出される**データ構造。
FIFO（先入れ先出し）と異なり、後から入れた要素でも優先度が高ければ先に取り出される。

```
FIFO（DeltaFIFO 等）:   入れた順に取り出す
  入: [A, B, C] → 出: A → B → C

優先度付きキュー（Heap）: 優先度順に取り出す
  入: [Pod(priority=10), Pod(priority=50), Pod(priority=1)]
  出: priority=50 → priority=10 → priority=1
```

Heap という名前は、内部実装に**ヒープ木**（親ノードが子より必ず優先度が高い二分木）を使うことに由来する。
要素の追加・取り出しがどちらも O(log n) で行えるため、大量の Pod でも高速に動作する。

SchedulingQueue では Pod の `spec.priority`（PriorityClass から設定される整数値）が優先度として使われる。

**FIFO と優先度付きキューの使い分け**:
Kubernetes では用途に応じて2種類のキューが使い分けられている。

| | DeltaFIFO | SchedulingQueue |
|---|---|---|
| 何を並べるか | オブジェクトの変更イベント | スケジューリング待ち Pod |
| 順序の基準 | 発生した時刻（先着順） | Pod の優先度（重要度順） |
| 順序を崩すと | 状態が壊れる（作成より先に削除が処理されるなど） | 重要な Pod が後回しになる |

「**イベントの正確な順序が重要な場面は FIFO**」「**重要度に応じた処理順が必要な場面は Heap**」という設計判断。

```
SchedulingQueue の内部構造:

  activeQ         ← 優先度順に並んだスケジューリング待ち Pod（Heap）
  backoffQ        ← スケジューリング失敗後バックオフ中の Pod
  unschedulablePods ← 現在のクラスタ状態では配置不可能な Pod
```

優先度は `spec.priorityClassName` で指定された PriorityClass の値で決まる。
高優先度の Pod は低優先度の Pod を **Preemption（先取り）** で追い出せる。

**Pod が backoffQ から activeQ に戻る条件**:
バックオフ時間経過後、またはクラスタに変化（Node 追加・Pod 削除等）があったとき。
この「クラスタ変化 → 再試行」の連携は `EnqueueExtensions` プラグインで管理される。

### 3-2. Scheduler.Run()：メインループ

```go
// pkg/scheduler/scheduler.go:536
func (sched *Scheduler) Run(ctx context.Context) {
    sched.SchedulingQueue.Run(logger)        // キューの内部 goroutine を起動
    go wait.UntilWithContext(ctx, sched.ScheduleOne, 0)  // スケジューリングループ
    <-ctx.Done()
    sched.SchedulingQueue.Close()
}
```

`ScheduleOne` は **1 Pod 分のスケジューリング全体**を担当する。
`wait.UntilWithContext` でループし、`NextPod()`（= キューからの取り出し）がブロッキングで待つ。

### 3-3. scheduleOnePod()：1 Pod の処理

```go
// pkg/scheduler/schedule_one.go:98
func (sched *Scheduler) scheduleOnePod(ctx context.Context, podInfo *QueuedPodInfo) {
    state := framework.NewCycleState()  // このサイクル専用の状態ストア

    // Scheduling Cycle（直列）
    scheduleResult, assumedPodInfo, status := sched.schedulingCycle(ctx, state, fwk, podInfo, ...)
    if !status.IsSuccess() {
        sched.FailureHandler(...)  // 失敗 → バックオフ or Preemption
        return
    }

    // Binding Cycle（非同期）
    go sched.runBindingCycle(ctx, state, fwk, scheduleResult, assumedPodInfo, ...)
}
```

### 3-4. CycleState：プラグイン間の状態共有

`CycleState` は **1 回のスケジューリングサイクル専用のスレッドセーフなキーバリューストア**。

```go
// staging/src/k8s.io/kube-scheduler/framework/cycle_state.go
type CycleState interface {
    Read(key StateKey) (StateData, error)
    Write(key StateKey, val StateData)
    Delete(key StateKey)
}
```

**用途の例**:

- `PreFilter` プラグインが Pod の要件を事前計算して CycleState に保存
- `Filter` プラグインが各 Node の評価時に CycleState から計算済みデータを取り出す
- 毎サイクル新しい CycleState が作られ、サイクル終了後に破棄される

---

## 4. Scheduling Framework のプラグイン拡張点

**Scheduling Framework とは何か**:
Kubernetes 1.15 で導入された、スケジューリング処理を**プラグインとして差し替え可能にする仕組み**。

導入前は「スケジューリングのロジックを変えたい」場合にスケジューラ本体のコードを直接修正する必要があり、
upstream の更新についていくのが困難だった。

```
導入前（モノリシック）:
  スケジューラ本体にすべてのロジックが埋め込まれている
  → カスタマイズ = フォークして本体を改変

導入後（Scheduling Framework）:
  スケジューリングの各ステップが「拡張点」として定義されている
  → カスタマイズ = 拡張点に対応したプラグインを実装して登録するだけ
  → スケジューラ本体には手を触れない
```

**プラグインの仕組み**:
各拡張点は Go のインターフェースとして定義されており、
プラグインはそのインターフェースを実装した構造体として作る。
1つのプラグインが複数の拡張点を実装することもできる。

```
例: NodeResourcesFit プラグイン
  → PreFilterPlugin と FilterPlugin の両方を実装
  → PreFilter で Pod のリソース要求を計算しておき
    Filter で各 Node のリソース空き容量と比較する
```

各拡張点（Extension Point）は Go のインターフェースとして定義されており、
プラグインはこの中から必要なものだけ実装する。

**拡張点ごとに独自のインターフェースがある**:
拡張点の数だけインターフェースが存在し、プラグインは必要なものだけ実装する。

```go
// 拡張点ごとに別々のインターフェース
type FilterPlugin interface {
    Filter(ctx, state, pod, nodeInfo) *Status
}
type ScorePlugin interface {
    Score(ctx, state, pod, nodeInfo) (int64, *Status)
}
type BindPlugin interface {
    Bind(ctx, state, pod, nodeName) *Status
}
// ... 拡張点の数だけインターフェースがある
```

プラグインはこの中から必要なものだけ実装すればよい。

```
NodeResourcesFit プラグイン:
  ✓ PreFilterPlugin を実装  （リソース要求の事前計算）
  ✓ FilterPlugin を実装     （Node のリソース空き確認）
  ✗ ScorePlugin は実装しない
  ✗ BindPlugin は実装しない

ImageLocality プラグイン:
  ✗ FilterPlugin は実装しない
  ✓ ScorePlugin を実装      （イメージキャッシュによる加点）
```

Framework は起動時に「このプラグインはどの拡張点を実装しているか」を
**型アサーション**で確認し、対応する拡張点にだけ登録する。
これが Go のインターフェースをプラグインシステムとして使う典型パターン。

### 拡張点一覧

| 拡張点 | フェーズ | 目的 |
|---|---|---|
| `QueueSort` | キュー | Pod の優先順位付け |
| `PreEnqueue` | キュー | キューへの追加可否（SchedulingGates） |
| `PreFilter` | Scheduling | Filter の事前計算・Node 候補の絞り込み |
| `Filter` | Scheduling | Node ごとの配置可否判定 |
| `PostFilter` | Scheduling | 全 Node が Filter 失敗時（Preemption） |
| `PreScore` | Scheduling | Score の事前計算 |
| `Score` | Scheduling | Node のスコアリング |
| `Reserve` | Scheduling | 選択 Node のキャッシュ上仮予約 |
| `Permit` | Scheduling | 外部コンポーネントの承認待ち |
| `PreBind` | Binding | Bind 前の処理（Volume バインド等） |
| `Bind` | Binding | Pod の Node への割り当て |
| `PostBind` | Binding | Bind 後の後処理 |

### Filter プラグインのインターフェース

```go
// staging/src/k8s.io/kube-scheduler/framework/interface.go:505
type FilterPlugin interface {
    Plugin
    Filter(ctx context.Context, state CycleState, pod *v1.Pod, nodeInfo NodeInfo) *Status
}
```

戻り値の `*Status` で配置可否を表現する。

```
Status.Code の意味（Filter）:

  Success                   → この Node に配置可能
  Unschedulable             → 配置不可（他の PostFilter プラグインが解決できる可能性あり）
  UnschedulableAndUnresolvable → 配置不可かつ解決不可（Preemption も試みない）
  Error                     → プラグイン内部エラー（即時リトライ）
```

### Score プラグインのインターフェース

```go
// staging/src/k8s.io/kube-scheduler/framework/interface.go:582
type ScorePlugin interface {
    Plugin
    Score(ctx context.Context, state CycleState, pod *v1.Pod, nodeInfo NodeInfo) (int64, *Status)
    ScoreExtensions() ScoreExtensions  // NormalizeScore を提供
}

type ScoreExtensions interface {
    NormalizeScore(ctx context.Context, state CycleState, pod *v1.Pod, scores NodeScoreList) *Status
}
```

`Score()` は各 Node に対して呼ばれ、0 以上の整数スコアを返す。
`NormalizeScore()` は全 Node のスコアを 0〜100 の範囲に正規化する（任意実装）。

最終スコアは **全 Score プラグインのスコアを weight 付きで合算**した値になる。

---

## 5. 組み込みプラグイン一覧

`pkg/scheduler/framework/plugins/` に全組み込みプラグインが実装されている。

### 補足: アフィニティとは

**アフィニティ（Affinity）** とは「親和性・引き付け」という意味で、
**「この Pod をどこに置きたいか（または置きたくないか）の希望条件」** を表す設定。

```
nodeAffinity（Node アフィニティ）
  → 特定ラベルを持つ Node にだけ配置したい
  例: SSD を搭載した Node にだけ配置したい
      disk-type=ssd ラベルを持つ Node を要求

podAffinity（Pod アフィニティ）
  → 特定の Pod と同じ Node に配置したい
  例: キャッシュ Pod と同じ Node にアプリ Pod を置く（通信を高速化）

podAntiAffinity（Pod アンチアフィニティ）
  → 特定の Pod とは別の Node に配置したい
  例: 同じアプリの Pod を別々の Node に分散させる（冗長性確保）
```

**required と preferred の違い**:

```
required（必須）: 条件を満たす Node がなければスケジュールしない → Filter で評価
preferred（希望）: 満たせれば嬉しいが、なければ他の Node でもよい → Score で加点
```

`NodeAffinity` プラグインと `InterPodAffinity` プラグインが Filter・Score の両方に登録されており、
`required` 条件は Filter フェーズで、`preferred` 条件は Score フェーズで評価される。

---

### Filter 系プラグイン（主要なもの）

| プラグイン | ディレクトリ | 判定内容 |
|---|---|---|
| `NodeResourcesFit` | `noderesources/` | CPU・メモリ・GPU 等のリソース要求が満たされるか |
| `NodeName` | `nodename/` | `spec.nodeName` 指定がある場合の一致確認 |
| `NodePorts` | `nodeports/` | `hostPort` の衝突確認 |
| `NodeAffinity` | `nodeaffinity/` | `nodeSelector` / `nodeAffinity` の条件確認 |
| `TaintToleration` | `tainttoleration/` | Node の Taint を Pod が Tolerate できるか |
| `InterPodAffinity` | `interpodaffinity/` | Pod 間アフィニティ/アンチアフィニティ |
| `PodTopologySpread` | `podtopologyspread/` | トポロジー分散制約 |
| `NodeUnschedulable` | `nodeunschedulable/` | `spec.unschedulable=true` の Node を除外 |

### Score 系プラグイン（主要なもの）

| プラグイン | ディレクトリ | スコアの根拠 |
|---|---|---|
| `NodeResourcesLeastAllocated` | `noderesources/` | リソース使用率が低い Node を優先（分散） |
| `NodeResourcesMostAllocated` | `noderesources/` | リソース使用率が高い Node を優先（集約） |
| `ImageLocality` | `imagelocality/` | すでにコンテナイメージをキャッシュしている Node を優先 |
| `NodeAffinity` | `nodeaffinity/` | preferred な Node に加点 |
| `InterPodAffinity` | `interpodaffinity/` | preferred なアフィニティ条件に加点 |

---

## 6. Filter フェーズ：並列処理の仕組み

Filter は全 Node に対して実行するため、**goroutine を使った並列評価**が行われる。

```
全 Node 数: 5000
   │
   ├── 並列化（parallelize パッケージ）
   │     goroutine pool で Filter プラグインを並列実行
   │
   └── 結果: feasibleNodes（Filter を通過した Node リスト）

性能最適化: percentageOfNodesToScore
  デフォルト: 50 - (Node 数 / 125) ≈ 大規模クラスタでは約 10%
  最低保証: 100 Node または全体の 5%
  → 5000 Node クラスタでは ~500 Node 評価して打ち切り
```

**なぜ全 Node を評価しないのか**:
Filter を通過した Node が一定数集まれば十分な品質のスケジューリングが可能であり、
全 Node 評価はスケジューリングのレイテンシを増大させる。
この設定は `percentageOfNodesToScore` フィールドで調整できる。

### parallelize パッケージの仕組み

```
pkg/scheduler/framework/parallelize/parallelism.go
```

```go
const DefaultParallelism int = 16  // デフォルトの goroutine 数

type Parallelizer struct {
    parallelism int  // 同時に動かす goroutine の上限
}

func (p Parallelizer) Until(ctx context.Context, pieces int,
    doWorkPiece func(index int), operation string) {

    chunkSize := chunkSizeFor(pieces, p.parallelism)
    // chunkSize = max(1, min(√pieces, pieces/parallelism))
    // 例: 5000 Node / 16 workers = 約 312 Node ずつ処理

    workqueue.ParallelizeUntil(ctx, p.parallelism, pieces, doWorkPiece, ...)
}
```

**動作イメージ**:

```
pieces = 500 Node, parallelism = 16

Node[0..31]   → goroutine 1
Node[32..63]  → goroutine 2
Node[64..95]  → goroutine 3
...
Node[480..499]→ goroutine 16

全 goroutine が並列に Filter プラグインを実行
→ 終わったら結果をまとめる
```

`chunkSize` の計算に √Node 数が使われているのは、
goroutine の起動コスト vs 並列効果のバランスを取るため。
Node 数が少なければ chunk を大きくして goroutine 数を抑え、
Node 数が多ければ細かく分割して並列度を上げる。

Score フェーズも同じ `Parallelizer.Until()` で並列実行される。

```go
// framework/runtime/framework.go
// 各 Node に対して Score プラグインを並列実行
f.Parallelizer().Until(ctx, len(nodes), func(index int) {
    nodeInfo := nodes[index]
    for _, pl := range plugins {
        score, status := pl.Score(ctx, state, pod, nodeInfo)
        // ...
    }
}, metrics.Score)
```

---

## 7. Score フェーズとスコアの合算

Filter 通過後、残 Node に対して Score プラグインが並列で実行される。

### スコアが表しているもの

Score の値（0〜100）は **「この Node がこの Pod にとってどれだけ適切か」の度合い**を表す。
値が大きいほど「この Node に置くと良い」という意味で、最終的に最高スコアの Node が選ばれる。

各プラグインが独自の観点でスコアを計算する：

```
NodeResourcesLeastAllocated（リソース分散）
  → Node の空きリソースが多いほど高スコア
  → リソース使用率が低い Node を優先することで、クラスタ全体に Pod を分散させる
  → 空き 80% → Score=80、空き 40% → Score=40

ImageLocality（イメージキャッシュ）
  → Pod が使うコンテナイメージがすでにその Node にキャッシュされていれば高スコア
  → イメージのダウンロード不要 → 起動が速くなる
  → キャッシュあり → Score=100、なし → Score=0

NodeAffinity（preferred 条件）
  → Pod の preferred nodeAffinity 条件に一致するラベルを持つ Node に加点
  → 「できれば SSD Node に置きたい」という希望が満たされる度合い
```

**weight（重み）の意味**:
プラグインごとに weight を設定することで、「どの観点を重視するか」を調整できる。
`NodeAffinity (weight: 2)` は `ImageLocality (weight: 1)` より2倍影響が大きい。

```
Node A のスコア計算例:

  NodeResourcesLeastAllocated (weight: 1): Score = 80
    → CPU・メモリの空きが多い Node（空き 80%）

  ImageLocality               (weight: 1): Score = 40
    → Pod のイメージが一部だけキャッシュされている

  NodeAffinity                (weight: 2): Score = 60
    → preferred 条件に部分的に一致

  最終スコア = (80×1 + 40×1 + 60×2) / (1+1+2) = 240/4 = 60
```

この計算を全 Node に対して行い、最終スコアが最も高い Node が選ばれる。
同スコアの場合はランダム選択。

---

## 8. Reserve と Assume：楽観的更新

### Bind とは

**Bind** は「Pod をどの Node に配置するか」という決定を **apiserver に書き込む処理**。

```
Bind が行うこと:

  Pod.spec.nodeName = "node-a" を apiserver に書き込む（PATCH リクエスト）
        │
        └── etcd に永続化される
              │
              └── node-a 上の kubelet が Watch で検知 → コンテナを起動
```

```
Bind 前: Pod.spec.nodeName = ""（未割り当て） → Pending 状態
Bind 後: Pod.spec.nodeName = "node-a"         → kubelet が起動処理を開始
```

Bind は apiserver へのネットワーク越しのリクエストなので時間がかかる。
そのため Binding Cycle を非同期 goroutine に分離し、スケジューラはすぐ次の Pod の処理に移る。

---

### 楽観的更新（Assume）とは

**楽観的更新**とは「完了を確認する前に、成功したと仮定して先に進む」設計パターン。

Bind は非同期なので、Bind 完了を待たずに次の Pod のスケジューリングが始まる。
楽観的更新なしだと二重カウントが発生する：

```
（楽観的更新なし）

  Pod X → Node A に配置決定（Bind 開始、まだ完了していない）
  Pod Y → スケジューリング開始
           Node A を評価 → "まだ空いている" とみなしてしまう
           Pod Y も Node A に配置決定 → リソースが二重にカウントされる（バグ）
```

**Assume（楽観的更新）で解決する**：

```
  Pod X → Node A に配置決定
           ↓ Bind 開始（非同期）と同時に
           sched.Cache.AssumedPod(X, "node-a")
           → キャッシュ上で "Node A に Pod X がいる" と先に反映

  Pod Y → スケジューリング開始
           Node A を評価 → "Pod X 分のリソースは使用済み" として正しく評価
           → 二重カウントが起きない
```

**失敗した場合**：

```
  Bind 成功 → キャッシュが実態に合った状態になる（そのまま）
  Bind 失敗 → sched.Cache.ForgetPod()  ← 楽観的更新を取り消す
               → Pod X は再度 Pending に戻り、再スケジュールされる
```

「楽観的」とは「どうせ成功するだろう」と仮定して先に進む、という意味。
Web フロントエンドでもよく使われるパターンで、いいねボタンを押した時に
API レスポンスを待たずに即座に UI のカウントを増やし、失敗したら元に戻す、というのが同じ発想。

Bind に失敗してもスケジューラは正しく動き続ける（Pod は再度 Pending になり再スケジュール）。

---

## 9. Preemption（先取り）

高優先度 Pod がどの Node にも配置できない場合、`PostFilter` 拡張点で **Preemption** が試みられる。

```
Pod A（高優先度）が全 Node で Filter 失敗
  │
  └── PostFilter: DefaultPreemption プラグイン
        │
        ├── どの Node から低優先度 Pod を削除すれば Pod A が配置できるか評価
        ├── 候補 Node を選択し、低優先度 Pod を Delete
        └── Pod A を nominatedNode に記録して再キュー
              → 次の Scheduling Cycle で該当 Node に配置
```

Preemption は即座に配置するのではなく、
低優先度 Pod の削除後に Pod A を再スケジューリングするため、わずかに遅延が発生する。

---

## 10. よくある疑問 Q&A

**Q: Scheduler は1台しかないが、スケール問題は起きないのか？**

Scheduling Cycle は直列（1 Pod ずつ）だが、Binding Cycle は並列。
また Filter/Score フェーズは goroutine 並列で高速化されている。
大規模クラスタでは `percentageOfNodesToScore` による評価 Node 数の削減でレイテンシを制御する。
それでも不足な場合は複数の Scheduler インスタンスを `schedulerName` で分けて運用できる。

**Q: `schedulerName` とは何か？**

Pod の `spec.schedulerName` フィールドで「どの Scheduler に処理させるか」を指定できる。
デフォルトは `"default-scheduler"`。カスタム Scheduler を運用する場合はここを変える。
`Profiles` を使うと1つの Scheduler バイナリ内で複数のプラグイン設定を持てる。

**Q: Node が追加されたら Pending Pod が即座に再スケジュールされるのか？**

`EnqueueExtensions` インターフェースを実装したプラグインが
「Node が追加された → この Pod を再試行すべき」というヒントを提供する。
これを受けて SchedulingQueue が `unschedulablePods` → `activeQ` に Pod を移動させる。

**Q: Filter プラグインの呼び出し順序は？**

登録順（`KubeSchedulerProfile` の `plugins.filter.enabled` リストの順）に呼ばれる。
1つでも `Unschedulable` を返したら、その Node への残りの Filter プラグインはスキップされる（早期終了）。

---

## 11. 次に読むべきファイル

### Scheduler のメインループ

```
pkg/scheduler/scheduler.go:536
  └── func (sched *Scheduler) Run()
      → SchedulingQueue 起動・ScheduleOne ループ開始

pkg/scheduler/schedule_one.go:98
  └── func (sched *Scheduler) scheduleOnePod()
      → Scheduling Cycle と Binding Cycle の分岐
```

### プラグインインターフェース定義

```
staging/src/k8s.io/kube-scheduler/framework/interface.go:419
  └── QueueSortPlugin, PreFilterPlugin, FilterPlugin, ScorePlugin,
      ReservePlugin, PermitPlugin, BindPlugin の定義

staging/src/k8s.io/kube-scheduler/framework/cycle_state.go:45
  └── CycleState インターフェース（プラグイン間の状態共有）
```

### 組み込みプラグインの実装

```
pkg/scheduler/framework/plugins/noderesources/fit.go
  └── NodeResourcesFit（Filter）- CPU/メモリ要求チェックの実装

pkg/scheduler/framework/plugins/tainttoleration/
  └── TaintToleration（Filter）- Taint/Toleration の判定ロジック

pkg/scheduler/framework/plugins/imagelocality/
  └── ImageLocality（Score）- イメージキャッシュによるスコアリング

pkg/scheduler/framework/plugins/defaultpreemption/
  └── DefaultPreemption（PostFilter）- Preemption の実装
```

### スケジューリングキューの実装

```
pkg/scheduler/backend/queue/scheduling_queue.go
  └── SchedulingQueue の実装（activeQ/backoffQ/unschedulablePods の管理）
```

### スケジューラの設定

```
pkg/scheduler/apis/config/types.go
  └── KubeSchedulerProfile - プロファイルごとのプラグイン設定
      → plugins.filter.enabled/disabled でプラグインの有効化を制御
```
