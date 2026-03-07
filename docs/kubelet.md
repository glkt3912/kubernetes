# kubelet の Pod 管理ループ

## 1. kubelet とは何か

kubelet は **各 Node 上で動作するエージェント**。
apiserver から「この Node に Pod を置く」という指示（`spec.nodeName` の書き込み）を受け取り、
実際にコンテナを起動・監視・停止する役割を担う。

```
apiserver
    │
    │ Watch（自分の Node に割り当てられた Pod を監視）
    v
  kubelet（Node 上で動作）
    │
    │ CRI（Container Runtime Interface）
    v
  containerd / CRI-O（コンテナランタイム）
    │
    v
  コンテナ（実際に動いているプロセス）
```

**kubelet の主な責務**:

```
1. Pod のライフサイクル管理  → コンテナの起動・停止・再起動
2. ヘルスチェック（Probe）   → liveness / readiness / startup probe の実行
3. Node ステータス報告       → CPU・メモリ使用量・Node の状態を apiserver に報告
4. ボリューム管理            → Pod が使う PersistentVolume のマウント
5. ガベージコレクション      → 停止済みコンテナ・未使用イメージの削除
```

---

## 2. 全体処理フロー

```
apiserver
    │ Watch: 自 Node に割り当てられた Pod
    v
  syncLoop()  ← kubelet のメインループ（pkg/kubelet/kubelet.go:1944）
    │
    │ Pod の追加・更新・削除イベント
    v
  syncLoopIteration()
    │
    ├── HandlePodAdditions()    ← 新規 Pod
    ├── HandlePodUpdates()      ← 更新 Pod
    └── HandlePodRemoves()      ← 削除 Pod
          │
          v
        podWorkers.UpdatePod()  ← Pod ごとの worker goroutine に委譲
          │
          v
        SyncPod()               ← 1 Pod の同期処理（コンテナ起動等）
```

---

## 3. 主要コンポーネント

### 3-1. syncLoop：メインループ

kubelet の心臓部。`Run()` の末尾で呼ばれ、プロセスが終了するまで動き続ける。

```go
// pkg/kubelet/kubelet.go:1944
kl.syncLoop(ctx, updates, kl)
```

`syncLoop` は複数のイベントソースを `select` で待ち受ける：

```
イベントソース:
  configCh   ← apiserver / staticPod / ファイルからの Pod 設定変更
  plegCh     ← PLEG（Pod Lifecycle Event Generator）からのコンテナ状態変化
  syncCh     ← 定期リシンク（1秒ごと）
  housekeepingCh ← ハウスキーピング処理（2秒ごと）
  livenessManager / readinessManager ← Probe 結果
```

### 3-2. PLEG：コンテナ状態の変化検知

**PLEG（Pod Lifecycle Event Generator）** は、コンテナランタイムを定期的にポーリングして
「コンテナが起動した / 停止した」などの変化を検知し、イベントとして通知するコンポーネント。

```
PLEG の動作:
  1. 1秒ごとにコンテナランタイムに全コンテナの状態を問い合わせる
  2. 前回の状態と比較して変化を検出する
  3. 変化を PodLifecycleEvent として plegCh に送る
  4. syncLoop が受け取って対応する Pod を再同期する
```

**なぜ PLEG が必要か**:
Watch だけでは「コンテナがクラッシュした」などのランタイム側の変化を検知できない。
PLEG がランタイム側の変化を拾い、kubelet に通知する役割を担う。

### 3-3. podWorkers：Pod ごとの worker goroutine

Pod の追加・更新・削除を受け取ると、podWorkers が **Pod ごとに専用の goroutine** を立て、
その goroutine が `SyncPod()` を呼ぶ。

```
Pod A → goroutine A → SyncPod(Pod A)
Pod B → goroutine B → SyncPod(Pod B)  ← 並行して処理できる
Pod C → goroutine C → SyncPod(Pod C)
```

同じ Pod への複数の更新は **直列に処理される**（goroutine は1 Pod に1つだけ）。
これにより「古い更新が新しい更新を上書きする」といった競合を防ぐ。

---

## 4. SyncPod：1 Pod の同期処理

`SyncPod()` は Pod の「あるべき状態」に現状を合わせる処理本体（reconcile）。

