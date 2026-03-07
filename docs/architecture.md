# Kubernetes アーキテクチャ全体像

## 1. 全体構成

```
+--------------------------------------------------+
|                  Control Plane                   |
|                                                  |
|  +--------------+    +------------------------+ |
|  | kube-         |    | kube-controller-manager| |
|  | apiserver     |    | (各種 Controller)       | |
|  |               |    +------------------------+ |
|  | 全コンポーネント |                               |
|  | の通信ハブ    |    +------------------------+ |
|  |               |    | kube-scheduler         | |
|  |               |    | (Pod 配置決定)          | |
|  +------+--------+    +------------------------+ |
|         |                                        |
|  +------+--------+                              |
|  | etcd           |  (分散KVストア / 真実の源)  |
|  +---------------+                              |
+--------------------------------------------------+
         |  (Watch/List API)
+--------------------------------------------------+
|                   Data Plane                     |
|  Node 1                    Node 2                |
|  +--------------------+    +------------------+ |
|  | kubelet            |    | kubelet          | |
|  | (Pod 生死管理)      |    | (Pod 生死管理)   | |
|  | kube-proxy         |    | kube-proxy       | |
|  | (iptables/ipvs)    |    | (iptables/ipvs)  | |
|  +--------------------+    +------------------+ |
+--------------------------------------------------+
```

**通信の原則**: 全コンポーネントは **kube-apiserver 経由** で状態を読み書きする。
コンポーネント同士が直接通信することはなく、etcd への直接アクセスも apiserver のみ。

---

## 2. コンポーネント詳細

### kube-apiserver

- **エントリポイント**: `cmd/kube-apiserver/`
- **主実装**: `pkg/kubeapiserver/`
- **役割**:
  - REST API を提供（kubectl や他コンポーネントが使用）
  - 認証（Authentication）→ 認可（Authorization / RBAC）→ Admission Control のパイプライン
  - etcd への永続化
  - Watch ストリームを通じてコンポーネントに変更を通知
- **重要パッケージ**:
  - `staging/src/k8s.io/apiserver/` - APIサーバーの共通フレームワーク
  - `staging/src/k8s.io/apiserver/pkg/storage/` - etcd ストレージ抽象化
  - `pkg/registry/` - 各リソース（Pod, Deployment など）のストレージ実装

### kube-controller-manager

- **エントリポイント**: `cmd/kube-controller-manager/`
- **主実装**: `pkg/controller/`
- **役割**:
  - クラスタの「あるべき状態」を維持するコントローラ群のホスト
  - 代表的なコントローラ: Deployment, ReplicaSet, StatefulSet, Job, Node, Namespace など
  - 各コントローラは独立した goroutine で動作
- **重要パッケージ**:
  - `pkg/controller/deployment/` - Deployment コントローラ
  - `pkg/controller/replicaset/` - ReplicaSet コントローラ
  - `pkg/controller/nodelifecycle/` - Node ライフサイクル管理

### kube-scheduler

- **エントリポイント**: `cmd/kube-scheduler/`
- **主実装**: `pkg/scheduler/`
- **役割**:
  - Pending 状態の Pod を監視し、最適な Node を選択して割り当て
  - Filter（配置不可 Node を除外）→ Score（最適 Node をスコアリング）の2段階
  - プラグインシステムで拡張可能（Scheduling Framework）
- **重要パッケージ**:
  - `pkg/scheduler/framework/` - スケジューリングフレームワーク定義
  - `pkg/scheduler/framework/plugins/` - 組み込みプラグイン群

### kubelet

- **エントリポイント**: `cmd/kubelet/`
- **主実装**: `pkg/kubelet/`
- **役割**:
  - 各 Node 上で動作するエージェント
  - apiserver から自 Node に割り当てられた Pod を Watch
  - CRI（Container Runtime Interface）経由でコンテナを起動/停止
  - Pod のヘルスチェック（liveness/readiness probe）
  - Node のリソース使用量を apiserver に報告
- **重要パッケージ**:
  - `pkg/kubelet/kuberuntime/` - CRI を使ったコンテナ操作
  - `pkg/kubelet/prober/` - ヘルスプローブ実装

