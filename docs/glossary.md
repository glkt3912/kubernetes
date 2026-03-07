# 用語集

Kubernetes のコードを読む際に頻出する型・概念の簡潔な説明。
詳細は各ドキュメントを参照。

---

## API オブジェクトの基本型

### TypeMeta

全 Kubernetes オブジェクトが持つ「種類」と「API バージョン」の情報。

```go
// staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go:42
type TypeMeta struct {
    Kind       string  // オブジェクトの種類（例: "Pod", "Deployment"）
    APIVersion string  // API バージョン（例: "v1", "apps/v1"）
}
```

YAML で書くと:

```yaml
apiVersion: apps/v1   # APIVersion
kind: Deployment      # Kind
```

---

### ObjectMeta

全永続オブジェクトが持つメタデータ。`metadata:` フィールドに対応する。

```go
// staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go:111
type ObjectMeta struct {
    Name            string            // オブジェクト名（Namespace 内で一意）
    Namespace       string            // 所属 Namespace（クラスタスコープのリソースは空）
    UID             types.UID         // サーバーが付与する一意 ID（UUID）
    ResourceVersion string            // 楽観的同時実行制御に使うバージョン番号
    Generation      int64             // spec の変更回数（更新のたびに +1）
    Labels          map[string]string // ラベル（セレクタで検索可能）
    Annotations     map[string]string // アノテーション（任意のメタデータ）
    OwnerReferences []OwnerReference  // 親オブジェクトへの参照
    Finalizers      []string          // 削除ブロック用フラグ
    DeletionTimestamp *Time           // グレースフル削除の予定時刻
}
```

---

### ResourceVersion

etcd が付与する単調増加する整数値（文字列型）。
オブジェクトが更新されるたびに増加し、**楽観的同時実行制御**（楽観的更新）に使われる。

```
更新フロー:
  GET → ResourceVersion: "42" を取得
  変更して PUT（ResourceVersion: "42" を付けて送る）
  → etcd が「現在の RV == 42?」を確認
  → 一致: 書き込み成功（RV が "43" になる）
  → 不一致: 409 Conflict（他が先に更新した）
```

Reflector が Watch を再開する際にも使う（「この RV 以降の変更を教えて」という意味で渡す）。
詳細は **[docs/informer-deep-dive.md](informer-deep-dive.md)** を参照。

---

### Generation

`spec` の変更回数を表す整数。コントローラが「この spec をまだ処理したか」を判定するために使う。

```
Generation と ObservedGeneration の関係:

  Deployment.spec を更新 → Generation: 3 になる
  controller が処理完了 → status.observedGeneration: 3 にする

  Generation > ObservedGeneration なら「まだ処理中」
  Generation == ObservedGeneration なら「処理完了」
```

`ResourceVersion` は etcd の全更新（spec / status 両方）で増加するが、
`Generation` は **spec の変更のみ**で増加する。

---

### UID

サーバーが生成する UUID（例: `"a1b2c3d4-..."`）。
`Name` は削除して再作成すると同じ名前が使えるが、`UID` は絶対に再利用されない。
コントローラが「同名の別オブジェクト」を区別するために使う。

---

### Labels と Annotations

どちらも `map[string]string` だが用途が異なる。

| | Labels | Annotations |
|---|---|---|
| 用途 | セレクタで検索・絞り込み | 任意のメタデータ保存 |
| 検索 | 可能（`-l app=nginx` 等） | 不可 |
| 値の長さ | 制限あり（63文字以下） | 制限なし |
| 例 | `app: nginx`, `env: prod` | ビルド番号・デプロイツール情報 |

コントローラが管理対象を見つける際は Labels + LabelSelector を使う。

---

### OwnerReference

「このオブジェクトは誰が作ったか」という親子関係を表すフィールド。

```yaml
ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: my-replicaset
    uid: "a1b2c3..."
    controller: true       # このオブジェクトの管理者
    blockOwnerDeletion: true  # 親削除前にこのオブジェクトを先に削除する
```