```go
// pkg/kubelet/kubelet.go:1996
func (kl *Kubelet) SyncPod(ctx context.Context, updateType kubetypes.SyncPodType,
    pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error)
```

**SyncPod の処理ステップ**（`pkg/kubelet/kubelet.go:1973` のコメントより）:

```
1. API 用の PodStatus を生成（generateAPIPodStatus）
2. statusManager に現在のステータスを記録
3. Pod が実行可能かチェック（Admission: リソース不足等で拒否することがある）
4. データディレクトリの作成（/var/lib/kubelet/pods/<uid>/）
5. ボリュームのアタッチ・マウント待ち
6. イメージ pull シークレットの取得
7. コンテナランタイムの SyncPod を呼び出す（実際のコンテナ操作）
8. ネットワーク帯域制限の更新
```

各ステップでエラーが発生した場合はそのまま `error` を返し、
podWorkers が再スケジュールして次の呼び出しで再試行する（Reconciliation Loop の特性）。

---

## 5. CRI：コンテナランタイムとの接続

kubelet はコンテナを直接操作せず、**CRI（Container Runtime Interface）** という抽象化レイヤーを通じて
コンテナランタイムに指示を出す。

```
kubelet
    │
    │ gRPC（CRI）
    v
コンテナランタイム
  containerd    ← デフォルト。Docker の代替として普及
  CRI-O         ← Kubernetes 専用の軽量ランタイム
```

**CRI のメリット**:
kubelet のコードがランタイムの実装に依存しない。
containerd を CRI-O に切り替えても kubelet のコードを変更しなくてよい。

**実装**（`pkg/kubelet/kuberuntime/`）:

```
kuberuntime_manager.go  ← kubelet が使うランタイムマネージャ
kuberuntime_sandbox.go  ← Pod サンドボックス（ネットワーク名前空間等）の管理
kuberuntime_container.go ← コンテナの起動・停止
```

**Pod サンドボックスとは**:
同じ Pod 内のコンテナが共有するネットワーク名前空間のこと。
「pause コンテナ」とも呼ばれ、Pod の IP アドレスはこのサンドボックスに割り当てられる。

```
Pod（ネットワーク名前空間を共有）
  ├── pause コンテナ（サンドボックス。IP アドレスを保持）
  ├── コンテナ A（pause のネットワーク名前空間を使う）
  └── コンテナ B（同上）
```

---

## 6. Probe：ヘルスチェック

kubelet は Pod の各コンテナに対してヘルスチェック（Probe）を定期的に実行する。

```
実装: pkg/kubelet/prober/
  prober_manager.go  ← Probe の管理・goroutine の起動
  worker.go          ← 各コンテナの Probe を実行する goroutine
  prober.go          ← 実際の Probe 実行（HTTP/TCP/Exec）
```

**3種類の Probe**:

```
liveness probe（生存確認）:
  失敗するとコンテナを再起動する
  → デッドロックや応答不能になったコンテナを自動回復

readiness probe（準備確認）:
  失敗すると Pod を Service のエンドポイントから外す
  → 起動中・処理中で外部トラフィックを受け付けられない間だけ外す
  → コンテナは再起動されない

startup probe（起動確認）:
  起動完了を確認するまで liveness/readiness probe の実行を保留する
  → 起動に時間がかかるアプリで liveness が誤って失敗するのを防ぐ
```

**Probe の実行方法**:

```
HTTP GET   → 指定したエンドポイントに HTTP リクエストを送る（2xx/3xx = 成功）
TCP Socket → 指定したポートへの TCP 接続を試みる（接続成功 = 成功）
Exec       → コンテナ内でコマンドを実行する（exit code 0 = 成功）
gRPC       → gRPC ヘルスチェックプロトコルを使う
```

**実装の仕組み**:
コンテナごとに専用の worker goroutine が起動し、独立して Probe を実行する。
結果は `ResultManager` に格納され、syncLoop が参照して Pod の再同期をトリガーする。

---

## 7. Node ステータス報告

kubelet は定期的に自 Node の状態を apiserver に報告する。

```go
// pkg/kubelet/kubelet.go:1902
wait.JitterUntil(func() { kl.syncNodeStatus(ctx) },
    kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
```