### kube-proxy

- **エントリポイント**: `cmd/kube-proxy/`
- **役割**:
  - Service の ClusterIP / NodePort を実現するネットワークルール管理
  - iptables モードまたは ipvs モードで動作
  - Endpoints の変更を Watch し、転送ルールを動的に更新

### etcd（外部依存）

- Kubernetes が直接持つのではなく外部クラスタとして運用
- **Kubernetes 側のコード**: `staging/src/k8s.io/apiserver/pkg/storage/etcd3/`
- Watch ストリームを利用して変更を効率よく通知

---

## 3. 設計パターン：Informer / WorkQueue / Reconciliation Loop

Kubernetes の全コントローラに共通する設計パターン。

```
  apiserver
     |
     | List & Watch（初回フル取得 + 差分ストリーム）
     v
+----------+     +----------+     +------------+
| Reflector| --> |  DeltaFIFO  | --> | Indexer    |
| (Watch)  |     | (差分キュー) |     | (ローカルキャッシュ)|
+----------+     +----------+     +------------+
                      |                  |
                      v                  v
              +---------------+   イベントハンドラ
              |  EventHandler |   (OnAdd/OnUpdate/OnDelete)
              +-------+-------+
                      |  キーのみエンキュー（obj key: "namespace/name"）
                      v
              +---------------+
              |   WorkQueue   |  (rate-limited, deduplication)
              +-------+-------+
                      |
                      v
              +---------------+
              |  reconcile()  |  ← ここがコントローラのビジネスロジック
              | （Worker）    |  現在状態 vs あるべき状態を比較し差分を解消
              +---------------+
                      |
                      | API 呼び出し（CREATE/UPDATE/DELETE）
                      v
                 apiserver
```

**重要な設計原則**:

- Informer はローカルキャッシュを持つため、reconcile 内では apiserver でなくキャッシュを読む
- WorkQueue は重複排除と Rate Limiting を自動的に行う
- reconcile は冪等（idempotent）に設計する（何度呼ばれても同じ結果になること）
- エラー時は `queue.AddRateLimited(key)` でリトライ

**実装場所**:

- `staging/src/k8s.io/client-go/tools/cache/` - Reflector, DeltaFIFO, Indexer, Informer
- `staging/src/k8s.io/client-go/util/workqueue/` - WorkQueue

---

## 4. staging/ パッケージ群

`staging/src/k8s.io/` 以下のパッケージは、外部に独立したモジュールとして公開されている。

| パッケージ | 用途 |
|---|---|
| `client-go` | Kubernetes API クライアント + Informer/WorkQueue の共通実装 |
| `apiserver` | APIServer フレームワーク（カスタムAPIServer 構築に使用） |
| `api` | API オブジェクト定義（external version） |
| `apimachinery` | API の型システム（TypeMeta, ObjectMeta, runtime.Object など） |
| `apiextensions-apiserver` | CRD（CustomResourceDefinition）のサポート |
| `controller-manager` | コントローラマネージャの共通フレームワーク |
| `kube-aggregator` | API Aggregation レイヤー |
| `metrics` | Prometheus メトリクス共通ライブラリ |
| `component-base` | コンポーネント共通基盤（flags, logs, version など） |

---

## 5. コードリーディングの起点

### Informer を理解する

詳細は **[docs/informer-deep-dive.md](informer-deep-dive.md)** を参照。

```
staging/src/k8s.io/client-go/tools/cache/controller.go
  └── NewInformer(), NewIndexerInformer()
      └── Controller.Run() → processLoop() → DeltaFIFO.Pop()

staging/src/k8s.io/client-go/tools/cache/shared_informer.go
  └── SharedIndexInformer（実際のコントローラが使う実装）
      └── Run() → reflector.Run() + controller.Run()

staging/src/k8s.io/client-go/tools/cache/reflector.go
  └── ListWatch から DeltaFIFO へデータを流す
```

**読む順序**: `reflector.go` → `delta_fifo.go` → `store.go` → `shared_informer.go`