親オブジェクトが削除されると、`OwnerReference` を持つ子オブジェクトも自動削除される（Garbage Collection）。
詳細は **[docs/controller-pattern.md](controller-pattern.md)** を参照。

---

### Finalizers

削除を一時ブロックするためのフラグのリスト。

```
Finalizers が空でない間 → DeletionTimestamp は設定されるが etcd から消えない
コントローラがクリーンアップ後に Finalizers から自分のエントリを削除
→ 全 Finalizers が空になったら etcd から完全削除
```

外部リソース（クラウドのロードバランサー等）と連携するコントローラが
「削除前に外部リソースもクリーンアップする」ために使う。

---

### DeletionTimestamp

グレースフル削除が要求されたオブジェクトに設定される時刻。

**グレースフル削除とは**:
「コンテナに後処理の時間を与えてから削除する」仕組み。
即座に強制終了（SIGKILL）するのではなく、先に SIGTERM を送り一定時間待つ。

```
kubectl delete pod my-pod
  │
  ├── apiserver が DeletionTimestamp を設定（猶予期間の終了時刻）
  │
  ├── kubelet が SIGTERM をコンテナに送る
  │     → アプリが「終了するよ」シグナルを受け取り後処理ができる
  │       （接続を閉じる・処理中リクエストを完了させる等）
  │
  ├── 猶予期間（デフォルト 30 秒）待つ
  │
  ├── 猶予期間内に終了しなかった場合 → SIGKILL（強制終了）
  │
  └── コンテナ終了 → Finalizers 処理 → etcd から削除
```

**なぜ必要か**:

```
即座に強制終了すると:
  → 処理中だった HTTP リクエストが途中で切断される
  → クライアントがエラーを受け取る

グレースフル削除なら:
  → SIGTERM を受けて新規リクエストの受付を止める
  → 処理中のリクエストが完了してから自分で終了する
  → クライアントへの影響がない
```

猶予期間は `spec.terminationGracePeriodSeconds`（デフォルト 30 秒）で変更できる。
`kubectl delete pod my-pod --grace-period=0 --force` で即座に強制削除も可能。

`DeletionTimestamp != nil` をコントローラが確認することで「削除中オブジェクト」を識別できる。

---

---

## etcd

**Kubernetes の「唯一の真実の源（Source of Truth）」となる分散キーバリューストア。**

Kubernetes の全状態（Pod・Deployment・Service 等の全オブジェクト）はここに永続化される。
apiserver だけが直接アクセスし、他のコンポーネントは apiserver 経由でのみ読み書きする。

```
全コンポーネント
    │
    │ REST API
    v
  apiserver  ← 唯一の入口
    │
    │ gRPC（etcd クライアント）
    v
  etcd クラスタ（通常 3 または 5 台で冗長構成）
```

**キーバリューストアとは**:

```
Key（パス形式）                         Value（JSON/Protobuf）
/registry/pods/default/my-pod       →  { "apiVersion": "v1", "kind": "Pod", ... }
/registry/deployments/default/web   →  { "apiVersion": "apps/v1", ... }
```

ファイルシステムのようなパス形式でキーを管理し、値は Go の構造体をシリアライズしたもの。

**etcd の主な特徴**:

```
分散合意（Raft）:
  複数台のうち過半数が合意したときだけ書き込みが成功する
  → 1台クラッシュしても残りで継続できる（3台構成なら1台故障まで耐えられる）

Watch:
  特定キーの変更を購読できる
  → apiserver がこれを使ってコンポーネントに変更を通知する（Informer の基盤）

MVCC（多版同時実行制御）:
  全変更に Revision 番号が付き、過去の状態も参照できる
  → ResourceVersion の実体はこの Revision 番号
```

**なぜ apiserver 経由に限定するのか**:

