# RBAC 設計パターン

## 1. RBAC の全体構造

RBAC（Role-Based Access Control）は「誰が、何に対して、何をできるか」を定義する仕組み。

```
+----------------+     参照     +------------------+
|  Subject       | ←---------- | RoleBinding /    |
|  (誰が)        |             | ClusterRoleBinding|
|                |             +------------------+
| User           |                    |
| Group          |                    | 参照
| ServiceAccount |                    v
+----------------+             +------------------+
                               | Role /           |
                               | ClusterRole      |
                               | (何に何を)        |
                               +------------------+
                                      |
                                      | PolicyRule
                                      v
                               APIGroups + Resources + Verbs
```

4つのリソースの関係:

| リソース | スコープ | 役割 |
|---|---|---|
| **Role** | Namespace | Namespace 内のリソースへのルール集 |
| **ClusterRole** | Cluster | クラスター全体 / 非 Namespace リソースへのルール集 |
| **RoleBinding** | Namespace | Subject と Role/ClusterRole を結ぶ |
| **ClusterRoleBinding** | Cluster | Subject と ClusterRole を結ぶ |

---

## 2. 型定義

```
pkg/apis/rbac/types.go
```

```go
type PolicyRule struct {
    Verbs           []string  // get, list, watch, create, update, patch, delete
    APIGroups       []string  // "" (core), "apps", "batch" など
    Resources       []string  // pods, deployments, configmaps など
    ResourceNames   []string  // 特定リソース名に絞る（省略可）
    NonResourceURLs []string  // /healthz などの非リソース URL
}

type Subject struct {
    Kind      string  // User / Group / ServiceAccount
    APIGroup  string
    Name      string
    Namespace string  // ServiceAccount の場合に必須
}

type RoleRef struct {
    APIGroup string  // rbac.authorization.k8s.io
    Kind     string  // Role or ClusterRole
    Name     string
}
```

### PolicyRule の特殊値

| フィールド | `"*"` の意味 |
|---|---|
| Verbs | 全操作 |
| APIGroups | 全 API グループ |
| Resources | 全リソース |

---

## 3. RBAC Authorizer の実装

`plugin/pkg/auth/authorizer/rbac/rbac.go`

```go
type RBACAuthorizer struct {
    authorizationRuleResolver RequestToRuleMapper
}

func (r *RBACAuthorizer) Authorize(
    ctx context.Context,
    requestAttributes authorizer.Attributes,
) (authorizer.Decision, string, error) {

    ruleCheckingVisitor := &authorizingVisitor{requestAttributes: requestAttributes}

    // ユーザーに適用される全 PolicyRule を訪問
    r.authorizationRuleResolver.VisitRulesFor(
        ctx,
        requestAttributes.GetUser(),
        requestAttributes.GetNamespace(),
        ruleCheckingVisitor.visit,
    )

    if ruleCheckingVisitor.allowed {
        return authorizer.DecisionAllow, ruleCheckingVisitor.reason, nil
    }
    return authorizer.DecisionNoOpinion, reason, nil
}
```

### 判定アルゴリズム

```
1. ClusterRoleBinding を全て評価 → 短絡 Allow あり？
        |
        v（なし）
2. RoleBinding（リクエストの Namespace 内）を全て評価 → Allow あり？
        |
        v（なし）
3. Deny（DecisionNoOpinion）
```

RBAC は「許可するルールを探す」のであって、明示的に Deny するルールはない。
どのルールにも当たらなければ NoOpinion（実質 Deny）。

### RuleAllows の実装

```go
func RuleAllows(requestAttributes authorizer.Attributes, rule *rbacv1.PolicyRule) bool {
    return rbacv1helpers.VerbMatches(rule, requestAttributes.GetVerb()) &&
        rbacv1helpers.APIGroupMatches(rule, requestAttributes.GetAPIGroup()) &&
        rbacv1helpers.ResourceMatches(rule, combinedResource, subresource) &&
        rbacv1helpers.ResourceNameMatches(rule, requestAttributes.GetName())
}
```