### Go パターンを理解する

詳細は **[docs/go-patterns.md](go-patterns.md)** を参照。

```
goroutine   → go func() による並行処理
channel     → goroutine 間のデータ受け渡し
select      → 複数 channel の待ち受け
interface   → Duck Typing・プラグインシステム・型アサーション
context     → キャンセル・タイムアウトの伝播
WaitGroup   → goroutine の完了待ち
Mutex       → 共有データの保護
defer       → 確実な後処理（Mutex 解放・シャットダウン）
```

### APIServer を理解する

詳細は **[docs/apiserver.md](apiserver.md)** を参照。

```
cmd/kube-apiserver/apiserver.go:32
  └── main() → app.NewAPIServerCommand()
      └── Run() → CreateServerChain()
          └── AggregatorServer → KubeAPIServer → APIExtensionsServer（委譲チェーン）

staging/src/k8s.io/apiserver/pkg/endpoints/filters/
  └── authentication.go / authorization.go（認証・認可フィルタ）

staging/src/k8s.io/apiserver/pkg/admission/
  └── Admission Control プラグインシステム

staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go
  └── etcd への読み書き実装
```

### Scheduler を理解する

詳細は **[docs/scheduler.md](scheduler.md)** を参照。

```
cmd/kube-scheduler/main.go
  └── app.NewSchedulerCommand()

pkg/scheduler/scheduler.go
  └── Scheduler.Run()
      └── scheduleOne() ← 1 Pod の配置を決定するメインループ
          └── findNodesThatFitPod() → Filter plugins
          └── prioritizeNodes() → Score plugins
          └── bind()
```

### kubelet を理解する

詳細は **[docs/kubelet.md](kubelet.md)** を参照。

```
pkg/kubelet/kubelet.go:1828
  └── Run() - 各サブシステム起動・syncLoop 呼び出し

pkg/kubelet/kubelet.go:1944
  └── syncLoop() - メインイベントループ（PLEG/Watch/定期リシンクを select で待ち受け）

pkg/kubelet/kubelet.go:1996
  └── SyncPod() - 1 Pod の同期処理（コンテナ起動・ボリュームマウント等）

pkg/kubelet/kuberuntime/kuberuntime_manager.go
  └── CRI 経由のコンテナ操作

pkg/kubelet/prober/
  └── liveness/readiness/startup probe の実装
```

### Deployment コントローラを理解する（コントローラの典型例）

詳細は **[docs/controller-pattern.md](controller-pattern.md)** を参照。

```
pkg/controller/deployment/deployment_controller.go
  └── DeploymentController
      └── Run() → worker() → syncDeployment()
          └── 現状確認 → ReplicaSet の作成/更新/削除

pkg/controller/deployment/sync.go
  └── syncDeployment() 実装本体
```

---

### 用語を調べる

詳細は **[docs/glossary.md](glossary.md)** を参照。

```
TypeMeta / ObjectMeta / ResourceVersion / Generation / UID
Labels / Annotations / OwnerReference / Finalizers / DeletionTimestamp
Namespace / Node / Pod / Deployment / ReplicaSet
Lister / HasSynced / Service / PV / PVC
略称一覧（k8s / CRI / CNI / RBAC / CRD / PLEG 等）
```

### API バージョニングを理解する

詳細は **[docs/api-versioning.md](api-versioning.md)** を参照。

```
External / Internal / Storage の3バージョン構成
Scheme: GVK ↔ Go型 の登録簿
Conversion: バージョン間の型変換（internal を中継点に N 個の変換関数）

staging/src/k8s.io/apimachinery/pkg/runtime/scheme.go
  └── Scheme 構造体・ConvertToVersion

pkg/apis/core/v1/conversion.go        ← 手動変換関数
pkg/apis/core/v1/zz_generated.conversion.go ← 自動生成変換関数
```

### Taint / Toleration を理解する

詳細は **[docs/taint-toleration.md](taint-toleration.md)** を参照。