**報告する情報**:

```
Node.status.conditions:
  Ready          → kubelet が正常に動作しており Pod を受け付けられるか
  MemoryPressure → メモリが逼迫しているか
  DiskPressure   → ディスクが逼迫しているか
  PIDPressure    → プロセス数が上限に近いか

Node.status.capacity / allocatable:
  CPU・メモリ・ストレージ・最大 Pod 数の上限と使用可能量

Node.status.addresses:
  Node の IP アドレス・ホスト名
```

**Node Lease**:
ステータス全体を毎回送ると apiserver の負荷が高くなるため、
「kubelet が生きている」というハートビートは `Lease` オブジェクト（軽量）で定期送信する。
Node が応答しなくなると Lease が更新されなくなり、Node が NotReady と判定される。

---

## 8. ガベージコレクション

kubelet は停止済みコンテナや不要なイメージを定期的に削除する。

```go
// pkg/kubelet/kubelet.go:1672
go wait.Until(func() {
    if err := kl.containerGC.GarbageCollect(ctx); err != nil { ... }
}, ContainerGCPeriod, wait.NeverStop)
```

```
コンテナ GC: 停止済みコンテナを削除（デフォルト 1 分ごと）
イメージ GC:  ディスク使用率が閾値を超えたら古いイメージを削除（デフォルト 5 分ごと）
```

---

## 9. よくある疑問 Q&A

**Q: kubelet は何を Watch しているのか？**

自 Node に割り当てられた Pod だけを Watch する（`spec.nodeName == <自分の Node 名>`）。
全 Pod を Watch すると apiserver の負荷が大きいため、フィールドセレクタで絞る。

**Q: Static Pod とは何か？**

apiserver を通さず、ファイルシステム上の YAML ファイルから直接 kubelet が管理する Pod。
`/etc/kubernetes/manifests/` に置くと自動で起動される。
etcd・apiserver・controller-manager・scheduler 自体がこの仕組みで動いている（self-hosted）。

**Q: コンテナが crash loop に入ったらどうなるか？**

liveness probe が失敗するか、コンテナが exit code 非ゼロで終了すると kubelet が再起動する。
`RestartPolicy` に従い、再起動間隔は指数バックオフで増加する（最大 5 分）。
この状態が `CrashLoopBackOff` として表示される。

**Q: 「Node が NotReady」になる条件は？**

kubelet が Node Lease を更新しなくなってから一定時間（デフォルト 40 秒）経過すると
`node-lifecycle-controller` が Node を NotReady に変更し、Pod の Eviction を開始する。

---

## 10. 次に読むべきファイル

### kubelet のメインループ

```
pkg/kubelet/kubelet.go:1828
  └── Run() - 各サブシステムの起動・syncLoop の呼び出し

pkg/kubelet/kubelet.go:1944
  └── syncLoop() - メインイベントループ（select で複数ソースを待ち受け）

pkg/kubelet/kubelet.go:1996
  └── SyncPod() - 1 Pod の同期処理（コメントにワークフロー全体が記述されている）
```

### Pod worker と状態管理

```
pkg/kubelet/pod_workers.go:264
  └── podSyncer インターフェース - SyncPod/SyncTerminatingPod/SyncTerminatedPod の定義
      Pod のライフサイクルステートマシンを理解する起点
```

### CRI（コンテナランタイム連携）

```
pkg/kubelet/kuberuntime/kuberuntime_manager.go
  └── kubeGenericRuntimeManager - CRI 経由のコンテナ操作の中核

pkg/kubelet/kuberuntime/kuberuntime_container.go
  └── startContainer() - コンテナ起動の実装

pkg/kubelet/kuberuntime/kuberuntime_sandbox.go
  └── createPodSandbox() - Pod サンドボックス（pause コンテナ）の作成
```

### Probe（ヘルスチェック）

```
pkg/kubelet/prober/prober_manager.go
  └── manager - Probe goroutine の管理

pkg/kubelet/prober/worker.go
  └── worker.run() - 各コンテナの Probe 実行ループ
```

### Node ステータス

```
pkg/kubelet/kubelet_node_status.go
  └── syncNodeStatus() - Node ステータスを apiserver に報告する処理
```