全フィールドが AND 条件。1つでも外れれば拒否。

---

## 4. Role vs ClusterRole の設計判断

### RoleBinding + ClusterRole の組み合わせ

ClusterRole を RoleBinding で参照すると、
**ClusterRole に定義されたルールを特定の Namespace でのみ適用** できる。

```
ClusterRole: pod-reader
  - get / list / watch pods

RoleBinding: read-pods-in-production
  - subjects: alice
  - roleRef: ClusterRole/pod-reader
  - namespace: production
```

この場合 alice は `production` Namespace の Pod しか読めない。

```
ClusterRole（ルール定義を再利用）
    +-- RoleBinding(ns=prod)  → prod だけアクセス可
    +-- RoleBinding(ns=dev)   → dev だけアクセス可
    +-- ClusterRoleBinding    → 全 Namespace アクセス可
```

### 選択の基準

| やりたいこと | 使うべき組み合わせ |
|---|---|
| 特定 Namespace のリソース管理 | Role + RoleBinding |
| 複数 Namespace で同じ権限 | ClusterRole + RoleBinding（Namespace ごと）|
| クラスター全体のリソース管理 | ClusterRole + ClusterRoleBinding |
| Node / PV など非 Namespace リソース | ClusterRole + ClusterRoleBinding のみ |

---

## 5. ServiceAccount との連携

### ServiceAccount の役割

Pod がクラスター内から API を呼ぶ際の「アイデンティティ」。

```
Pod 起動
  └── ServiceAccount トークンが自動マウント
        /var/run/secrets/kubernetes.io/serviceaccount/token
              |
              v
        apiserver に認証（JWT）
              |
              v
        RBAC で認可
```

### ServiceAccount に権限を付与する

```yaml
# ServiceAccount を作る
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-controller
  namespace: kube-system

---
# ClusterRole を作る
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-controller-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# ClusterRoleBinding で結ぶ
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-controller-binding
subjects:
- kind: ServiceAccount
  name: my-controller
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: my-controller-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 6. 最小権限の原則と設計パターン

### アンチパターン：広すぎる権限

```yaml
# 悪い例：全リソースに全操作
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### パターン1：読み取り専用

```yaml
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
```

### パターン2：サブリソースのみ

```yaml
# Pod のログだけ読める（Pod 本体は読めない）
rules:
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### パターン3：特定リソース名のみ

```yaml
# "my-config" という ConfigMap だけ読める
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

### パターン4：Status サブリソースのみ書き込み

```go
// コントローラは自分が管理するリソースの status だけ更新する
rules:
- apiGroups: ["myapp.example.com"]
  resources: ["widgets/status"]
  verbs: ["update", "patch"]
```

---

## 7. AggregatedClusterRole

ClusterRole を `aggregationRule` で組み合わせて継承できる。

```yaml
# admin role は複数の ClusterRole をまとめる
kind: ClusterRole
metadata:
  name: admin
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-admin: "true"
```

```yaml
# CRD ごとに admin 権限を追加する
kind: ClusterRole
metadata:
  name: widget-admin
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["myapp.example.com"]
  resources: ["widgets"]
  verbs: ["*"]
```

これにより CRD を追加するだけで既存の admin ロールに権限が自動追加される。

---

## 8. コードリーディングの起点

```
plugin/pkg/auth/authorizer/rbac/rbac.go
  └── RBACAuthorizer.Authorize() - 認可メインロジック
  └── RuleAllows() - PolicyRule とリクエストのマッチング

plugin/pkg/auth/authorizer/rbac/subject_locator.go
  └── VisitRulesFor() - Subject に適用される全ルールを列挙

pkg/apis/rbac/types.go
  └── PolicyRule / Subject / Role / ClusterRole の型定義

staging/src/k8s.io/api/rbac/v1/types.go
  └── external version の型定義

pkg/registry/rbac/
  └── RBAC リソースのストレージ実装

staging/src/k8s.io/apiserver/pkg/authorization/
  └── Authorizer インターフェース定義
```