```
Taint: ノードが Pod を拒否する仕組み
Toleration: Pod が Taint を許容する設定
Effect: NoSchedule / PreferNoSchedule / NoExecute

pkg/scheduler/framework/plugins/tainttoleration/taint_toleration.go
  └── Filter（配置拒否）・Score（ソフト制約）の実装
```

### CRD と拡張を理解する

詳細は **[docs/crd.md](crd.md)** を参照。

```
CRD: 新しいリソース種別をアドオンで追加する仕組み
Unstructured: CRD リソースの内部表現（map[string]interface{}）
Conversion Webhook: CRD の複数バージョン間変換

staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/
  └── customresource_handler.go  ← 動的エンドポイント生成
  └── customresource_discovery_controller.go
```

### RBAC を理解する

詳細は **[docs/rbac-patterns.md](rbac-patterns.md)** を参照。

```
Role / ClusterRole: 何に何をできるか（PolicyRule の集合）
RoleBinding / ClusterRoleBinding: 誰に Role を割り当てるか
ServiceAccount: Pod のアイデンティティ

plugin/pkg/auth/authorizer/rbac/rbac.go
  └── RBACAuthorizer.Authorize() ← 認可メインロジック
```

---

### ストレージを理解する

詳細は **[docs/storage.md](storage.md)** を参照。

```
PV: 実際のストレージ（管理者が用意）
PVC: ストレージの要求（ユーザーが作成）
StorageClass: PVC から PV を自動作成する動的プロビジョニング

pkg/controller/volume/persistentvolume/pv_controller.go
  └── PV と PVC のバインディングロジック
```

### ネットワークを理解する

詳細は **[docs/network.md](network.md)** を参照。

```
Pod/Service/外部ネットワークの3層構成
Service: Pod の前に置く安定した仮想 IP（ClusterIP）
kube-proxy: iptables ルールを書いて Service → Pod 転送を実現

staging/src/k8s.io/api/core/v1/types.go:5942
  └── ServiceSpec（Type / Selector / ClusterIP / Ports）

pkg/proxy/iptables/proxier.go
  └── KUBE-SERVICES チェーン生成（iptables モード）
```

### オートスケールを理解する

詳細は **[docs/hpa.md](hpa.md)** を参照。

```
HPA（Horizontal Pod Autoscaler）:
  Metrics Server がメトリクスを収集
  HPA Controller が定期的に確認し Deployment の replicas を書き換える
  ReplicaSet が Pod を増減させる

pkg/controller/podautoscaler/horizontal.go
  └── HorizontalController / reconcileAutoscaler()

pkg/controller/podautoscaler/replica_calculator.go
  └── GetResourceReplicas() - レプリカ数の計算式
```

### Deployment / ReplicaSet を理解する

詳細は **[docs/deployment-replicaset.md](deployment-replicaset.md)** を参照。

```
Deployment（宣言）→ ReplicaSet（Pod 維持）→ Pod

pkg/controller/deployment/deployment_controller.go
  └── DeploymentController: ReplicaSet の replicas を管理
pkg/controller/deployment/rolling.go
  └── rolloutRolling(): MaxUnavailable/MaxSurge を守ったローリングアップデート
pkg/controller/replicaset/replica_set.go
  └── ReplicaSetController: Pod を作成・削除して replicas を維持
```

### ConfigMap / Secret を理解する

詳細は **[docs/configmap-secret.md](configmap-secret.md)** を参照。

```
ConfigMap: アプリ設定（DB_HOST, TIMEOUT ...）
Secret: 機密情報（DB_PASS, TLS_CERT ...）
注入方法: 環境変数 / Volume マウント / envFrom
暗号化: EncryptionConfiguration で etcd 内を暗号化可能

staging/src/k8s.io/api/core/v1/types.go
  └── ConfigMap / Secret の型定義
pkg/kubelet/configmap/ / pkg/kubelet/secret/
  └── kubelet による Volume マウント処理
```

### GC と OwnerReference を理解する

詳細は **[docs/garbage-collection.md](garbage-collection.md)** を参照。

