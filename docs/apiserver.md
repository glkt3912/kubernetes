# APIServer のリクエストパイプライン

## 1. APIServer とは何か

kube-apiserver は Kubernetes の**唯一の通信ハブ**。
etcd への読み書きも、コンポーネント間の連携も、すべて apiserver 経由で行われる。

```
kubectl / コントローラ / kubelet
        │
        │ REST API（HTTP/HTTPS）
        v
  kube-apiserver  ← 唯一の入口
        │
        v
      etcd（永続化）
```

**なぜ apiserver を経由するのか**:
- 認証・認可・バリデーションを一箇所に集約できる
- etcd への直接アクセスを防ぎ、データ整合性を保証できる
- Watch ストリームを通じて変更通知を一元管理できる

---

## 2. リクエストパイプライン全体像

kubectl などから届いた HTTP リクエストは、以下のパイプラインを順番に通過する。

```
HTTP リクエスト受信
        │
        v
┌──────────────────────────────────────┐
│  1. Authentication（認証）           │
│     「誰からのリクエストか？」        │
│     → 失敗: 401 Unauthorized         │
└───────────────┬──────────────────────┘
                │
                v
┌──────────────────────────────────────┐
│  2. Authorization（認可 / RBAC）     │
│     「その操作をしてよいか？」        │
│     → 失敗: 403 Forbidden            │
└───────────────┬──────────────────────┘
                │
                v
┌──────────────────────────────────────┐
│  3. Admission Control（入場審査）    │
│     「オブジェクトを変換・検証する」  │
│     Mutating → Validating の順       │
│     → 失敗: 400/422 Bad Request      │
└───────────────┬──────────────────────┘
                │
                v
┌──────────────────────────────────────┐
│  4. etcd への永続化                  │
│     バリデーション通過後に書き込む   │
└───────────────┬──────────────────────┘
                │
                v
       レスポンス返却 / Watch 通知
```

各ステップは HTTP ミドルウェア（`http.Handler` をラップする関数）として実装されており、
`staging/src/k8s.io/apiserver/pkg/endpoints/filters/` 以下にある。

---

## 3. サーバーチェーン構造

kube-apiserver は単一のサーバーではなく、**3つのサーバーが委譲（delegation）で連結**された構造になっている。

```
リクエスト
    │
    v
┌─────────────────────────┐
│ AggregatorServer        │  ← 最前段。APIService（拡張 API）へのルーティング
│ (kube-aggregator)       │
└───────────┬─────────────┘
            │ 自分で処理できなければ委譲
            v
┌─────────────────────────┐
│ KubeAPIServer           │  ← Pod/Deployment 等のコア API を処理
│ (pkg/controlplane)      │
└───────────┬─────────────┘
            │ 自分で処理できなければ委譲
            v
┌─────────────────────────┐
│ APIExtensionsServer     │  ← CRD（CustomResourceDefinition）を処理
│ (apiextensions-apiserver)│
└─────────────────────────┘
```

**実装**（`cmd/kube-apiserver/app/server.go:176`）:

```go
func CreateServerChain(config CompletedConfig) (*aggregatorapiserver.APIAggregator, error) {
    // 末端から順に生成し、前段に委譲先として渡す
    apiExtensionsServer, _ := config.ApiExtensions.New(notFoundHandler)
    kubeAPIServer, _ := config.KubeAPIs.New(apiExtensionsServer.GenericAPIServer)
    aggregatorServer, _ := CreateAggregatorServer(config.Aggregator, kubeAPIServer.GenericAPIServer, ...)
    return aggregatorServer, nil
}
```

---

## 4. Authentication（認証）

**「このリクエストは誰からか」を確認する**フェーズ。

```
実装: staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go:46
  WithAuthentication(handler, auth, failed, ...) http.Handler
```

**認証の方式**（複数を並列で試み、最初に成功したものを採用）:

```
X.509 クライアント証明書    ← コンポーネント間通信（kubelet, controller-manager）
Bearer トークン             ← ServiceAccount トークン（Pod 内からのアクセス）
Bootstrap トークン          ← Node の初回登録
OpenID Connect (OIDC)       ← 外部 IdP との連携（Dex, Keycloak 等）
Webhook トークン認証        ← 外部サービスに認証を委譲
```

認証に成功すると、`UserInfo`（ユーザー名・グループ・UID）が `context` に格納され、
後続のフィルタで参照される。

**匿名アクセス**:
どの認証方式にもマッチしなかった場合、`system:anonymous` ユーザーとして扱われる（設定により無効化可能）。

---

## 5. Authorization（認可）

**「そのユーザーはその操作をしてよいか」を確認する**フェーズ。

```
実装: staging/src/k8s.io/apiserver/pkg/endpoints/filters/authorization.go:53
  WithAuthorization(handler, auth, serializer) http.Handler
```

**RBAC（Role-Based Access Control）**:
Kubernetes のデフォルトの認可方式。「誰が・何に・何をできるか」をルールで定義する。

```
Role / ClusterRole     → 操作ルールの定義（verb: get/list/create 等、resource: pods 等）
RoleBinding / ClusterRoleBinding → ユーザー/グループ/ServiceAccount に Role を紐付ける

例:
  ClusterRole: view-pods
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list", "watch"]

  ClusterRoleBinding: alice-view-pods
    - subject: alice
    - roleRef: view-pods
```

認可の判定は `Authorizer` インターフェースが担う。
RBAC 以外にも Node 認可・ABAC・Webhook 認可なども選択できる。

---

## 6. Admission Control（入場審査）

**「リクエストを通過させる前に変換・検証する」**フェーズ。
認証・認可とは異なり、**オブジェクトの内容を変更できる**（Mutating）。

