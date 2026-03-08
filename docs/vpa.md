# VPA（Vertical Pod Autoscaler）

## 1. HPA との違い

```
水平スケール（HPA）: Pod を増減する
  [Pod][Pod][Pod]  →  [Pod][Pod][Pod][Pod][Pod]

垂直スケール（VPA）: Pod 1台の CPU・メモリを増減する
  [Pod CPU:1 Mem:1Gi]  →  [Pod CPU:4 Mem:4Gi]
```

| | HPA | VPA |
|---|---|---|
| スケール方向 | 水平（Pod 数） | 垂直（CPU・メモリ） |
| 向いているワークロード | ステートレス（Web サーバー等） | ステートフル・シングルトン（DB 等）|
| 即時性 | 高い（Pod を増やすだけ） | 低い（Pod の再起動が必要） |
| 同時利用 | VPA と同時使用は注意が必要 | HPA（CPU/Memory）と同時使用は非推奨 |

**VPA が向いているケース**:

```
・DB（MySQL, PostgreSQL）やシングルトンプロセス
  → 水平に増やせないので垂直で対応

・requests/limits の適切な値がわからないアプリ
  → VPA の recommendation を参考に設定値を決める

・バッチ処理（Job）で実行時間を短縮したい
  → 十分な CPU を割り当てる
```

---

## 2. VPA の3コンポーネント構成

VPA は **kubernetes/autoscaler** リポジトリで管理される外部コンポーネント。
クラスターに別途インストールする必要がある。

```
+--------------------------------------------------+
|                    VPA                           |
|                                                  |
|  +--------------+  +-----------+  +-----------+ |
|  | Recommender  |  |  Updater  |  | Admission | |
|  |              |  |           |  | Controller| |
|  | 過去データを  |  | Pod を    |  | Pod 作成時 | |
|  | 分析して      |  | 再起動して |  | に推奨値を | |
|  | 推奨値を計算  |  | 推奨値を  |  | 注入する  | |
|  |              |  | 適用する  |  |           | |
|  +--------------+  +-----------+  +-----------+ |
+--------------------------------------------------+
          ↑                  ↑              ↑
   Metrics API         VPA Object      Pod 作成 Webhook
```

### Recommender

- Metrics API（Metrics Server / Prometheus Adapter）から Pod の CPU・メモリ使用量を継続的に収集
- 過去の使用量履歴（デフォルト 8 日分）を分析して推奨 requests/limits を計算
- 計算結果を **VPA オブジェクトの Status** に書き込む

```yaml
# VPA の Status に推奨値が書き込まれる
status:
  recommendation:
    containerRecommendations:
    - containerName: app
      lowerBound:
        cpu: 100m
        memory: 128Mi
      target:              # Recommender が推奨する最適値
        cpu: 500m
        memory: 512Mi
      upperBound:
        cpu: "1"
        memory: 1Gi
      uncappedTarget:      # LimitRange を無視した場合の推奨値
        cpu: 600m
        memory: 600Mi
```

### Updater

- VPA の `updateMode` が `Auto` または `Recreate` の場合に動作
- 現在動いている Pod の requests が推奨値から大きく外れている場合、Pod を **Evict（追い出す）**
- PodDisruptionBudget を尊重して、一度に Evict する Pod 数を制限
- Pod が削除されると Deployment/ReplicaSet が新しい Pod を作成する
- 新しい Pod 作成時に Admission Controller が推奨値を注入する

### Admission Controller

- Pod 作成リクエストを Mutating Webhook でインターセプト
- VPA オブジェクトの Status から推奨値を取得
- Pod の `resources.requests` を推奨値で上書きして注入する

```
kubectl apply (Deployment)
    ↓
ReplicaSet が Pod を作成
    ↓
VPA Admission Controller（Mutating Webhook）
    ↓ resources.requests を推奨値に書き換える
    ↓
Pod が実際に requests: cpu=500m で起動する
```

---