```
OwnerReference: 親子関係を UID で記録
GarbageCollector: オーナー消滅時に依存オブジェクトを自動削除
削除パターン: Background / Foreground / Orphan
Finalizers: DeletionTimestamp セット後も物理削除を遅らせる仕組み

pkg/controller/garbagecollector/garbagecollector.go
  └── GarbageCollector / attemptToDeleteItem()
pkg/controller/garbagecollector/graph.go
  └── 依存グラフのノード定義
```

### StatefulSet を理解する

詳細は **[docs/statefulset.md](statefulset.md)** を参照。

```
StatefulSet: 安定したアイデンティティを持つ Pod（DB, Kafka, Elasticsearch）
  Pod 名: web-0, web-1, web-2（序数固定）
  ネットワーク: Headless Service で安定した DNS（web-0.svc.ns.svc.cluster.local）
  ストレージ: VolumeClaimTemplates で各 Pod が専用 PVC を保持

pkg/controller/statefulset/stateful_set_control.go
  └── updateStatefulSet() → processReplica() / processCondemned()
```

### DaemonSet を理解する

詳細は **[docs/daemonset.md](daemonset.md)** を参照。

```
DaemonSet: 全ノードに 1 Pod を保証（ログ収集, 監視, ネットワークプラグイン）
  Node 追加時: addNode() → nodeUpdateQueue → 自動で Pod 作成
  配置判定: podsShouldBeOnNode() → NodeSelector / Taints / Tolerations を評価

pkg/controller/daemon/daemon_controller.go
  └── syncDaemonSet() → manage() → syncNodes()
pkg/controller/daemon/update.go
  └── rollingUpdate()（maxUnavailable による更新制御）
```

### Job / CronJob を理解する

詳細は **[docs/job-cronjob.md](job-cronjob.md)** を参照。

```
Job: バッチ処理の完了保証（completions / parallelism / backoffLimit）
  失敗時: 指数バックオフ（10s → 20s → ... → 10min）で再試行
  Indexed Job: 各 Pod に連番インデックスを渡す

CronJob: スケジュールに従って Job を生成
  concurrencyPolicy: Allow / Forbid / Replace
  startingDeadlineSeconds: スケジュール遅延の許容時間

pkg/controller/job/job_controller.go
  └── Job Controller / syncJob()
pkg/controller/job/backoff_utils.go
  └── 指数バックオフの実装
pkg/controller/cronjob/cronjob_controllerv2.go
  └── CronJob Controller / syncCronJob()
```

---

## 6. 今後の学習ワークフロー

`kubernetes/` ディレクトリで `claude` を起動すると `.mcp.json` が読み込まれ、以下の MCP ツールが利用可能になる。

```bash
cd /Volumes/Dev-SSD/dev/kubernetes && claude
```

### 利用可能な MCP ツール

| やること | ツール呼び出し例 |
|---|---|
| OSS エイリアス確認 | `list_oss_sources` |
| リポジトリ構造分析 | `analyze_oss(source="kubernetes")` |
| ドキュメント自動生成 | `generate_oss_doc(source="kubernetes", focus="patterns")` |
| コード検索 | `search_oss_code(source="kubernetes", query="syncDeployment")` |
| 学習ドキュメント検索 | `search_metadata(query="Informer")` |
| ドキュメント取得 | `get_document_content(path="docs/architecture.md")` |

### 学習の進め方（推奨順序）

1. **Informer パターンを掴む** - `client-go/tools/cache/` を読む（全コントローラの基盤）
2. **小さなコントローラを読む** - `pkg/controller/deployment/` で Reconciliation Loop を体感
3. **Scheduler を読む** - `pkg/scheduler/` でプラグインシステムを理解
4. **APIServer を読む** - `pkg/kubeapiserver/` + `staging/src/k8s.io/apiserver/` で全体像把握
5. **kubelet を読む** - `pkg/kubelet/` で Node 側の実装を理解

### 新しいトピックのドキュメント追加方法

```
1. generate_oss_doc(source="kubernetes", focus="<トピック>") で草稿生成
2. docs/<topic>.md として保存
3. docs/metadata.json に新エントリを追加
```
