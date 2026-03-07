# ConfigMap / Secret

## 1. 一言で言うと

**ConfigMap** はアプリ設定を、**Secret** は機密情報をコンテナから切り離して管理するリソース。

```
コンテナイメージ（不変）
  +
ConfigMap（設定値: DB_HOST, TIMEOUT ...）
  +
Secret（機密情報: DB_PASS, TLS_CERT ...）
  ↓
実行中の Pod（設定が注入される）
```

**設定をイメージに焼き込まない**のが原則。環境（dev/staging/prod）ごとに異なる設定を、イメージの再ビルドなしに切り替えられる。

---

## 2. 型定義

### ConfigMap

```
staging/src/k8s.io/api/core/v1/types.go
```

```go
type ConfigMap struct {
    metav1.TypeMeta
    metav1.ObjectMeta

    Immutable   *bool               // true にすると変更不可（kubelet のキャッシュ最適化）
    Data        map[string]string   // テキストデータ
    BinaryData  map[string][]byte   // バイナリデータ（UTF-8 以外）
}
```

`Data` と `BinaryData` でキーの重複は禁止（バリデーションで強制）。

### Secret

```go
type Secret struct {
    metav1.TypeMeta
    metav1.ObjectMeta

    Immutable   *bool               // 変更不可フラグ
    Data        map[string][]byte   // base64 エンコードされたバイナリデータ
    StringData  map[string]string   // write-only の平文入力（読み取り時は Data に統合される）
    Type        SecretType          // 種別
}
```

**ConfigMap との違い**: `Data` の値型が `string` ではなく `[]byte`。etcd 上ではどちらも平文で保存されるが、Secret は暗号化設定（EncryptionConfiguration）により etcd 内で暗号化できる。

---

## 3. Secret の種別（Type）

| Type | 用途 |
|---|---|
| `Opaque`（デフォルト）| 任意のユーザーデータ |
| `kubernetes.io/service-account-token` | ServiceAccount トークン |
| `kubernetes.io/dockerconfigjson` | コンテナレジストリ認証 |
| `kubernetes.io/tls` | TLS 証明書と秘密鍵 |
| `kubernetes.io/basic-auth` | Basic 認証情報 |
| `kubernetes.io/ssh-auth` | SSH 秘密鍵 |
| `bootstrap.kubernetes.io/token` | ノード参加時の Bootstrap トークン |

`Type` は主にバリデーションのヒント。`Opaque` 以外は必須フィールドが検証される。

---

## 4. Pod への注入方法

### 4-1. 環境変数として注入

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: database.host
  - name: DB_PASS
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: password
```

**特徴**: シンプルだが、**Pod 起動後に ConfigMap/Secret を更新しても環境変数は変わらない**。
再起動が必要。

### 4-2. Volume としてマウント

```yaml
volumes:
  - name: config-vol
    configMap:
      name: app-config
  - name: secret-vol
    secret:
      secretName: app-secret

containers:
  - volumeMounts:
      - name: config-vol
        mountPath: /etc/config
      - name: secret-vol
        mountPath: /etc/secret
        readOnly: true
```

**特徴**: ファイルとして提供される。**kubelet が定期的に更新を反映する**（デフォルト約 60 秒）。
アプリが設定ファイルを再読み込みするならゼロダウンタイムで設定変更が可能。

### 4-3. envFrom で全キーを一括注入

```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secret
```

ConfigMap / Secret の全キーを環境変数として一括取り込む。

---

## 5. kubelet による Volume マウントの仕組み

```
pkg/kubelet/configmap/configmap_manager.go
pkg/kubelet/secret/secret_manager.go
```

### kubelet の処理フロー

```
Pod がスケジュール
  ↓
kubelet が SyncPod を実行
  ↓
ConfigMap / Secret Manager が対象リソースを取得
  ↓
emptyDir に展開（tmpfs）
  ├── /etc/config/database.host  → "mysql.example.com"
  └── /etc/secret/password       → "s3cr3t"
  ↓
コンテナに bind-mount
```

### キャッシュと更新

kubelet はデフォルトで ConfigMap/Secret をキャッシュし、TTL ごとに API Server から再取得する。
`Immutable: true` を設定すると kubelet はキャッシュを永続化し、API Server へのポーリングを停止する。
これにより大規模クラスタでの API Server 負荷を大幅に削減できる。

```
通常:   kubelet → (60秒ごと) → API Server → 最新値を反映
Immutable: kubelet のローカルキャッシュのみ使用（API Server 通信なし）
```

---

## 6. Secret の暗号化

デフォルトでは Secret は etcd に **平文で保存される**（base64 はエンコーディングであり暗号化ではない）。

### EncryptionConfiguration による暗号化

```yaml
# /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}  # 暗号化なし（フォールバック）
```

```
書き込み: kube-apiserver が Secret を AES-CBC で暗号化して etcd に保存
読み込み: kube-apiserver が etcd から取得して復号し、クライアントに返す
```

**コンポーネントが意識する必要はない**: 暗号化・復号は apiserver が透過的に行う。

### KMS との連携

クラウド環境では外部 KMS（AWS KMS, GCP KMS など）と連携し、
暗号化キー自体をクラスタ外で管理できる（Envelope Encryption）。

---

## 7. RBAC との連携

ConfigMap は通常 `get`/`list` を広く許可してよいが、
Secret は最小権限が重要。

```
× 悪い例: Secret に list 権限を与えると全 Secret を列挙できる
✓ 良い例: 特定の Secret 名への get のみ許可

rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["app-secret"]  # 特定リソースに限定
    verbs: ["get"]
```

---

## 8. 設計上の注意点

### ConfigMap / Secret の最大サイズ

- ConfigMap: etcd のオブジェクトサイズ上限（デフォルト 1.5 MiB）
- Secret: `MaxSecretSize = 1 MiB`（コードで定義）

大きなバイナリデータは別の仕組み（PVC, 外部ストレージ）を使うべき。

### SubPath マウントの注意

```yaml
volumeMounts:
  - name: config-vol
    mountPath: /etc/app/config.yaml
    subPath: config.yaml
```

`subPath` を使うと特定キーだけをファイルにマウントできるが、
**ConfigMap/Secret の更新が自動反映されない**（symlink による更新ができないため）。

### 環境変数注入 vs Volume マウント

| 方法 | 更新反映 | セキュリティ | 用途 |
|---|---|---|---|
| 環境変数 | 再起動が必要 | プロセス一覧に露出する可能性 | シンプルな設定値 |
| Volume | 自動反映（約60秒） | ファイル権限で制御可能 | 設定ファイル、証明書 |

---

## 9. コードリーディングの起点

| 処理 | ファイル |
|---|---|
| 型定義 | `staging/src/k8s.io/api/core/v1/types.go` |
| kubelet ConfigMap Manager | `pkg/kubelet/configmap/configmap_manager.go` |
| kubelet Secret Manager | `pkg/kubelet/secret/secret_manager.go` |
| Volume マウント処理 | `pkg/kubelet/volumemanager/` |
| 暗号化設定 | `staging/src/k8s.io/apiserver/pkg/server/options/encryptionconfig/` |
