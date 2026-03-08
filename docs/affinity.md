# Node Affinity / Pod Affinity

## 1. 配置制御の全体像

Kubernetes には Pod をどの Node に配置するかを制御する仕組みが複数ある。

```
配置制御の仕組み（制約の強さ順）

NodeSelector          → Node のラベルで単純フィルタ（最も単純）
NodeAffinity          → Node のラベルで柔軟フィルタ（必須 or ソフト）
PodAffinity           → 他の Pod と「近くに」配置
PodAntiAffinity       → 他の Pod と「遠くに」配置
Taint / Toleration    → Node 側が Pod を拒否する（docs/taint-toleration.md 参照）
```

---

## 2. NodeSelector（シンプル版）

最も単純な Node 選択。Node のラベルに完全一致するものだけに配置。

```yaml
spec:
  nodeSelector:
    disktype: ssd        # disktype=ssd ラベルを持つ Node にのみ配置
```

柔軟性がなく、一致しなければ Pending になるだけ。
より柔軟な制御が必要なときは NodeAffinity を使う。

---

## 3. NodeAffinity

### 2種類のルール

```go
// pkg/apis/core/types.go
type NodeAffinity struct {
    // 必須条件: 満たさない Node には配置しない（フィルタ）
    RequiredDuringSchedulingIgnoredDuringExecution *NodeSelector

    // 優先条件: 満たす Node を優先するが、なければ他に配置してもよい（スコア）
    PreferredDuringSchedulingIgnoredDuringExecution []PreferredSchedulingTerm
}
```

| フィールド名 | 種別 | 意味 |
|---|---|---|
| `RequiredDuringSchedulingIgnoredDuringExecution` | 必須 | 満たさない Node には配置しない。**IgnoredDuringExecution** = 実行中に条件が外れても退去しない |
| `PreferredDuringSchedulingIgnoredDuringExecution` | ソフト | weight を使って「できれば」を表現する |

### 必須条件の例

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:         # 複数 terms は OR 条件
      - matchExpressions:        # 1つの term 内は AND 条件
        - key: topology.kubernetes.io/zone
          operator: In
          values: [us-east-1a, us-east-1b]
        - key: node-type
          operator: NotIn
          values: [spot]         # スポットインスタンスを除外
```

### 演算子の種類

| operator | 意味 |
|---|---|
| `In` | values のいずれかに一致 |
| `NotIn` | values のいずれにも一致しない |
| `Exists` | ラベルキーが存在する（values 不要）|
| `DoesNotExist` | ラベルキーが存在しない |
| `Gt` / `Lt` | 数値として比較（ラベル値が数値の場合）|

### ソフト条件の例

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80                 # 高い weight ほど優先
      preference:
        matchExpressions:
        - key: disktype
          operator: In
          values: [ssd]
    - weight: 20
      preference:
        matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: [us-east-1a]
```

weight（1〜100）のスコア合計が高い Node が選ばれる。

### 内部実装

```
pkg/scheduler/framework/plugins/nodeaffinity/node_affinity.go

NodeAffinity プラグイン（Filter + Score の両方を実装）:
  var _ fwk.FilterPlugin = &NodeAffinity{}
  var _ fwk.ScorePlugin  = &NodeAffinity{}

Filter フェーズ: Required 条件を満たさない Node を除外
Score  フェーズ: Preferred 条件の weight 合計でスコアを計算
```

---

## 4. PodAffinity / PodAntiAffinity

### 目的

**Pod 同士の位置関係**を制御する。

```
PodAffinity:     「このラベルの Pod と同じ Zone に配置したい」
PodAntiAffinity: 「同じラベルの Pod と別の Node に配置したい（HA のため）」
```

### topologyKey

「どの単位で近い/遠いを判断するか」を指定するラベルキー。

```
topologyKey: kubernetes.io/hostname              → 同じ Node（1台単位）
topologyKey: topology.kubernetes.io/zone         → 同じ Zone
topologyKey: topology.kubernetes.io/region       → 同じ Region
```

### PodAffinity の例

