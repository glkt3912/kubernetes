# API バージョニングと型変換

## 1. なぜバージョニングが必要か

Kubernetes の API は時間をかけて進化する。`v1alpha1` → `v1beta1` → `v1` のように
安定度ランクが上がり、フィールドの追加・削除・リネームが起こる。
しかし etcd に保存されたオブジェクトは古い形式のまま残っているし、
古いクライアントは古いバージョンで API を呼ぶ。

この「複数バージョンの共存」を実現するために Kubernetes は
**3 種類のバージョン** と **Scheme + Conversion** の仕組みを持つ。

---

## 2. 3 種類のバージョン

```
+------------------+        +------------------+        +------------------+
| External Version |        | Internal Version |        | Storage Version  |
| (公開 API)        | <----> | (内部処理用)      | <----> | (etcd 永続化)    |
|                  |        |                  |        |                  |
| pkg/apis/core/v1/|        | pkg/apis/core/   |        | v1 (通常は最新   |
| staging/.../api/ |        | types.go         |        | stable version)  |
+------------------+        +------------------+        +------------------+
```

| 種別 | 説明 | 実装場所 |
|---|---|---|
| **External Version** | クライアントが使う公開 API の型。JSON/YAML でやりとり | `staging/src/k8s.io/api/core/v1/` |
| **Internal Version** | APIServer 内部で使う型。バージョン間変換のハブ | `pkg/apis/core/types.go` |
| **Storage Version** | etcd に永続化するバージョン（通常は安定した external version）| `pkg/apis/core/v1/` |

### 設計の要点

External と Internal が分かれている理由は「バージョン爆発を防ぐため」。

```
v1alpha1 <---> internal <---> v1beta1
                  ^
                  |
                 v1
```

もし internal なしで各バージョン間に変換を書くと N^2 個の変換関数が必要になる。
internal を中継点にすることで N 個の変換で済む。

---

## 3. Scheme：型とバージョンの登録簿

`Scheme` は Kubernetes の型システムの中核。
Go の型（`reflect.Type`）と GroupVersionKind（GVK）のマッピングを管理する。

```
staging/src/k8s.io/apimachinery/pkg/runtime/scheme.go
```

```go
type Scheme struct {
    gvkToType   map[schema.GroupVersionKind]reflect.Type  // GVK → Go型
    typeToGVK   map[reflect.Type][]schema.GroupVersionKind // Go型 → GVK
    converter   *conversion.Converter  // 変換関数の登録簿
    versionPriority map[string][]string // バージョン優先順位
}
```

### Scheme への型登録

```go
// pkg/apis/core/v1/register.go
func init() {
    // v1 パッケージの型を "core/v1" として登録
    scheme.AddToScheme(SchemeBuilder.Build())
}
```

```go
// pkg/apis/core/register.go
func init() {
    // internal バージョンを "__internal" として登録
    scheme.AddInternalGroupVersion(SchemeGroupVersion)
}
```

`APIVersionInternal = "__internal"` という特別な定数が internal バージョンのキー。

---

## 4. Conversion：バージョン間の型変換

### 変換の流れ

```
クライアントが v1beta1 で POST
        |
        v
  Decode (JSON → v1beta1 の Go 型)
        |
        v  Convert (v1beta1 → internal)
  internal 型でバリデーション・Admission
        |
        v  Convert (internal → storage version)
  etcd に保存（v1 形式）
        |
        v  Convert (v1 → リクエストされたバージョン)
  クライアントへ返却
```

### 変換関数の登録

```go
// pkg/apis/core/v1/conversion.go
func init() {
    SchemeBuilder.Register(addConversionFuncs)
}

func addConversionFuncs(scheme *runtime.Scheme) error {
    // v1.Pod → core.Pod (internal)
    err := scheme.AddConversionFunc(
        (*v1.Pod)(nil),
        (*core.Pod)(nil),
        func(a, b interface{}, scope conversion.Scope) error {
            return Convert_v1_Pod_To_core_Pod(a.(*v1.Pod), b.(*core.Pod), scope)
        },
    )
    return err
}
```

変換関数は自動生成（`zz_generated.conversion.go`）と手動実装の2種類がある。
フィールド名が変わった場合やデフォルト値が必要な場合は手動で書く。

### ConvertToVersion の呼び出し

```go
// scheme.go
func (s *Scheme) ConvertToVersion(in Object, target GroupVersioner) (Object, error) {
    // 1. 変換先 GVK を決定
    // 2. 変換先の Go 型を生成
    // 3. 登録済み変換関数を実行
    // 4. TypeMeta を書き換えて返す
}
```

---

## 5. Storage Version と Conversion

etcd には1つのバージョンのみ保存される（Storage Version）。
読み出し時に必要なバージョンに変換される。

```
etcd (v1 形式で保存)
        |
        v  GET /apis/apps/v1beta1/deployments/foo
  Read → v1 を Decode
        |
        v  Convert v1 → internal → v1beta1
  Response として v1beta1 を返す
```

### Hub バージョンとしての Internal

```
v1alpha1 ──→ internal ←── v1beta1
                 │
                 └──→ v1 (storage)
```

新しいバージョンを追加するときは「internal ↔ new_version」の変換だけ書けば良い。

---

## 6. CRD における バージョニング

CRD はビルトインリソースとは異なり、internal バージョンを持たない。
代わりに **Conversion Webhook** で外部サービスがバージョン変換を担う。

```go
// staging/.../apiextensions/v1/types.go
type CustomResourceConversion struct {
    Strategy ConversionStrategyType  // "None" or "Webhook"
    Webhook  *WebhookConversion
}
```

- `None`: apiVersion フィールドを書き換えるだけ（フィールド構造が同一の場合）
- `Webhook`: 外部 Webhook が変換ロジックを実装

---

## 7. コードリーディングの起点

```
staging/src/k8s.io/apimachinery/pkg/runtime/scheme.go
  └── Scheme 構造体・AddConversionFunc・ConvertToVersion

staging/src/k8s.io/apimachinery/pkg/conversion/converter.go
  └── 変換関数の実行エンジン

pkg/apis/core/v1/zz_generated.conversion.go
  └── 自動生成された変換関数（フィールドのコピー）

pkg/apis/core/v1/conversion.go
  └── 手動で書いた変換関数（フィールド名変更・デフォルト値など）

staging/src/k8s.io/apiserver/pkg/registry/rest/rest.go
  └── ストレージ層での Decode → 変換 → Encode フロー
```

| ファイル | 役割 |
|---|---|
| `pkg/apis/core/types.go` | internal version の型定義 |
| `staging/.../api/core/v1/types.go` | external version の型定義 |
| `pkg/apis/core/v1/register.go` | Scheme への型登録 |
| `pkg/apis/core/v1/conversion.go` | 手動変換関数 |
| `pkg/apis/core/v1/zz_generated.conversion.go` | 自動生成変換関数 |
