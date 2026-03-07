# CRD と Kubernetes 拡張

## 1. Kubernetes 拡張の全体像

Kubernetes を拡張する方法は主に3つある。

```
+---------------------------+
|  Kubernetes 拡張の方法     |
|                           |
|  1. CRD                   |
|     カスタムリソース定義   |
|     → 最も手軽            |
|                           |
|  2. Aggregated API Server  |
|     独自の APIServer を    |
|     サブパスで公開         |
|     → 独自バリデーション   |
|       や独自ストレージが   |
|       必要な場合           |
|                           |
|  3. Admission Webhook     |
|     既存リソースの         |
|     検証・変換を拡張       |
+---------------------------+
```

CRD（CustomResourceDefinition）は「新しいリソース種別を定義する」方法。
`kubectl apply -f my-crd.yaml` で新しいリソースを Kubernetes に追加できる。

---

## 2. CRD の型定義

```
staging/src/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1/types.go
```

```go
type CustomResourceDefinitionSpec struct {
    Group   string                        // API グループ名 (例: myapp.example.com)
    Names   CustomResourceDefinitionNames // リソース名の定義
    Scope   ResourceScope                 // "Namespaced" or "Cluster"
    Versions []CustomResourceDefinitionVersion
    Conversion *CustomResourceConversion  // バージョン変換設定
}

type CustomResourceDefinitionNames struct {
    Plural   string   // 複数形 (例: widgets)
    Singular string   // 単数形 (例: widget)
    Kind     string   // CamelCase 種別名 (例: Widget)
    ShortNames []string // 略称 (例: ["wg"])
    Categories []string // グループ (例: ["all"])
}

type CustomResourceDefinitionVersion struct {
    Name    string  // バージョン名 (例: v1alpha1)
    Served  bool    // この API バージョンを公開するか
    Storage bool    // etcd への保存に使うバージョン（1つだけ true）
    Schema  *CustomResourceValidation // OpenAPI v3 スキーマ
    Subresources *CustomResourceSubresources // status / scale サブリソース
}
```

### CRD の YAML 例

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.myapp.example.com
spec:
  group: myapp.example.com
  scope: Namespaced
  names:
    plural: widgets
    singular: widget
    kind: Widget
  versions:
  - name: v1
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
```

---

## 3. CRD の処理フロー

### APIExtensionsServer の役割

kube-apiserver のサーバーチェーンの末端に `APIExtensionsServer` がある。

```
AggregatorServer
  └── KubeAPIServer（ビルトインリソース）
        └── APIExtensionsServer（CRD を処理）
```

CRD に対するリクエストは `customresource_handler.go` が動的に処理する。

```
staging/.../apiextensions-apiserver/pkg/apiserver/customresource_handler.go
```

CRD が登録されると、`customresource_discovery_controller.go` が検知して
動的に REST エンドポイントを生成する。

### CRD 登録から利用まで

```
1. kubectl apply -f my-crd.yaml
        |
        v
   CRD オブジェクトが apiserver に保存（etcd）
        |
        v
   customresource_discovery_controller が CRD の Watch イベントを受け取る
        |
        v
   新しい REST ハンドラを動的に登録
   /apis/myapp.example.com/v1/namespaces/*/widgets
        |
        v
2. kubectl apply -f my-widget.yaml  ← これで使える
```

---

## 4. カスタムリソースの内部表現

CRD で定義されたリソースは `Unstructured` 型として扱われる。
ビルトインリソース（Pod など）と違い、固定の Go 構造体を持たない。

```go
// staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/
type Unstructured struct {
    Object map[string]interface{}  // JSON そのものをマップで保持
}
```

カスタムコントローラでは `dynamic.Interface` か `typed client` を使って読む。

```go
// dynamic client: 型なし（Unstructured として読む）
dynamicClient.Resource(gvr).Namespace("default").Get(ctx, "my-widget", ...)

// typed client: code-generator で生成した型付きクライアント
widgetClient.MyappV1().Widgets("default").Get(ctx, "my-widget", ...)
```

---

## 5. CRD バージョン変換

CRD が複数のバージョンを持つ場合、変換が必要になる。
ビルトインリソースが internal version を使うのと異なり、
CRD は **Conversion Webhook** で外部変換を行う。

```go
type CustomResourceConversion struct {
    Strategy ConversionStrategyType  // "None" or "Webhook"
    Webhook  *WebhookConversion
}
```

```
クライアントが v1alpha1 で GET リクエスト
        |
        v
  etcd から v1（storage version）を取得
        |
        v  Conversion Webhook 呼び出し
  外部サービスが v1 → v1alpha1 に変換
        |
        v
  クライアントへ v1alpha1 で返却
```

---

## 6. カスタムコントローラの構造

CRD で定義したリソースを管理するコントローラは
ビルトインのコントローラパターンと同じ構造で書く。

```
CustomResource の変更
        |
        v
  SharedInformer（Reflector → DeltaFIFO → Indexer）
        |
        v
  EventHandler → WorkQueue にキー追加
        |
        v
  Worker goroutine → reconcile()
        |
        v
  現在状態の取得 → あるべき状態との比較 → API 呼び出し
```

```go
// code-generator で生成したクライアントを使う典型的なパターン
type WidgetController struct {
    client      clientset.Interface
    widgetLister listers.WidgetLister
    widgetSynced cache.InformerSynced
    queue       workqueue.TypedRateLimitingInterface[string]
}

func (c *WidgetController) reconcile(key string) error {
    ns, name, _ := cache.SplitMetaNamespaceKey(key)
    widget, err := c.widgetLister.Widgets(ns).Get(name)
    // ... あるべき状態と比較して差分を解消
}
```

---

## 7. CRD の高度な機能

| 機能 | 説明 |
|---|---|
| **Status サブリソース** | `spec` と `status` の更新を分離。コントローラが status を書く専用エンドポイント |
| **Scale サブリソース** | HPA（HorizontalPodAutoscaler）との連携に必要 |
| **Validation（CEL）** | OpenAPI スキーマ + CEL 式でバリデーション |
| **Defaulting** | スキーマの `default` フィールドでデフォルト値設定 |
| **Pruning** | スキーマに定義されていないフィールドを自動削除 |
| **Printer Columns** | `kubectl get` での表示カラムのカスタマイズ |

### CEL バリデーションの例

```yaml
x-kubernetes-validations:
  - rule: "self.spec.replicas <= 10"
    message: "replicas must be 10 or less"
```

---

## 8. コードリーディングの起点

```
staging/src/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1/types.go
  └── CRD の型定義

staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go
  └── APIExtensionsServer のセットアップ

staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/customresource_handler.go
  └── CRD リソースへの HTTP ハンドラ（動的生成）

staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/customresource_discovery_controller.go
  └── CRD 登録を Watch してエンドポイントを追加

staging/src/k8s.io/client-go/dynamic/
  └── 動的クライアント（Unstructured を扱う）

staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/
  └── Unstructured 型の実装
```