```yaml
# 同じ Zone に cache Pod が存在する Node を好む
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: cache         # このラベルの Pod が存在する Zone を優先
        topologyKey: topology.kubernetes.io/zone
```

### PodAntiAffinity の例（HA パターン）

```yaml
# 同じ Node に自分と同じ Pod を置かない（必須）
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-app           # 自分自身のラベルを指定
      topologyKey: kubernetes.io/hostname  # Node 単位で分散
```

これにより `replicas: 3` の Deployment で各 Pod が別々の Node に配置される。

### 型定義

```go
// pkg/apis/core/types.go
type PodAffinityTerm struct {
    LabelSelector     *metav1.LabelSelector  // 対象 Pod を選ぶラベルセレクタ
    TopologyKey       string                  // 何を「同じ場所」とみなすか
    Namespaces        []string               // 対象 Namespace（省略で同じ Namespace）
    NamespaceSelector *metav1.LabelSelector  // Namespace のラベルでも指定可能
}
```

### 内部実装

```
pkg/scheduler/framework/plugins/interpodaffinity/

filtering.go  ← Filter フェーズ: Required 条件チェック
scoring.go    ← Score フェーズ: Preferred 条件の weight 集計
plugin.go     ← プラグイン登録・PreFilter/Filter/PreScore/Score を実装
```

Filter の仕組み:

```
新しい Pod を Node X に配置しようとする
  ↓
Node X と同じ topology（Zone 等）に存在する Pod を全件チェック
  ↓
PodAffinity Required を満たさない → Node X を除外
PodAntiAffinity Required を違反  → Node X を除外
```

---

## 5. Taint / Toleration との違い

| | NodeAffinity | Taint / Toleration |
|---|---|---|
| 誰が指定するか | Pod 側 | Node 側（Taint）+ Pod 側（Toleration）|
| 目的 | 「この Node に行きたい」| 「この Node には来るな」|
| 方向 | Pod → Node | Node → Pod |
| 強さ | Required（必須）or Preferred（ソフト）| NoSchedule / PreferNoSchedule / NoExecute |

**組み合わせて使う**:

```
専用 Node（GPU 等）への配置制御:
  Taint: gpu=true:NoSchedule       ← GPU 以外の Pod を弾く
  NodeAffinity: Required gpu=true  ← GPU Node を明示的に選ぶ
```

---

## 6. よくある使用パターン

### ゾーン分散（HA）

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-app
      topologyKey: topology.kubernetes.io/zone
```

### 同じ Node に相性の良い Pod を集める（コロケーション）

```yaml
# Web サーバーを Cache と同じ Node に置く（レイテンシ削減）
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: cache
      topologyKey: kubernetes.io/hostname
```

### 特定 Node タイプへの配置

```yaml
# GPU Node にのみ配置
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: accelerator
          operator: In
          values: [nvidia-tesla-v100]
```

---

## 7. パフォーマンスの注意

PodAffinity / PodAntiAffinity は計算コストが高い。

```
NodeAffinity:
  Node のラベルだけ見ればよい → O(Node 数)

PodAffinity:
  全 Node × そのトポロジー内の全 Pod を見る → O(Node 数 × Pod 数)

大規模クラスターで PodAffinity を多用すると Scheduler が遅くなる。
PodAntiAffinity の requiredDuringScheduling は特にコストが高い。
```

---

## 8. コードリーディングの起点

```
pkg/apis/core/types.go
  :3390 NodeAffinity の型定義
  :3430 PodAffinity の型定義
  :3462 PodAntiAffinity の型定義
  :3400 PodAffinityTerm の型定義

pkg/scheduler/framework/plugins/nodeaffinity/node_affinity.go
  └── NodeAffinity プラグイン（Filter + Score）

pkg/scheduler/framework/plugins/interpodaffinity/
  └── filtering.go ← PodAffinity の Filter 実装
  └── scoring.go   ← PodAffinity の Score 実装

staging/src/k8s.io/component-helpers/scheduling/corev1/nodeaffinity/
  └── node_affinity.go ← NodeSelector / PreferredSchedulingTerms の評価ロジック
```
