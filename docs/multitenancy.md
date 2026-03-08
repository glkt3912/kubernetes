# Namespace とマルチテナンシー設計

## 1. Namespace とは

リソースを分離する「仕切り」。同じ名前の Pod でも Namespace が違えば別物。

```
Namespace A（チーム1 / 本番）
  Pod: my-app（IP: 10.0.0.1）
  Service: my-service（ClusterIP: 10.96.0.1）

Namespace B（チーム2 / 本番）
  Pod: my-app（IP: 10.0.0.2）← 同名だが別物
  Service: my-service（ClusterIP: 10.96.0.2）
```

---

## 2. Namespace でできること・できないこと

| できること | できないこと |
|---|---|
| リソース名の衝突を防ぐ | Node / PersistentVolume の分離 |
| RBAC でチームごとのアクセス制御 | ネットワーク通信の自動遮断（NetworkPolicy が別途必要）|
| ResourceQuota でリソース上限を設定 | Namespace をまたいだ操作の完全な防止 |
| LimitRange でデフォルト値を設定 | |

**Namespace は弱い分離**:
デフォルトでは別 Namespace の Pod も `my-svc.other-ns.svc.cluster.local` でアクセスできる。
完全な分離には NetworkPolicy / RBAC / ResourceQuota の組み合わせが必要。

---

## 3. Namespace の種類

| Namespace | 用途 |
|---|---|
| `default` | 何も指定しないと使われる |
| `kube-system` | Kubernetes のシステムコンポーネント（CoreDNS / kube-proxy 等）|
| `kube-public` | 全ユーザーが読める公開情報（cluster-info 等）|
| `kube-node-lease` | Node の死活監視（NodeLease リソース）|

---

## 4. マルチテナンシーの設計パターン

### パターン 1: Namespace per Team（最も一般的）

```
namespace: team-a-prod
namespace: team-a-dev
namespace: team-b-prod
namespace: team-b-dev
```

```yaml
# 各 Namespace に ResourceQuota + LimitRange + RBAC をセットで設定
---
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a-prod
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    pods: "100"

---
kind: RoleBinding
metadata:
  name: team-a-admin
  namespace: team-a-prod
subjects:
- kind: Group
  name: team-a
roleRef:
  kind: ClusterRole
  name: admin
```

### パターン 2: Namespace per Application

```
namespace: frontend-prod
namespace: backend-prod
namespace: database-prod
```

アプリ間の依存関係が明確な場合に有効。NetworkPolicy で通信を制御しやすい。

### パターン 3: Namespace per Environment（最小構成）

```
namespace: production
namespace: staging
namespace: development
```

小規模チームや学習環境に向く。本番と開発が同一クラスターに混在するリスクあり。

---

## 5. Namespace の完全な分離に必要な設定

```
Namespace を安全に分離するには以下を組み合わせる:

1. RBAC          → 誰が何を操作できるか
2. ResourceQuota → リソースの上限（他チームへの影響防止）
3. LimitRange    → Pod 個別のデフォルト値と上限
4. NetworkPolicy → Pod 間通信の制限
5. PSA ラベル    → Pod のセキュリティポリシー
```

### 標準セット（Namespace 作成時に適用するテンプレート）

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-team-prod
  labels:
    team: my-team
    env: prod
    pod-security.kubernetes.io/enforce: restricted

---
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
  namespace: my-team-prod
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"

---
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: my-team-prod
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "4"
      memory: 4Gi

---
# network-policy.yaml（デフォルト拒否 + 同 Namespace 内のみ許可）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: my-team-prod
spec:
  podSelector: {}       # 全 Pod に適用
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: my-team-prod  # 同 Namespace のみ許可
```

---

## 6. Namespace をまたいだ通信の制御

```
デフォルト（NetworkPolicy なし）:
  Namespace A の Pod → Namespace B の Pod へ直接アクセス可能

NetworkPolicy で制限:
  Namespace A の Pod → Namespace B の Pod へのアクセスを拒否
  ただし DNS 解決は CoreDNS を経由するため制限不要（53番ポートは許可する）
```

### Namespace をまたいだ通信を許可する例

```yaml
# Namespace B に適用: Namespace A からのアクセスのみ許可
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-team-a
  namespace: team-b-prod
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: team-a   # Namespace のラベルで指定
```

---

## 7. Hierarchical Namespace（HNC）

大規模組織向けに Namespace を階層構造にする拡張機能（kubernetes-sigs/hierarchical-namespaces）。

```
org/
  ├── team-a/
  │   ├── team-a-prod    ← 親の ResourceQuota / RBAC を継承
  │   └── team-a-dev
  └── team-b/
      ├── team-b-prod
      └── team-b-dev
```

親 Namespace に設定した RBAC / ResourceQuota を子 Namespace に自動で継承できる。
Kubernetes 本体の機能ではなく、別途インストールが必要。

---

## 8. クラスターレベルの分離（より強い分離）

Namespace による分離では不十分なケース:

```
・セキュリティ要件が厳しいテナント（金融・医療等）
・完全なリソース保証が必要（ノイジーネイバー問題）
・異なるバージョンの Kubernetes が必要
```

より強い分離の選択肢:

| 方式 | 分離度 | コスト |
|---|---|---|
| Namespace 分離 | 低〜中 | 低 |
| Virtual Cluster（vcluster）| 高（API レベル）| 中 |
| 別クラスター | 最高 | 高 |

---

## 9. コードリーディングの起点

```
pkg/apis/core/types.go
  └── Namespace の型定義

pkg/controller/namespace/deletion/namespaced_resources_deleter.go
  └── Namespace 削除時に配下のリソースを全削除する処理

staging/src/k8s.io/api/core/v1/types.go
  └── Namespace / NamespaceSpec / NamespaceStatus

docs/resource-quota.md  ← ResourceQuota / LimitRange の詳細
docs/rbac-patterns.md   ← RBAC の詳細
docs/network.md         ← NetworkPolicy の詳細
docs/pod-security.md    ← Pod Security Admission の詳細
```
