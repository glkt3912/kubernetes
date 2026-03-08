# PriorityClass と Preemption（プリエンプション）

## 1. なぜ必要か

デフォルトでは全 Pod の優先度は同等。リソース不足時にどの Pod を先に配置するか、または既存 Pod を退去させてでも配置するかを制御できない。

```
クラスターが満杯の状態で:

優先度なし:
  重要な本番 Pod → Pending のまま待ち続ける
  開発用 Pod が Node を占有していても退去されない

PriorityClass あり:
  本番 Pod（高優先度）→ 開発用 Pod（低優先度）を退去させて配置
```

---

## 2. PriorityClass

### 型定義

```go
// pkg/apis/scheduling/types.go:48
type PriorityClass struct {
    metav1.TypeMeta
    metav1.ObjectMeta

    Value            int32              // 優先度の数値（大きいほど高優先）
    GlobalDefault    bool               // PriorityClass 未指定 Pod のデフォルトにするか
    Description      string             // 使用ガイドライン（任意）
    PreemptionPolicy *PreemptionPolicy  // Never / PreemptLowerPriority
}
```

優先度の値の範囲:

```
-2147483648 〜 1,000,000,000  ← ユーザー定義可能な範囲
1,000,000,001 〜             ← システム予約（system-cluster-critical 等）
```

### システム組み込みの PriorityClass

```go
// pkg/apis/scheduling/types.go
const (
    HighestUserDefinablePriority = int32(1000000000)     // ユーザー定義の上限
    SystemCriticalPriority       = 2 * HighestUserDefinablePriority  // システム予約

    SystemClusterCritical = "system-cluster-critical"   // CoreDNS 等
    SystemNodeCritical    = "system-node-critical"       // kubelet 等
)
```

```bash
kubectl get priorityclass
# NAME                      VALUE        GLOBAL-DEFAULT
# system-cluster-critical   2000000000   false
# system-node-critical      2000001000   false
```

### PriorityClass の作成例

```yaml
# 本番用
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "本番ワークロード用。低優先度 Pod を退去させて配置する"
preemptionPolicy: PreemptLowerPriority  # デフォルト

---
# 開発用
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: true     # 未指定 Pod のデフォルト
description: "開発・検証用。本番 Pod に退去させられる場合がある"
preemptionPolicy: Never  # 他の Pod を退去させない
```

### Pod への適用

```yaml
spec:
  priorityClassName: high-priority  # ← PriorityClass 名を指定するだけ
  containers:
  - name: app
    image: myapp:latest
```

Admission Plugin（`plugin/pkg/admission/priority/`）が `priorityClassName` を読み取り、
`spec.priority` に数値を自動注入する。

---

## 3. Preemption（プリエンプション）

### 動作フロー

高優先度 Pod が Pending になったとき、Scheduler が低優先度 Pod を退去させて配置する仕組み。

```
高優先度 Pod が Pending（Node に空きがない）
        |
        v  Scheduler の PostFilter フェーズ
  Preemption プラグインが動作

  ① Preempt() を呼び出す
  ② 各 Node について「どの Pod を退去させれば配置できるか」を計算（DryRun）
  ③ 最もコストの低い Node・Victim の組み合わせを選択
  ④ Victim Pod を Evict（削除）
  ⑤ 高優先度 Pod の nominatedNodeName に選んだ Node をセット
  ⑥ Victim Pod の terminationGracePeriod 完了後に再スケジューリング
```

### 実装

```go
// pkg/scheduler/framework/preemption/preemption.go

type Evaluator struct {
    PluginName string
    Handler    fwk.Handle
    PodLister  corelisters.PodLister
    PdbLister  policylisters.PodDisruptionBudgetLister
    Interface  // SelectVictimsOnNode 等を提供
}

// PostFilter フェーズで呼ばれるメイン関数
func (ev *Evaluator) Preempt(ctx context.Context, state fwk.CycleState,
    pod *v1.Pod, m fwk.NodeToStatusReader) (*fwk.PostFilterResult, *fwk.Status)
```

