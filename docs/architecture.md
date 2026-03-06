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

### APIServer を理解する

```
cmd/kube-apiserver/main.go
  └── app.NewAPIServerCommand()
      └── Run() → CreateServerChain()
          └── CreateKubeAPIServer()
              └── pkg/kubeapiserver/server.go

staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go
  └── GenericAPIServer（全 APIServer の共通基盤）
```

### Scheduler を理解する

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

```
cmd/kubelet/kubelet.go
  └── app.Run()

pkg/kubelet/kubelet.go
  └── Kubelet.Run()
      └── syncLoop() ← メインループ
          └── syncLoopIteration()
              └── HandlePodAdditions / HandlePodUpdates
                  └── dispatchWork() → podWorkers.UpdatePod()
                      └── syncPod() ← 実際の Pod 同期処理
```

### Deployment コントローラを理解する（コントローラの典型例）

```
pkg/controller/deployment/deployment_controller.go
  └── DeploymentController
      └── Run() → worker() → syncDeployment()
          └── 現状確認 → ReplicaSet の作成/更新/削除

pkg/controller/deployment/sync.go
  └── syncDeployment() 実装本体
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
