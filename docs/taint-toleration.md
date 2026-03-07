# Taint / Toleration の仕組み

## 1. 一言で言うと

**Taint** = ノードに付ける「お断りタグ」
**Toleration** = Pod が持つ「このタグは無視していいよ」という許可証

```
Node A（GPU 専用）
Taint: gpu=true:NoSchedule
  ↓ 「GPU 専用。許可なき Pod はお断り」

普通の Pod → Toleration なし → 配置できない
GPU Pod   → Toleration あり → 配置できる
```

---

## 2. Taint の付け方

```bash
kubectl taint nodes node1 gpu=true:NoSchedule
```

形式は `キー=値:Effect`。

```
gpu   =   true   :   NoSchedule
↑         ↑          ↑
キー       値          Effect（拒否したときどうするか）
```

---

## 3. Effect の 3 種類

|  | 新規 Pod | 既存 Pod |
|---|---|---|
| `NoSchedule` | 拒否 | そのまま |
| `PreferNoSchedule` | できれば拒否 | そのまま |
| `NoExecute` | 拒否 | 追い出す |

### NoSchedule

新しく来る Pod を置かない。今いる Pod はそのまま動き続ける。

### PreferNoSchedule

できれば置かない。他のノードに空きがなければしかたなく置く（ソフト制約）。

### NoExecute

新規 Pod を置かない、かつ今いる Pod も削除する。
ノードが壊れたときなどに Kubernetes が自動で付ける。

---

## 4. Toleration（許可証）

Pod の spec に書く。Taint を無視するための設定。

```yaml
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### Taint との対応

```
Node の Taint          Pod の Toleration
gpu = true : NoSchedule
↑     ↑       ↑
key   value   effect    ← この3つが一致したら「マッチ」→ 配置できる
```

### Operator の 2 種類

```yaml
# Equal: 値まで一致する必要がある
- key: "gpu"
  operator: "Equal"
  value: "true"   # gpu=true にだけマッチ

# Exists: キーがあればOK（値は問わない）
- key: "gpu"
  operator: "Exists"  # gpu=anything にマッチ
```

### 全ての Taint を許容

```yaml
- operator: "Exists"  # key も effect も空 = 全部許容
```

kube-proxy などシステム Pod がこれを使っている。

---

## 5. 全体の流れ

```
① kubectl taint nodes node1 gpu=true:NoSchedule
        |
        v  Node.Spec.Taints に追加 → etcd に保存

② kubectl apply -f pod.yaml
        |
        v  Scheduler が配置先を探す

③ Filter フェーズ（TaintToleration プラグイン）
  node1 を候補にしようとする
        |
        v  node1 の Taint を確認
  Pod の Toleration でカバーできない Taint が 1 つでもある？
    YES → node1 はアウト（Unschedulable）
    NO  → node1 は通過

④ Score フェーズ
  PreferNoSchedule の Taint を Tolerate できない数が多いノードほど
  スコアが低くなる → 避けられる
```

---

## 6. NoExecute と TolerationSeconds（猶予時間）

NoExecute Taint が付いたノードから既存 Pod を退去させるのは
**NodeLifecycle Controller**（Scheduler ではない）。

```
Node が突然 NotReady になった
        |
        v  NodeLifecycle Controller が自動で付与
  node1: NoExecute Taint が付く
        |
        v  Pod ごとに判定
  Toleration なし           → 即座に削除
  tolerationSeconds: 300    → 300秒（5分）待ってから削除
  Toleration あり（秒なし） → ずっと残る
```

### TolerationSeconds を使う理由

ネットワークの一時的な揺れで NotReady になることがある。
すぐ Pod を消すと不要な再起動が起きる。
数分待てば自然に復帰することも多い。

```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300  # 5分待ってから退去
```

---

## 7. Kubernetes が自動で付与する Taint

| Taint | いつ付く | Effect |
|---|---|---|
| `node.kubernetes.io/not-ready` | Node が NotReady | NoExecute |
| `node.kubernetes.io/unreachable` | Node と疎通できない | NoExecute |
| `node.kubernetes.io/unschedulable` | `kubectl cordon` 後 | NoSchedule |
| `node.kubernetes.io/memory-pressure` | メモリ不足 | NoSchedule |
| `node-role.kubernetes.io/control-plane` | Control Plane ノード | NoSchedule |

---

## 8. コードリーディングの起点

```
staging/src/k8s.io/api/core/v1/types.go:4040
  └── Taint / Toleration の型定義

pkg/scheduler/framework/plugins/tainttoleration/taint_toleration.go
  └── Filter（配置拒否）・Score（ソフト制約）の実装

staging/src/k8s.io/component-helpers/scheduling/corev1/helpers.go
  └── FindMatchingUntoleratedTaint() - マッチング判定のロジック

pkg/controller/nodelifecycle/node_lifecycle_controller.go
  └── NoExecute Taint による Pod 退去の制御
```