```
実装: staging/src/k8s.io/apiserver/pkg/admission/
  interfaces.go    → MutationInterface / ValidationInterface の定義
  chain.go         → 複数プラグインをチェーンで呼ぶ実装
```

**2段階で処理される**:

```
1. Mutating Admission（変換）
   → オブジェクトを変更してよい
   例: デフォルト値の注入・サイドカーコンテナの自動挿入

2. Validating Admission（検証）
   → オブジェクトの変更は不可、検証のみ
   例: 必須フィールドの確認・ポリシー違反のチェック
```

**組み込み Admission プラグインの例**:

| プラグイン | 種類 | 動作 |
|---|---|---|
| `NamespaceLifecycle` | Validating | 削除中の Namespace へのオブジェクト作成を拒否 |
| `LimitRanger` | Mutating/Validating | Namespace の LimitRange に基づいてデフォルトリソースを注入 |
| `ServiceAccount` | Mutating | Pod に ServiceAccount トークンを自動マウント |
| `ResourceQuota` | Validating | Namespace のリソース上限を超えないか確認 |
| `PodSecurity` | Validating | Pod Security Standards に準拠しているか確認 |

**Webhook Admission**:
外部 HTTP サーバーに審査を委譲できる。
`MutatingWebhookConfiguration` / `ValidatingWebhookConfiguration` で設定する。

```
リクエスト → apiserver → 外部 Webhook サーバー（任意のロジック）
                              → 許可/拒否/オブジェクト変換を返す
```

---

## 7. etcd への永続化

Admission を通過したオブジェクトは etcd に書き込まれる。

```
実装: staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go
```

**apiserver と etcd の関係**:

```
apiserver は etcd の唯一のクライアント
  → 他のコンポーネントは etcd に直接アクセスしない
  → apiserver がストレージ抽象化レイヤーを提供する

storage.Interface（抽象）
  └── etcd3.store（実装）
        └── etcd クライアント（clientv3）
              └── etcd クラスタ
```

**シリアライズ（直列化）**:
Go の構造体（Pod 等）は JSON または Protobuf に変換されて etcd に保存される。
Protobuf の方が小さく高速なため、コンポーネント間通信では Protobuf が優先される。

**ResourceVersion と楽観的同時実行制御**:

```
etcd の全オブジェクトには ResourceVersion（単調増加する整数）がある

更新フロー:
  1. GET → ResourceVersion: 42 を取得
  2. 変更して PUT（ResourceVersion: 42 を付けて送る）
  3. etcd が「現在の RV == 42?」を確認
     → 一致: 書き込み成功（RV が 43 になる）
     → 不一致: 409 Conflict（他が先に更新した）→ 再取得して再試行
```

この仕組みにより、複数コントローラが同時に更新しても矛盾が生じない。

---

## 8. Watch ストリーム

etcd への書き込みが完了すると、**Watch 中のクライアントにイベントが通知される**。

```
kubelet / コントローラ
    │
    │ GET /api/v1/pods?watch=true  ← Watch リクエスト（HTTP の長持続接続）
    v
  apiserver
    │
    ├── etcd の Watch ストリームを購読
    │
    └── etcd で変更が発生
          → apiserver がイベントを受け取る
          → Watch 中の全クライアントに転送
```

**Reflector との関係**:
`client-go` の Reflector は `List & Watch` を使って apiserver から変更を受け取る。
内部的には apiserver の Watch エンドポイントに HTTP 接続を張り続けている。

---

## 9. よくある疑問 Q&A

**Q: kubectl apply はどのパイプラインを通るか？**

```
kubectl apply
  → HTTP PATCH（Server-Side Apply）または POST/PUT を apiserver に送る
  → Authentication → Authorization → Admission → etcd 書き込み
  → Watch 中のコントローラに通知 → Reconcile が動く
```

**Q: ServiceAccount とは何か？**

Pod が apiserver にアクセスするための「Pod 専用のユーザー」。
Pod が起動すると `/var/run/secrets/kubernetes.io/serviceaccount/token` に
JWT トークンが自動マウントされ、このトークンで認証する。

**Q: DryRun とは何か？**

`kubectl apply --dry-run=server` を使うと、
Admission まで実行されるが etcd への書き込みは行われない。
「このリソースを作成したら通過するか？」を事前確認するために使う。

---

## 10. 次に読むべきファイル

### サーバーの起動フロー

```
cmd/kube-apiserver/apiserver.go:32
  └── main() → app.NewAPIServerCommand()

cmd/kube-apiserver/app/server.go:148
  └── Run() → NewConfig() → CreateServerChain() → PrepareRun() → Run()

cmd/kube-apiserver/app/server.go:176
  └── CreateServerChain() - 3サーバーの委譲チェーンを構築
```

### リクエストフィルタ

```
staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go:46
  └── WithAuthentication() - 認証フィルタ

staging/src/k8s.io/apiserver/pkg/endpoints/filters/authorization.go:53
  └── WithAuthorization() - 認可フィルタ
```

### Admission Control

```
staging/src/k8s.io/apiserver/pkg/admission/interfaces.go:31
  └── Attributes, MutationInterface, ValidationInterface の定義

staging/src/k8s.io/apiserver/pkg/admission/chain.go
  └── chainAdmissionHandler - 複数プラグインを順に呼ぶ実装
```

### etcd ストレージ

```
staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:80
  └── store 構造体 - etcd への読み書き実装

staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go
  └── Watch ストリームの実装
```

### GenericAPIServer（共通基盤）

```
staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go
  └── GenericAPIServer - 全 APIServer の共通基盤
      フィルタチェーンの組み立て・ルーティングの登録
```
