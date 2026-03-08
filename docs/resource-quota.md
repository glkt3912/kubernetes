# ResourceQuota / LimitRange

## 1. 何のためにあるか

Kubernetes のデフォルトでは、Pod は無制限にリソース（CPU・メモリ）を要求できる。
複数チームや複数アプリが同一クラスターを共有する場合、1つの Namespace がリソースを使い切ると他に影響する。

```
Namespace A（本番）
Namespace B（開発）← Pod を大量起動してクラスターのメモリを使い切った
Namespace A の Pod が OOMKill される
```

**2つの仕組みで制御する**:

| リソース | 対象 | 役割 |
|---|---|---|
| **ResourceQuota** | Namespace 全体 | Namespace 内の合計使用量に上限を設ける |
| **LimitRange** | Pod / Container 個別 | 個々の Pod・Container のリソース指定にデフォルト値と上限を設ける |

---

## 2. ResourceQuota

### 何ができるか

Namespace 内で使えるリソースの **合計量** を制限する。

```
Namespace "dev" に ResourceQuota を設定:
  CPU 合計: 10 コア まで
  メモリ合計: 20Gi まで
  Pod 数: 50 まで

→ dev Namespace 内の全 Pod の requests 合計がこれを超えると
  新しい Pod の作成が拒否される
```

### 型定義

```go
// pkg/apis/core/types.go:6474
type ResourceQuotaSpec struct {
    Hard          ResourceList          // 各リソースの上限値
    Scopes        []ResourceQuotaScope  // 対象を絞るスコープ（省略可）
    ScopeSelector *ScopeSelector        // スコープのより細かい指定
}

type ResourceQuotaStatus struct {
    Hard ResourceList  // 設定した上限
    Used ResourceList  // 現在の使用量
}
```

### 制限できるリソース種別

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    # コンピュート
    requests.cpu: "10"        # CPU requests の合計
    requests.memory: 20Gi     # メモリ requests の合計
    limits.cpu: "20"          # CPU limits の合計
    limits.memory: 40Gi       # メモリ limits の合計

    # オブジェクト数
    pods: "50"                # Pod 数
    services: "20"            # Service 数
    persistentvolumeclaims: "10"  # PVC 数
    secrets: "50"             # Secret 数
    configmaps: "50"          # ConfigMap 数
    services.loadbalancers: "2"   # LoadBalancer 型 Service 数
```

### Status で使用量を確認できる

```bash
kubectl get resourcequota dev-quota -n dev -o yaml

# status:
#   hard:
#     pods: "50"
#     requests.cpu: "10"
#   used:
#     pods: "12"          ← 現在 12 Pod 動いている
#     requests.cpu: "3"   ← 現在 CPU 3 コア使用中
```

### Scope（スコープ）

ResourceQuota の対象を絞れる。

```go
// pkg/apis/core/types.go:6456
const (
    ResourceQuotaScopeTerminating    = "Terminating"    // activeDeadlineSeconds あり（Job など）
    ResourceQuotaScopeNotTerminating = "NotTerminating" // activeDeadlineSeconds なし（通常 Pod）
    ResourceQuotaScopeBestEffort     = "BestEffort"     // requests/limits なし の Pod
    ResourceQuotaScopeNotBestEffort  = "NotBestEffort"  // requests/limits あり の Pod
    ResourceQuotaScopePriorityClass  = "PriorityClass"  // 特定の PriorityClass の Pod
)
```

```yaml
# BestEffort Pod（requests/limits なし）は Pod 数だけ制限する例
spec:
  hard:
    pods: "10"
  scopes:
  - BestEffort
```

---

## 3. LimitRange

### 何ができるか

**Pod / Container 個別** に対してデフォルト値と上限・下限を設定する。

```
LimitRange を設定すると:
  requests/limits を書かなかった Container に デフォルト値を自動注入
  limits が大きすぎる Container の作成を拒否
  limits/requests の比率が大きすぎる Container を拒否
