# HPA（Horizontal Pod Autoscaler）

## 1. 一言で言うと

**負荷に応じて Pod の数を自動で増減させる仕組み**。

```
水平スケール（HPA）: Pod を増やす / 減らす（横方向）
  Pod  Pod  Pod  →  Pod  Pod  Pod  Pod  Pod
  （3台）              （5台に増えた）

垂直スケール（VPA）: Pod 1台あたりの CPU・メモリを増やす（縦方向）
  Pod        →  Pod
  CPU: 1         CPU: 4
```

HPA は Deployment / ReplicaSet / StatefulSet の `replicas` を自動で書き換えることでスケールを実現する。

---

## 2. 全体の仕組み

```
① Metrics Server が各 Pod の CPU 使用率を収集
       ↓
② HPA Controller が定期的に確認（デフォルト 15 秒）
       ↓
③ 目標値（例: CPU 使用率 50%）と現在値を比較
       ↓
④ 多ければ Pod を増やす / 少なければ Pod を減らす
       ↓
⑤ Deployment の replicas を書き換える
       ↓
⑥ ReplicaSet が Pod を増減させる
```

HPA Controller は通常のコントローラパターン（Informer + WorkQueue + Reconcile Loop）で実装されている。

---

## 3. 型定義

```
pkg/apis/autoscaling/types.go
staging/src/k8s.io/api/autoscaling/v2/types.go
```

```go
type HorizontalPodAutoscaler struct {
    Spec   HorizontalPodAutoscalerSpec
    Status HorizontalPodAutoscalerStatus
}

type HorizontalPodAutoscalerSpec struct {
    ScaleTargetRef CrossVersionObjectReference  // スケール対象（Deployment など）
    MinReplicas    *int32                       // 最小 Pod 数（デフォルト 1）
    MaxReplicas    int32                        // 最大 Pod 数
    Metrics        []MetricSpec                 // スケールの指標
}
```

設定例：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    kind: Deployment
    name: my-app
  minReplicas: 2     # 最小 Pod 数
  maxReplicas: 10    # 最大 Pod 数
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # CPU 使用率 50% を目標に維持
```

---

## 4. レプリカ数の計算

```go
// pkg/controller/podautoscaler/replica_calculator.go
// GetResourceReplicas: CPU・メモリ使用率からレプリカ数を計算する
```

**計算式**：

```
必要な Pod 数 = ceil（現在の Pod 数 × 現在の使用率 / 目標使用率）

例:
  現在 3台、CPU 使用率 90%、目標 50% の場合:
  ceil(3 × 90 / 50) = ceil(5.4) = 6台 に増やす
```

複数のメトリクスを指定した場合は**最大値を採用**する：

```go
// computeReplicasForMetrics() より
if replicaCountProposal > replicas {
    replicas = replicaCountProposal  // 各メトリクスの計算結果のうち最大を採用
}
```

```
CPU → 6台が必要
メモリ → 4台が必要
→ 6台を採用（保守的なスケールアップ）
```

---

## 5. スケールアップの上限制限

急激なスケールアップを防ぐため、1回のスケールで増やせる上限がある。

```go
// pkg/controller/podautoscaler/horizontal.go
scaleUpLimitFactor  = 2.0  // 現在の台数の2倍まで
scaleUpLimitMinimum = 4.0  // ただし最低でも4台は追加できる
```

```
現在 3台の場合:
  1回でスケールできる上限 = max(3 × 2, 3 + 4) = max(6, 7) = 7台
  → 一気に 100台にはならない
```

---

## 6. スケールダウンの安定化

スケールダウンは**慎重**に行う。一時的な負荷低下で即座に Pod を減らすと、
すぐまた負荷が上がって不安定になる（フラッピング）を防ぐため。

```go
// HorizontalController のフィールド
downscaleStabilisationWindow time.Duration  // デフォルト 5分
```

```
スケールアップ: 即座に実行
スケールダウン: 直近 5分間の推奨値のうち最大値を採用してから実行
               → 5分間ずっと「減らすべき」と判断されなければ実際には減らさない
```

---

## 7. メトリクスの種類

```go
// autoscalingv2.MetricSpec の Type フィールド
switch spec.Type {
case ResourceMetricSourceType:   // CPU・メモリ（Pod の requests に対する割合）
case PodsMetricSourceType:       // Pod が公開するカスタムメトリクス
case ObjectMetricSourceType:     // Kubernetes オブジェクトのメトリクス（Ingress のリクエスト数など）
case ExternalMetricSourceType:   // クラスター外のメトリクス（SQS キュー長など）
case ContainerResourceMetricSourceType: // 特定コンテナのリソース使用率
}
```

| 種類 | 例 |
|---|---|
| Resource | CPU 使用率 50%・メモリ使用率 80% |
| Pods | 1 Pod あたりのリクエスト数 100rps |
| Object | Ingress の接続数 1000 |
| External | SQS キューの待機メッセージ数 100 |

---

## 8. Metrics Server とは

HPA が参照するメトリクスを収集・提供するコンポーネント。

```
各 Node の kubelet（cAdvisor）
  ↓ 各 Pod の CPU・メモリ使用量を収集
Metrics Server（集約して apiserver に提供）
  ↓
HPA Controller が取得
```

```bash
# メトリクスの確認
kubectl top pods
kubectl top nodes
```

Metrics Server は Kubernetes 本体には含まれない。別途インストールが必要。
カスタムメトリクス（Pods / Object / External）には Prometheus Adapter などが必要。

---

## 9. HPA が動かない場合

```
よくある原因:
  1. Metrics Server が未インストール
     → kubectl top pods でエラーになる

  2. resources.requests が未設定
     → CPU 使用率は「requests に対する割合」なので requests がないと計算できない

  3. minReplicas / maxReplicas の範囲外
     → 計算結果がこの範囲を超えても min/max でクリップされる
```

---

## 10. コードリーディングの起点

```
pkg/controller/podautoscaler/horizontal.go
  └── HorizontalController - HPA コントローラの本体
  └── reconcileAutoscaler() - 1つの HPA の Reconcile 処理
  └── computeReplicasForMetrics() - 複数メトリクスからレプリカ数を計算

pkg/controller/podautoscaler/replica_calculator.go
  └── ReplicaCalculator - レプリカ数の計算ロジック
  └── GetResourceReplicas() - CPU・メモリのレプリカ数計算

pkg/apis/autoscaling/types.go
  └── HorizontalPodAutoscaler の型定義

staging/src/k8s.io/api/autoscaling/v2/types.go
  └── External バージョンの型定義（MetricSpec の種類）
```
