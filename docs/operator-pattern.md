# Operator パターン

## 1. Operator とは何か

**Operator = CRD + カスタムコントローラ**。

Kubernetes のコントローラパターン（Reconciliation Loop）を使って、
アプリケーション固有の運用知識をコードに埋め込む設計パターン。

```
通常のコントローラ（Kubernetes 本体）:
  Deployment Controller → Pod を指定数維持する
  ReplicaSet Controller → replicas を管理する

Operator（ユーザーが作る）:
  MySQLCluster Controller → MySQL のレプリカセットを管理する
  EtcdCluster Controller  → etcd クラスターのスケールアップ・バックアップ・更新を管理する
  PrometheusRule Controller → Prometheus の alerting rules を動的に反映する
```

**「運用者の知識をコードにする」**:

手動でやっていたことを Operator に任せる。

```
手動運用:
  MySQL のメジャーバージョンアップ
    1. レプリカを一台ずつ止める
    2. バックアップを取る
    3. バイナリを更新して起動
    4. レプリカの同期を確認
    5. プライマリに昇格
    6. 繰り返す

Operator 化:
  MySQLCluster.spec.version: "8.0" → "8.1" に変更するだけ
  Operator が上記の手順を自動実行する
```

---

## 2. Operator の構成要素

```
+-------------------------------+
|          Operator             |
|                               |
|  +----------+  +-----------+ |
|  |   CRD    |  | Controller| |
|  |          |  |           | |
|  | 新しい    |  | CRD を    | |
|  | リソース  |  | Watch して | |
|  | 種別を    |  | あるべき  | |
|  | 定義する  |  | 状態に    | |
|  |          |  | 収束させる | |
|  +----------+  +-----------+ |
+-------------------------------+
```

### CRD（CustomResourceDefinition）

新しいリソース種別の「スキーマ」を定義する。

```yaml
# MySQLCluster という新しいリソースを定義
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mysqlclusters.mysql.example.com
spec:
  group: mysql.example.com
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
              version:
                type: string
              storageSize:
                type: string
  scope: Namespaced
  names:
    plural: mysqlclusters
    singular: mysqlcluster
    kind: MySQLCluster
```

### カスタムリソース（CR）

CRD で定義した新しいリソースを実際に作成する。

```yaml
apiVersion: mysql.example.com/v1alpha1
kind: MySQLCluster
metadata:
  name: my-mysql
spec:
  replicas: 3
  version: "8.0"
  storageSize: "10Gi"
```

`kubectl apply -f` するとこの CR が etcd に保存され、
Controller が Watch して対応する StatefulSet / Service / PVC などを作成する。

### カスタムコントローラ

CRD を Watch して Reconciliation Loop を実行する。

```
MySQLCluster CR が作成される
        |
        v
Controller が Watch でイベントを受信
        |
        v
syncMySQLCluster() ← Reconcile 関数
  ├── StatefulSet が存在するか確認 → なければ作成
  ├── replicas が一致しているか確認 → 違えば更新
  ├── version が変わっているか確認 → ローリングアップデートを実行
  └── Status を更新（readyReplicas, phase など）
```

---

## 3. Reconciliation Loop の設計

Operator のコントローラも Kubernetes 本体のコントローラと同じパターンを使う。

```go
// 典型的な Operator の reconcile 関数
func (r *MySQLClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. CRD オブジェクトを取得
    cluster := &mysqlv1.MySQLCluster{}
    if err := r.Get(ctx, req.NamespacedName, cluster); err != nil {
        if apierrors.IsNotFound(err) {
            return ctrl.Result{}, nil  // 削除済み → 何もしない
        }
        return ctrl.Result{}, err
    }

    // 2. あるべき状態を計算
    desiredStatefulSet := buildStatefulSet(cluster)

    // 3. 現状を確認
    existingStatefulSet := &appsv1.StatefulSet{}
    if err := r.Get(ctx, req.NamespacedName, existingStatefulSet); err != nil {
        if apierrors.IsNotFound(err) {
            // 4. なければ作成
            return ctrl.Result{}, r.Create(ctx, desiredStatefulSet)
        }
        return ctrl.Result{}, err
    }

    // 5. 差分があれば更新
    if needsUpdate(existingStatefulSet, desiredStatefulSet) {
        return ctrl.Result{}, r.Update(ctx, desiredStatefulSet)
    }

    // 6. Status を更新
    cluster.Status.ReadyReplicas = existingStatefulSet.Status.ReadyReplicas
    return ctrl.Result{}, r.Status().Update(ctx, cluster)
}
```

**設計の原則**（Kubernetes 本体のコントローラと同じ）:

```
冪等性: 何度 reconcile を呼んでも同じ結果になる
  → 「StatefulSet が存在しなければ作る」（二重作成にならない）

あるべき状態との差分を解消する
  → 「今どうなっているか」より「どうあるべきか」を宣言する

エラーは再キューで回収する
  → return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
```

---

## 4. Status サブリソースによる状態報告

Operator は CR の Status に現在の状態を書き込む。

```yaml
# kubectl get mysqlcluster my-mysql -o yaml
status:
  phase: Running           # Pending / Running / Failed
  readyReplicas: 3
  currentVersion: "8.0"
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-03-08T00:00:00Z"
  - type: Updating
    status: "False"
```

Status は **spec とは別のサブリソース** として更新する。
これにより spec への権限なしに Status だけ更新できる。

```go
// spec の更新
r.Update(ctx, cluster)

// status の更新（別の API）
r.Status().Update(ctx, cluster)
```

---

## 5. Operator の成熟度レベル（Capability Model）