```

### 型定義

```go
// pkg/apis/core/types.go:6353
type LimitRangeItem struct {
    Type                 LimitType    // Container / Pod / PersistentVolumeClaim
    Max                  ResourceList // この種別のリソースの上限
    Min                  ResourceList // この種別のリソースの下限
    Default              ResourceList // limits のデフォルト値
    DefaultRequest       ResourceList // requests のデフォルト値
    MaxLimitRequestRatio ResourceList // limits/requests の最大比率
}
```

### 設定例

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:          # limits のデフォルト（未指定時に自動注入）
      cpu: 500m
      memory: 128Mi
    defaultRequest:   # requests のデフォルト（未指定時に自動注入）
      cpu: 100m
      memory: 64Mi
    max:              # limits の上限（これを超えたら拒否）
      cpu: "2"
      memory: 1Gi
    min:              # requests の下限（これを下回ったら拒否）
      cpu: 50m
      memory: 32Mi
    maxLimitRequestRatio:  # limits/requests の最大比率
      cpu: "4"        # limits = requests × 4 まで
```

### デフォルト値の自動注入

LimitRange が設定された Namespace に requests/limits を書かない Pod を作ると:

```yaml
# ユーザーが書いた Pod spec（resources を省略）
containers:
- name: app
  image: myapp:latest

# LimitRanger（Admission Plugin）が自動注入した結果
containers:
- name: app
  image: myapp:latest
  resources:
    requests:
      cpu: 100m
      memory: 64Mi
    limits:
      cpu: 500m
      memory: 128Mi
```

これにより requests/limits なし（BestEffort）の Pod が Namespace に紛れ込むのを防ぐ。

---

## 4. requests と limits の違い（前提知識）

```
requests: スケジューラが「この Node には空き容量があるか」を判断する値
          kubelet が保証する最低限のリソース

limits:   実際に使えるリソースの上限
          CPU は throttle（制限）、メモリは OOMKill

requests ≤ limits でなければならない
```

```
Node に空き CPU が 500m しかない状態で、
  requests.cpu: 600m の Pod を作ろうとする → Pending（スケジューリングできない）
  requests.cpu: 100m の Pod を作ろうとする → 配置できる
    → 実際に 800m 使おうとしても limits.cpu: 300m なら throttle される
```

**QoS クラス**（Quality of Service）:

| クラス | 条件 | 特徴 |
|---|---|---|
| Guaranteed | requests == limits（全リソース） | OOMKill されにくい。最優先で保護される |
| Burstable | requests < limits（一部でも設定あり） | requests 分は保証、limits まではバースト可 |
| BestEffort | requests も limits も設定なし | リソース不足時に最初に OOMKill される |

---

## 5. ResourceQuota と LimitRange の連携

LimitRange でデフォルト requests を注入することで、ResourceQuota の集計が正確になる。

```
ResourceQuota だけある場合:
  requests.cpu を書かない Pod を作る → requests = 0 とカウントされる
  → Quota 上は余裕があるように見えて、実は Node のリソースを圧迫

LimitRange で defaultRequest を設定:
  → 全 Pod に必ず requests が注入される
  → ResourceQuota の Used が実態を正確に反映する
```

**推奨パターン**: 両方をセットで設定する。

```
1. LimitRange で defaultRequest / default(limits) を設定
   → 全 Pod に requests/limits が確実に入る

2. ResourceQuota で Namespace 全体の合計を制限
   → LimitRange で注入された requests が正確にカウントされる
```

---

## 6. 内部実装：ResourceQuota Controller

### コントローラの構造

```go
// pkg/controller/resourcequota/resource_quota_controller.go:80
type Controller struct {
    rqClient   corev1client.ResourceQuotasGetter  // ResourceQuota を更新するクライアント
    rqLister   corelisters.ResourceQuotaLister    // ローカルキャッシュ
    queue      workqueue.TypedRateLimitingInterface[string]  // 通常の同期キュー
    missingUsageQueue workqueue.TypedRateLimitingInterface[string]  // 初回計算キュー
    registry   quota.Registry  // リソース種別ごとの使用量計算器
    quotaMonitor *QuotaMonitor // リソース変化を監視して再計算をトリガー
}
```

### syncResourceQuota の処理フロー

