# Taint / Toleration の仕組み

## 1. 設計思想

Taint / Toleration は **「ノードが Pod を拒否する」** という方向の制御を実現する仕組み。

NodeSelector や NodeAffinity が「Pod がノードを選ぶ」（引き付け）であるのに対し、
Taint は「ノードが望ましくない Pod を弾く」（反発）の概念。

```
NodeAffinity / NodeSelector : Pod → Node  （Pod がノードを指定）
Taint / Toleration          : Node → Pod  （Node が Pod を制限）
```

用途の典型例:
- GPU ノードに GPU を必要としない Pod を乗せたくない
- マスターノードにユーザー Pod を乗せたくない
- メンテナンス中のノードに新規 Pod を乗せたくない

---

## 2. 型定義

```
staging/src/k8s.io/api/core/v1/types.go
```

```go
type Taint struct {
    Key    string      // Taint のキー（必須）
    Value  string      // 値（省略可）
    Effect TaintEffect // NoSchedule / PreferNoSchedule / NoExecute
    TimeAdded *metav1.Time // NoExecute の場合、いつ付加されたか
}

type Toleration struct {
    Key      string             // 空 = 全キーにマッチ
    Operator TolerationOperator // Exists / Equal / Lt / Gt
    Value    string             // Operator が Equal の場合に使用
    Effect   TaintEffect        // 空 = 全 Effect にマッチ
    TolerationSeconds *int64    // NoExecute のみ: 何秒後に退去させるか
}
```

### Effect の意味

| Effect | スケジューラ | kubelet（実行中 Pod）|
|---|---|---|
| `NoSchedule` | 対応 Toleration なし → 配置拒否 | 影響なし（既存 Pod は残る）|
| `PreferNoSchedule` | できれば避ける（Soft 制約）| 影響なし |
| `NoExecute` | 配置拒否 | Toleration なし → 退去（Evict）|

### マッチングルール

Toleration が Taint にマッチする条件:

```
(Toleration.Key == Taint.Key または Toleration.Key == "")
AND
(
  Operator == Exists  → Value は見ない（ワイルドカード）
  Operator == Equal   → Toleration.Value == Taint.Value
)
AND
(Toleration.Effect == Taint.Effect または Toleration.Effect == "")
```

---

## 3. Scheduler との連携

Taint / Toleration の Filter と Score は
`pkg/scheduler/framework/plugins/tainttoleration/taint_toleration.go`
に実装されている。

### 実装される Framework インターフェース

```go
var _ fwk.FilterPlugin    = &TaintToleration{}  // 配置不可ノードを除外
var _ fwk.PreScorePlugin  = &TaintToleration{}  // Score 前の前処理
var _ fwk.ScorePlugin     = &TaintToleration{}  // ソフト制約でスコア計算
var _ fwk.EnqueueExtensions = &TaintToleration{} // 再スケジュール判定
```

### Filter フェーズ

```go
func (pl *TaintToleration) Filter(ctx context.Context,
    state fwk.CycleState, pod *v1.Pod, nodeInfo fwk.NodeInfo) *fwk.Status {

    taint, isUntolerated := v1helper.FindMatchingUntoleratedTaint(
        logger,
        node.Spec.Taints,        // ノードの Taint 一覧
        pod.Spec.Tolerations,     // Pod の Toleration 一覧
        helper.DoNotScheduleTaintsFilterFunc(), // NoSchedule/NoExecute のみ対象
        pl.enableTaintTolerationComparisonOperators,
    )
    if !isUntolerated {
        return nil  // 全 Taint を Tolerate できる → 合格
    }
    return fwk.NewStatus(fwk.UnschedulableAndUnresolvable, "...")
}
```

`DoNotScheduleTaintsFilterFunc()` は `PreferNoSchedule` を除外する。
NoSchedule と NoExecute の Taint だけを Filter の判定対象にしているのがポイント。

### Score フェーズ（ソフト制約）

```go
// PreScore: PreferNoSchedule 系の Toleration だけ事前に抽出して CycleState へ保存
func (pl *TaintToleration) PreScore(...) *fwk.Status {
    tolerationsPreferNoSchedule := getAllTolerationPreferNoSchedule(pod.Spec.Tolerations)
    cycleState.Write(preScoreStateKey, &preScoreState{...})
    return nil
}

// Score: 耐えられない PreferNoSchedule Taint の数をスコアとして返す
func (pl *TaintToleration) Score(...) (int64, *fwk.Status) {
    score := int64(countIntolerableTaintsPreferNoSchedule(...))
    return score, nil  // スコアが高い = 嫌われている → NormalizeScore で逆転
}
```

NormalizeScore で `true`（reverse）を渡しているため、
Taint が多いノードほど最終スコアが低くなり、配置が避けられる。

---

## 4. 処理フロー全体図

```
kubectl taint nodes node1 key=val:NoSchedule
        |
        v
  Node.Spec.Taints に追加 → apiserver → etcd
        |
        v  Watch イベント（UpdateNodeTaint）
  Scheduler の Informer が検知
        |
        v
  SchedulingQueue: 影響を受ける未スケジュール Pod を再キュー
  (isSchedulableAfterNodeChange で判定)
        |
        v  新しいスケジューリングサイクル
  Filter: TaintToleration.Filter()
    - NoSchedule/NoExecute Taint を Tolerate できるか？
    - できない → Unschedulable
        |
        v  (Filter を通過した場合)
  Score: TaintToleration.Score()
    - PreferNoSchedule Taint を何個 Tolerate できないか数える
    - スコアが低いノードを優先（反転）
```

---

## 5. NoExecute と TolerationSeconds

`NoExecute` の Taint が付いたノードの既存 Pod への影響は
**kubelet ではなく NodeLifecycle Controller** が管理する。

```
Node に NoExecute Taint 追加
        |
        v
  NodeLifecycle Controller が検知
        |
        v
  各 Pod の Tolerations を確認
    - Toleration なし → 即退去（Pod を Delete）
    - TolerationSeconds あり → 指定秒後に退去
    - TolerationSeconds なし（Toleration あり）→ 退去しない
```

`TolerationSeconds` の典型的な用途は
「ノードが一時的に不調な場合は N 秒待ってから退去する」という猶予設定。

---

## 6. 組み込み Taint 一覧

Kubernetes がシステムで自動付与する代表的な Taint:

| Taint | 付与タイミング | 意味 |
|---|---|---|
| `node.kubernetes.io/not-ready` | Node が NotReady | 準備できていない |
| `node.kubernetes.io/unreachable` | Node との疎通不可 | 到達不能 |
| `node.kubernetes.io/unschedulable` | `kubectl cordon` 後 | スケジュール停止 |
| `node.kubernetes.io/memory-pressure` | メモリ不足 | リソース逼迫 |
| `node.kubernetes.io/disk-pressure` | ディスク不足 | リソース逼迫 |
| `node-role.kubernetes.io/control-plane` | Control Plane ノード | マスター専用 |

---

## 7. コードリーディングの起点

```
staging/src/k8s.io/api/core/v1/types.go:4040
  └── Taint / Toleration の型定義

pkg/scheduler/framework/plugins/tainttoleration/taint_toleration.go
  └── Filter / PreScore / Score の実装

staging/src/k8s.io/component-helpers/scheduling/corev1/helpers.go
  └── FindMatchingUntoleratedTaint() - Taint/Toleration のマッチングロジック

pkg/controller/nodelifecycle/node_lifecycle_controller.go
  └── NoExecute Taint による Pod 退去の制御
```