Red Hat が定義した Operator の機能成熟度指標。

| Level | 機能 | 例 |
|---|---|---|
| Level 1 | 基本インストール | CR を作ると必要なリソースがデプロイされる |
| Level 2 | アップグレード | CR の version を変えるとローリングアップデートが走る |
| Level 3 | フルライフサイクル | バックアップ・リストア・障害復旧を自動化 |
| Level 4 | 深い観測性 | メトリクス・アラート・ログを自動設定 |
| Level 5 | 自動パイロット | 負荷に応じて自動でスケール・チューニング |

---

## 6. Operator フレームワーク

Operator の実装を効率化するフレームワーク。

### controller-runtime（kubebuilder）

Kubernetes 本体も使う controller-runtime を利用したフレームワーク。

```
github.com/kubernetes-sigs/controller-runtime
github.com/kubernetes-sigs/kubebuilder
```

```go
// kubebuilder を使った典型的な Operator の main.go
func main() {
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
    })

    // コントローラを登録
    (&MySQLClusterReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr)

    // マネージャを起動（Informer / LeaderElection / Healthz を管理）
    mgr.Start(ctrl.SetupSignalHandler())
}

// コントローラの登録
func (r *MySQLClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&mysqlv1.MySQLCluster{}).       // 監視対象の CR
        Owns(&appsv1.StatefulSet{}).         // 子リソース（StatefulSet）の変化も watch
        Owns(&corev1.Service{}).             // 子リソース（Service）の変化も watch
        Complete(r)
}
```

**`Owns()`**: コントローラが管理する子リソース（OwnerReference で紐付いたもの）を
Watch して、変化があったときに親 CR の Reconcile をトリガーする。

### Operator SDK

kubebuilder ベースにさらに Ansible / Helm による Operator 作成もサポートする。

```
github.com/operator-framework/operator-sdk
```

---

## 7. Operator の実装パターン

### OwnerReference で子リソースを管理

Operator が作成した子リソースには OwnerReference を設定する。

```go
// StatefulSet に MySQLCluster の OwnerReference を設定
controllerutil.SetControllerReference(cluster, statefulSet, r.Scheme)

// 設定されると:
// statefulSet.OwnerReferences = [{
//     APIVersion: "mysql.example.com/v1alpha1",
//     Kind:       "MySQLCluster",
//     Name:       "my-mysql",
//     UID:        "...",
//     Controller: true,
// }]

// MySQLCluster が削除されると StatefulSet も自動削除される（GC が処理）
```

### Finalizer でクリーンアップ

外部リソース（クラウドのオブジェクトストレージ等）のクリーンアップに使う。

```go
const myFinalizer = "mysql.example.com/finalizer"

func (r *MySQLClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 削除中かどうかを確認
    if !cluster.DeletionTimestamp.IsZero() {
        // Finalizer があれば外部リソースをクリーンアップ
        if controllerutil.ContainsFinalizer(cluster, myFinalizer) {
            if err := r.cleanupExternalResources(cluster); err != nil {
                return ctrl.Result{}, err
            }
            controllerutil.RemoveFinalizer(cluster, myFinalizer)
            r.Update(ctx, cluster)
        }
        return ctrl.Result{}, nil
    }

    // 通常処理: Finalizer を追加
    if !controllerutil.ContainsFinalizer(cluster, myFinalizer) {
        controllerutil.AddFinalizer(cluster, myFinalizer)
        r.Update(ctx, cluster)
    }

    // ... reconcile ロジック
}
```

### 条件（Conditions）で状態を表現

```go
// Conditions の標準的な使い方
meta.SetStatusCondition(&cluster.Status.Conditions, metav1.Condition{
    Type:               "Ready",
    Status:             metav1.ConditionTrue,
    Reason:             "AllReplicasReady",
    Message:            "All 3 replicas are ready",
    LastTransitionTime: metav1.Now(),
})
```

---

## 8. 有名な Operator の例

| Operator | 対象 | 主な機能 |
|---|---|---|
| prometheus-operator | Prometheus | ServiceMonitor/PrometheusRule → 設定自動生成 |
| cert-manager | TLS 証明書 | Let's Encrypt 証明書の自動取得・更新 |
| zalando/postgres-operator | PostgreSQL | クラスター作成・フェイルオーバー・バックアップ |
| strimzi | Apache Kafka | Kafka クラスターのライフサイクル管理 |
| ArgoCD | GitOps | Git リポジトリと Kubernetes の状態を同期 |

---

## 9. コードリーディングの起点

Kubernetes 本体のコントローラ実装を参考にする（同じパターン）。

```
pkg/controller/deployment/deployment_controller.go
  └── DeploymentController ← Operator の参考実装（典型的な Reconcile Loop）

staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/customresource_handler.go
  └── CRD リソースの動的エンドポイント生成

staging/src/k8s.io/client-go/tools/cache/
  └── Informer / WorkQueue（Operator でも同じ client-go を使う）
```

Operator フレームワークのコード:

```
github.com/kubernetes-sigs/controller-runtime/pkg/reconcile/
  └── Reconciler インターフェース（Operator の Reconcile 関数の契約）

github.com/kubernetes-sigs/controller-runtime/pkg/builder/
  └── ctrl.NewControllerManagedBy() の実装（For/Owns/Watches の登録）

github.com/kubernetes-sigs/controller-runtime/pkg/manager/
  └── Manager（Informer・LeaderElection・Healthz の統合管理）
```

**読む順序**: `docs/controller-pattern.md` → `docs/crd.md` → `docs/garbage-collection.md` → 本ドキュメント