### Victim 選択のルール

退去させる Pod（Victim）は以下の基準で選ぶ:

```
1. 優先度が低い Pod から選ぶ（高優先度 Pod は退去させない）
2. PodDisruptionBudget（PDB）を可能な限り尊重する
3. 退去後に Filter が通ることを DryRun で確認する
4. 退去 Pod 数を最小化する組み合わせを選ぶ
```

### nominatedNodeName

プリエンプション後、高優先度 Pod は即座に配置されるわけではない。

```
Victim Pod の Evict
  ↓ terminationGracePeriod（デフォルト 30 秒）が経過
  ↓ Node に空きができる
  ↓ 次のスケジューリングサイクルで高優先度 Pod が配置される
```

この間、Pod の `status.nominatedNodeName` に「予定している Node」が記録される。

```bash
kubectl get pod high-priority-pod -o yaml
# status:
#   nominatedNodeName: node-3    ← Evict 完了後にここに配置予定
#   phase: Pending
```

### preemptionPolicy: Never

`preemptionPolicy: Never` を設定すると、その Pod は他の Pod を退去させない。

```
高優先度だが Never に設定した Pod:
  Pending のまま待つ（退去はしない）
  ただし自分自身が Evict されることは防がない
```

低優先度だが退去されたくない処理（長時間バッチ等）に有用。

---

## 4. スケジューリングキューとの関係

Scheduler は優先度順に Pod をキューから取り出す。

```
SchedulingQueue（優先度付きキュー）
  [priority=1000000] 本番 Pod A  ← 先に取り出す
  [priority=1000000] 本番 Pod B
  [priority=100]     開発 Pod C
  [priority=100]     開発 Pod D
```

```
pkg/scheduler/internal/queue/scheduling_queue.go
  └── activeQ: heap（優先度順）
      → priority が高いほど先に scheduleOne() に渡される
```

---

## 5. PDB との関係

プリエンプション時も PodDisruptionBudget を尊重する（可能な限り）。

```
PDB: minAvailable: 2 の Deployment（replicas: 3）
  → 同時に退去できるのは 1 Pod まで

プリエンプション時:
  PDB 違反なしで退去できる Pod を優先的に選ぶ
  PDB 違反しないと配置できない場合は PDB 違反 Pod も退去対象（最終手段）
  → SelectVictimsOnNode() が PDB 違反数を最小化するよう最適化する
```

---

## 6. 設計上の注意

### 優先度の乱用を避ける

```
全 Pod を high-priority にすると効果がなくなる
→ PriorityClass は数種類に絞り、用途を明確に定義する

推奨例:
  system-critical: 2000000000  ← Kubernetes システムコンポーネント
  prod-critical:   1000000     ← 本番 SLO 影響ワークロード
  prod-standard:   100000      ← 本番 非クリティカル
  dev:             100         ← 開発・検証（globalDefault: true）
```

### 退去による影響を考慮する

```
低優先度 Pod が退去されたとき:
  処理途中での Evict → データ消失の可能性
  → PDB で最小稼働数を保護する
  → PreStop フックと terminationGracePeriodSeconds を適切に設定する
```

---

## 7. コードリーディングの起点

```
pkg/apis/scheduling/types.go
  :48 PriorityClass の型定義

pkg/scheduler/framework/preemption/preemption.go
  └── Evaluator / Preempt() ← プリエンプションのメインロジック

pkg/scheduler/framework/preemption/candidate.go
  └── Candidate ← プリエンプション候補（Node + 退去対象 Pod リスト）

pkg/scheduler/internal/queue/scheduling_queue.go
  └── activeQ ← 優先度順のスケジューリングキュー

plugin/pkg/admission/priority/admission.go
  └── priorityClassName → spec.priority に数値を注入する Admission Plugin
```
