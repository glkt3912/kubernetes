# ServiceAccount と TokenRequest

## 1. ServiceAccount とは

**Pod のアイデンティティ**。人間ユーザーではなくアプリケーション（Pod）が
Kubernetes API にアクセスするための認証情報。

```
人間ユーザー → kubectl → kubeconfig の証明書で認証
Pod（アプリ）→ apiserver → ServiceAccount のトークンで認証
```

---

## 2. 型定義

```go
// pkg/apis/core/types.go:5310
type ServiceAccount struct {
    metav1.TypeMeta
    metav1.ObjectMeta

    Secrets          []ObjectReference       // マウント可能な Secret 一覧（非推奨）
    ImagePullSecrets []LocalObjectReference  // イメージ Pull 用 Secret
    AutomountServiceAccountToken *bool       // トークンを自動マウントするか（デフォルト true）
}
```

---

## 3. デフォルト ServiceAccount

各 Namespace には `default` ServiceAccount が自動で作成される。
Pod に `serviceAccountName` を指定しない場合、`default` が使われる。

```bash
kubectl get serviceaccount -n default
# NAME      SECRETS   AGE
# default   0         10d   ← 自動作成される

kubectl get serviceaccount -n kube-system
# coredns   0         10d   ← CoreDNS 用
# ...
```

---

## 4. Pod へのトークンマウント

Pod 起動時、kubelet が ServiceAccount のトークンを自動でマウントする。

```
Pod 内のパス:
  /var/run/secrets/kubernetes.io/serviceaccount/token    ← JWT トークン
  /var/run/secrets/kubernetes.io/serviceaccount/ca.crt   ← apiserver の CA 証明書
  /var/run/secrets/kubernetes.io/serviceaccount/namespace ← 現在の Namespace 名
```

Pod 内から apiserver を呼び出す例:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

curl -H "Authorization: Bearer $TOKEN" \
     --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
     https://kubernetes.default.svc/api/v1/namespaces/$NAMESPACE/pods
```

### 自動マウントを無効化する

不要なトークンのマウントはセキュリティリスクになるため、不要な場合は無効にする。

```yaml
# ServiceAccount レベルで無効化
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
automountServiceAccountToken: false   # この SA を使う Pod にはマウントしない

---
# Pod レベルで個別に制御（SA の設定より優先）
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false
```

---

## 5. Bound Service Account Token（TokenRequest API）

従来のトークン（v1 Secret 方式）は有効期限がなく、無効化できないという問題があった。

**Bound Service Account Token**（1.22 GA）では:

```
・有効期限あり（デフォルト 1 時間）
・特定の Pod / Namespace / audience にバインドされる
・Pod が削除されると自動的に無効化される
・kubelet が自動でローテーションする
```

```go
// pkg/serviceaccount/claims.go
// JWT の payload の構造（Bound SA Token）
type privateClaims struct {
    Kubernetes kubernetes `json:"kubernetes.io"`
}

type kubernetes struct {
    Namespace string          `json:"namespace"`
    Pod       *ref            `json:"pod,omitempty"`   // バインドされた Pod
    ServiceAccount ref        `json:"serviceaccount"`
    Node       *ref           `json:"node,omitempty"`
}
```

### TokenRequest API で明示的にトークンを取得する

```bash
# 特定の audience 向けに 1 時間有効なトークンを発行
kubectl create token my-serviceaccount \
  --audience=https://my-api.example.com \
  --duration=3600s
```

```yaml
# Pod の Volume として projected token を設定
spec:
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          audience: https://my-api.example.com
          expirationSeconds: 3600
          path: token
  containers:
  - volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
```

---

## 6. RBAC との連携

ServiceAccount 単体では何の権限もない。RBAC で Role を付与して初めて操作できる。

```yaml
# 1. ServiceAccount の作成
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: default

---
# 2. Role の作成（Pod の一覧取得のみ許可）
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# 3. RoleBinding で ServiceAccount に Role を付与
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: pod-reader
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 7. ワークロードアイデンティティ（クラウド連携）

クラウド環境では ServiceAccount トークンをクラウドの IAM に連携できる。

### AWS IAM Roles for Service Accounts（IRSA）

```yaml
# ServiceAccount に AWS IAM Role の ARN を annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/S3AccessRole

---
# Pod はこの SA を使うだけで S3 にアクセスできる
spec:
  serviceAccountName: s3-access
  # AWS SDK が自動で SA トークンを使って AssumeRoleWithWebIdentity する
```

### GKE Workload Identity

```yaml
metadata:
  annotations:
    iam.gke.io/gcp-service-account: my-sa@my-project.iam.gserviceaccount.com
```

---

## 8. セキュリティのベストプラクティス

```
1. default ServiceAccount をアプリに使わない
   → 専用の ServiceAccount を作成し最小権限を付与する

2. automountServiceAccountToken: false を検討する
   → API アクセスが不要な Pod にはトークンをマウントしない

3. ClusterRole より Role を優先する
   → Namespace スコープの Role で Blast Radius を最小化

4. ServiceAccount の命名を明確にする
   → my-app-reader, my-app-writer のように権限を名前に反映する
```

---

## 9. コードリーディングの起点

```
pkg/apis/core/types.go:5310
  └── ServiceAccount の型定義

pkg/serviceaccount/claims.go
  └── Bound SA Token の JWT Claims 構造

pkg/serviceaccount/jwt.go
  └── JWT の生成・検証

staging/src/k8s.io/apiserver/pkg/authentication/serviceaccount/
  └── validator.go ← SA トークンの検証ロジック

plugin/pkg/admission/serviceaccount/admission.go
  └── Pod 作成時に SA トークンを自動マウントする Admission Plugin
```