```
ResourceQuota が変更 / Pod が作成・削除
        |
        v
QuotaMonitor が検知 → replenishQuota() → queue に enqueue
        |
        v
syncResourceQuotaFromKey()
  └── syncResourceQuota()
        |
        v  registry.CalculateUsage() で Namespace 内の全リソースを集計
        |  （namespace の Pod/Service/PVC などを全件 List して合計）
        |
        v
  ResourceQuota.Status.Used を更新（apiserver 経由）
```

```go
// pkg/controller/resourcequota/resource_quota_controller.go:367
func (rq *Controller) syncResourceQuota(ctx context.Context, resourceQuota *v1.ResourceQuota) (err error) {
    // spec.Hard と status.Hard が一致しているか確認
    statusLimitsDirty := !apiequality.Semantic.DeepEqual(resourceQuota.Spec.Hard, resourceQuota.Status.Hard)
    // ...
    // 現在の使用量を再計算
    // status を更新
}
```

### Admission での強制

ResourceQuota の **強制**（作成拒否）は Controller ではなく **Admission Plugin** が行う。

```
Pod 作成リクエスト
        |
        v  ResourceQuota Admission Plugin（apiserver 内）
  Namespace の ResourceQuota を確認
        |
  (現在の Used) + (新しい Pod の requests) > Hard ?
        |
        YES → 429 Too Many Requests で拒否
        NO  → 許可（楽観的に Used をインクリメント）
        |
        v
  Pod が作成される
```

楽観的インクリメントなので、並行作成時に一時的に超過する可能性がある。
その場合 Controller が次の同期時に Status を修正する。

---

## 7. LimitRanger の内部実装

LimitRange の強制は **LimitRanger Admission Plugin** が担う。

```
pkg/admission/plugin/limitranger/admission.go
  └── Admit() / Validate()
        ├── デフォルト値の注入（Mutating フェーズ）
        └── min/max/ratio のチェック（Validating フェーズ）
```

```
Pod 作成リクエスト
        |
        v  LimitRanger Admission Plugin
  Namespace の LimitRange を取得
        |
  requests/limits が未指定 → default / defaultRequest を注入
        |
  min/max チェック → 違反なら拒否
  maxLimitRequestRatio チェック → 違反なら拒否
        |
        v
  （注入済みの requests/limits で）Pod が作成される
```

---

## 8. トラブルシュート

### Pod が作れない（ResourceQuota 超過）

```bash
# エラーメッセージの例
Error from server (Forbidden): pods "my-pod" is forbidden:
  exceeded quota: dev-quota, requested: requests.cpu=500m,
  used: requests.cpu=9500m, limited: requests.cpu=10

# 現在の使用量を確認
kubectl describe resourcequota -n dev

# 上限を一時的に増やす
kubectl patch resourcequota dev-quota -n dev \
  --patch '{"spec":{"hard":{"requests.cpu":"20"}}}'
```

### Pod が作れない（LimitRange 違反）

```bash
# エラーメッセージの例
Error from server (Forbidden): pods "my-pod" is forbidden:
  [maximum cpu usage per Container is 2, but limit is 4]

# Namespace の LimitRange を確認
kubectl describe limitrange -n dev
```

### requests/limits が意図せず注入されている

```bash
# LimitRange を確認（defaultRequest/default が設定されていないか）
kubectl get limitrange -n dev -o yaml
```

---

## 9. コードリーディングの起点

```
pkg/apis/core/types.go:6383          ← LimitRange の型定義
pkg/apis/core/types.go:6538          ← ResourceQuota の型定義

pkg/controller/resourcequota/resource_quota_controller.go
  └── Controller / syncResourceQuota() ← 使用量の再計算・Status 更新

pkg/controller/resourcequota/resource_quota_monitor.go
  └── QuotaMonitor ← リソース変化を監視して再計算をトリガー

staging/src/k8s.io/apiserver/pkg/quota/v1/
  └── registry.go ← リソース種別ごとの使用量計算インターフェース

plugin/pkg/admission/resourcequota/
  └── admission.go ← ResourceQuota の強制（Admission Plugin）

plugin/pkg/admission/limitranger/
  └── admission.go ← LimitRange の強制・デフォルト注入（Admission Plugin）
```