## 3. VPA オブジェクト

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:               # どの Deployment に適用するか
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"     # Off / Initial / Recreate / Auto
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:          # 推奨値の下限（これより小さくならない）
        cpu: 100m
        memory: 128Mi
      maxAllowed:          # 推奨値の上限（これより大きくならない）
        cpu: "4"
        memory: 8Gi
      controlledResources: ["cpu", "memory"]  # 対象リソース
      controlledValues: RequestsAndLimits     # requests のみ or 両方
```

### updateMode の種類

| モード | 動作 |
|---|---|
| `Off` | 推奨値の計算のみ。Pod への適用なし。推奨値を参考に手動設定する用途 |
| `Initial` | Pod 作成時のみ注入。起動中の Pod は変更しない |
| `Recreate` | 推奨値から外れた Pod を Evict して再作成（起動中 Pod も更新対象） |
| `Auto` | 現在は `Recreate` と同等。将来的にインプレース更新対応予定 |

**推奨は `Off` または `Initial` から始める**:
いきなり `Auto` にすると Pod が予期せず再起動されるリスクがある。

---

## 4. インプレース更新（In-Place Pod Vertical Scaling）

Kubernetes 1.27+ で α 機能として追加（`InPlacePodVerticalScaling` feature gate）。

従来の VPA の課題:

```
requests を変更したい
  ↓ Pod を削除して再作成が必要（Evict → 新規作成）
  ↓ 再起動によるダウンタイム・セッション切断が発生
```

インプレース更新:

```
requests を変更したい
  ↓ Pod を削除せずに CPU・メモリを変更できる
  ↓ Container のリソース制限をカーネルに直接反映（cgroup を更新）
  ↓ ダウンタイムなし
```

```go
// Pod Spec の resizePolicy でリソース変更時の動作を制御
type ContainerResizePolicy struct {
    ResourceName  ResourceName        // cpu or memory
    RestartPolicy ResourceResizeRestartPolicy  // NotRequired / RestartContainer
}
```

---

## 5. HPA との同時使用の注意

**CPU/Memory メトリクスを対象にした HPA と VPA の同時使用は非推奨**。

理由:

```
VPA が requests.cpu を 100m → 500m に変更
  ↓
HPA は requests.cpu を基準に「使用率」を計算している
  ↓
requests が増えると使用率が下がる
  ↓
HPA が Pod 数を減らしてしまう（意図しないスケールダウン）
```

**安全な組み合わせ**:

```
HPA のメトリクス: カスタムメトリクス（RPS, QueueLength など）
VPA: CPU/Memory の requests 最適化

→ HPA と VPA が互いに干渉しない
```

---

## 6. VPA のメリット・デメリット

### メリット

```
✓ requests の適切な値を自動で学習・設定できる
✓ リソースの過剰確保（Overprovisioning）を削減できる
✓ スケールアウトできないワークロードでもリソースを適応させられる
✓ Off モードで「現在の設定が適切かどうかのレポート」として使える
```

### デメリット

```
✗ Pod の再起動が伴う（Auto/Recreate モード）
✗ 推奨値の計算に時間がかかる（数分〜数時間）
✗ 急激な負荷スパイクへの対応が遅い
✗ HPA と同時使用に制約がある
✗ StatefulSet への適用は特に注意が必要
```

---

## 7. コードリーディングの起点

VPA の実装は kubernetes/autoscaler リポジトリにある。

```
github.com/kubernetes/autoscaler/vertical-pod-autoscaler/

pkg/recommender/
  └── main.go → input/（メトリクス収集）→ logic/（推奨値計算）→ output/（Status 更新）

pkg/updater/
  └── main.go → logic/updater.go → eviction/（Pod の Evict）

pkg/admission-controller/
  └── main.go → logic/vpa_pod_patcher.go（requests の注入）
```

Kubernetes 本体側の関連コード:

```
staging/src/k8s.io/api/core/v1/types.go
  └── ResourceRequirements（requests / limits の型定義）

staging/src/k8s.io/api/core/v1/types.go
  └── ContainerResizePolicy（インプレース更新の設定）

pkg/kubelet/kuberuntime/kuberuntime_manager.go
  └── インプレース更新時の cgroup 変更処理
```