```
直接アクセスを許可すると:
  → 認証・認可・バリデーションをバイパスできてしまう
  → 不正なデータが書き込まれてクラスタが壊れる可能性がある

apiserver 経由に限定することで:
  → 全書き込みが Authentication → Authorization → Admission を通過する
  → データの一貫性と安全性を保証できる
```

詳細は **[docs/apiserver.md](apiserver.md)** を参照。

---

## gRPC

**「関数を呼ぶように別プロセスと通信できる」仕組み。** Google が開発した RPC（Remote Procedure Call）フレームワーク。

通常の REST（HTTP + JSON）と対比すると分かりやすい：

```
REST（HTTP + JSON）:
  送信: POST /containers/create  Body: {"name": "my-container", "image": "nginx"}
  受信: {"id": "abc123", "status": "created"}
  → テキスト形式。人間が読める。

gRPC（Protocol Buffers）:
  送信: CreateContainer(name="my-container", image="nginx")  ← 関数呼び出しに見える
  受信: ContainerResponse(id="abc123", status="created")
  → バイナリ形式。人間には読めないが速い。
```

| | REST（HTTP + JSON） | gRPC |
|---|---|---|
| データ形式 | テキスト（JSON） | バイナリ（Protobuf） |
| 速度 | 普通 | 速い（データが小さい） |
| 読みやすさ | 人間が読める | 読めない |
| 主な用途 | 外部 API・ブラウザ向け | 内部コンポーネント間 |

**Kubernetes での使われ方**:

```
kubelet ──gRPC──► containerd / CRI-O（コンテナの起動・停止）  ← CRI
apiserver ──gRPC──► etcd（データの読み書き）
```

内部コンポーネント間の通信は速度重視なので gRPC を使う。
外部向け（kubectl など）は人間が扱いやすい REST を使う。

**「RPC」とは**:
Remote Procedure Call（遠隔手続き呼び出し）の略。
ネットワーク越しの通信を「別サーバーの関数を呼ぶ」ように書ける設計思想。
通信先がローカルか別サーバーかを意識しなくてよい。

---

## Kubernetes のリソース階層

### Namespace

クラスタ内の論理的な分離単位。リソースの名前空間を分けることで同じ名前のオブジェクトを複数持てる。

```
cluster
  ├── namespace: default     ← 省略時のデフォルト
  ├── namespace: kube-system ← システムコンポーネント用
  └── namespace: my-app      ← アプリ用（自作）
        ├── Pod: web-1
        └── Pod: web-2
```

**クラスタスコープのリソース**（Namespace に属さないもの）:
`Node`, `PersistentVolume`, `ClusterRole`, `Namespace` 自体など。

---

### Node

Pod を実際に動かす VM またはベアメタルサーバー。kubelet が動いているマシン。

```
Node の状態:
  Ready        → 正常に動作中、Pod を受け付けられる
  NotReady     → kubelet からの Lease が途絶えた（ネットワーク断・クラッシュ等）
  Unknown      → Node コントローラが状態を確認できない
```

詳細は **[docs/kubelet.md](kubelet.md)** を参照。

---

### Pod

Kubernetes の最小デプロイ単位。1つ以上のコンテナをまとめたもの。
同じ Pod 内のコンテナはネットワーク名前空間（IP アドレス）を共有する。

```
Pod の状態（phase）:
  Pending   → スケジュール待ち、またはイメージ pull 中
  Running   → コンテナが起動している
  Succeeded → 全コンテナが正常終了（exit code 0）
  Failed    → コンテナが異常終了
  Unknown   → Pod の状態を取得できない
```

---

### Deployment / ReplicaSet

```
Deployment（ユーザーが操作）
  └── ReplicaSet（Deployment が自動作成）
        └── Pod × n（ReplicaSet が自動作成）
```

- **Deployment**: 「Web サーバーを 3 台維持する」という宣言。ローリングアップデートを管理する。
- **ReplicaSet**: 「Pod を n 台維持する」という実装。直接操作することは少ない。

---

## コントローラ関連

### Reconciliation Loop（和解ループ）

「あるべき状態（desired state）と現在の状態（current state）の差分を継続的に埋める」処理。

```
desired state: Deployment replicas=3
current state: 実際に動いている Pod が 2 個
差分: 1 個足りない → Pod を 1 個作成
→ 再確認 → 差分なし → 何もしない
```

冪等（idempotent）に設計する：何度呼ばれても同じ結果になること。
詳細は **[docs/controller-pattern.md](controller-pattern.md)** を参照。

---

### Lister

Informer のローカルキャッシュ（Indexer）から**読み取るだけ**のインターフェース。
`reconcile` 内では apiserver に問い合わせず Lister を使う。

```go
// Lister の使い方
pod, err := kl.podLister.Pods(namespace).Get(name)
// → apiserver に問い合わせず、メモリ内キャッシュから取得
```

---

### HasSynced

Informer の初期 List が完了してキャッシュが利用可能な状態になったかを示す関数。
コントローラは起動直後に `WaitForCacheSync` でこれを確認してから reconcile を始める。

```go
if !cache.WaitForNamedCacheSyncWithContext(ctx,
    dc.dListerSynced,    // Deployment の HasSynced
    dc.rsListerSynced,   // ReplicaSet の HasSynced
    dc.podListerSynced,  // Pod の HasSynced
) {
    return  // キャッシュ未準備のまま reconcile してはいけない
}
```

---

## ネットワーク・サービス関連

### Service

Pod の集合に対する安定した仮想 IP（ClusterIP）とロードバランシングを提供するリソース。
Pod は再起動のたびに IP が変わるが、Service の IP は変わらない。

```
Service (ClusterIP: 10.96.0.1)
  → LabelSelector で対象 Pod を選択
    → Endpoints に実際の Pod IP:Port を記録
      → kube-proxy が iptables/ipvs ルールを設定
```

### Endpoints / EndpointSlice

Service が転送先とする Pod の IP:Port のリスト。
Service Controller が自動で管理する（手動で操作することは少ない）。

---

## ストレージ関連

### PersistentVolume (PV) / PersistentVolumeClaim (PVC)

```
PersistentVolume（クラスタ管理者が用意）
  → 実際のストレージ（NFS / AWS EBS / GCE PD 等）の表現

PersistentVolumeClaim（アプリ開発者が要求）
  → 「10GB の読み書き可能なストレージがほしい」という要求

PVC ← (バインド) → PV
```

Pod は PVC を参照してボリュームをマウントする。
PV が StorageClass で自動プロビジョニングされる場合は PV を意識しなくてよい。

---

## 用語の読み方・略称

| 略称 | 正式名 |
|---|---|
| k8s | Kubernetes（8文字を "8" で略したもの） |
| CRI | Container Runtime Interface |
| CNI | Container Network Interface |
| CSI | Container Storage Interface |
| RBAC | Role-Based Access Control |
| CRD | CustomResourceDefinition |
| HPA | HorizontalPodAutoscaler |
| VPA | VerticalPodAutoscaler |
| GVK | GroupVersionKind |
| GVR | GroupVersionResource |
| OOM | Out Of Memory |
| PLEG | Pod Lifecycle Event Generator |

---

## 参照先ドキュメント

| トピック | 詳細ドキュメント |
|---|---|
| Informer / Reflector / DeltaFIFO | [informer-deep-dive.md](informer-deep-dive.md) |
| Reconciliation Loop / WorkQueue | [controller-pattern.md](controller-pattern.md) |
| Scheduling Framework | [scheduler.md](scheduler.md) |
| APIServer パイプライン | [apiserver.md](apiserver.md) |
| kubelet / CRI / Probe | [kubelet.md](kubelet.md) |
| Go パターン（goroutine / channel 等） | [go-patterns.md](go-patterns.md) |
| アーキテクチャ全体像 | [architecture.md](architecture.md) |
