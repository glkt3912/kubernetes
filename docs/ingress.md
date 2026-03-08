# Ingress / Gateway API

## 1. なぜ Ingress が必要か

Service の LoadBalancer 型は「1 Service = 1 ロードバランサ」。
多数のサービスがあると LB の費用・管理コストが増大する。

```
LoadBalancer 型（Ingress なし）:
  Service A → LB-A（外部IP: 1.2.3.4）
  Service B → LB-B（外部IP: 1.2.3.5）
  Service C → LB-C（外部IP: 1.2.3.6）
  → LB が 3 台必要・コスト大

Ingress あり:
  Ingress → LB 1台（外部IP: 1.2.3.4）
    /api   → Service A
    /web   → Service B
    /admin → Service C
  → LB 1台 + パスベースルーティング
```

Ingress = **L7 ロードバランサ**の設定を Kubernetes リソースとして宣言する仕組み。

---

## 2. Ingress の構成

```
外部クライアント
      ↓ HTTPS/HTTP
Ingress Controller（nginx / Traefik / AWS ALB 等）
      ↓ Ingress リソースの rules に従ってルーティング
Service（ClusterIP）
      ↓
Pod
```

**Ingress リソース**: ルーティングルールの宣言（YAML）
**Ingress Controller**: ルールを実装する実体（Pod として動く）

Kubernetes 本体は Ingress Controller を含まない。別途インストールが必要。

---

## 3. Ingress の型定義

```go
// staging/src/k8s.io/api/networking/v1/types.go:249
type Ingress struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    Spec   IngressSpec
    Status IngressStatus
}

type IngressSpec struct {
    IngressClassName *string         // どの IngressClass（Controller）を使うか
    DefaultBackend   *IngressBackend // rules にマッチしない場合のデフォルト転送先
    TLS              []IngressTLS    // TLS 終端の設定
    Rules            []IngressRule   // ルーティングルール
}

// staging/src/k8s.io/api/networking/v1/types.go:399
type IngressRule struct {
    Host             string          // ホスト名（例: api.example.com）
    IngressRuleValue                 // HTTP ルール（パスマッチング）
}

// staging/src/k8s.io/api/networking/v1/types.go:516
type IngressBackend struct {
    Service  *IngressServiceBackend         // 転送先 Service
    Resource *v1.TypedLocalObjectReference  // または他のリソース
}
```

### ルーティングルールの例

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /  # Controller 固有の設定
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert   # TLS 証明書を Secret から取得
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix       # /users, /users/1 ... にマッチ
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /orders
        pathType: Exact        # /orders にだけマッチ（/orders/1 は不可）
        backend:
          service:
            name: order-service
            port:
              number: 8080
  defaultBackend:              # どの rules にもマッチしない場合
    service:
      name: default-service
      port:
        number: 80
```

### pathType の種類

| pathType | 動作 |
|---|---|
| `Exact` | パスが完全一致 |
| `Prefix` | パスが前方一致（末尾 `/` の正規化あり）|
| `ImplementationSpecific` | Controller の実装依存 |

---

## 4. IngressClass

複数の Ingress Controller が存在する場合に、どの Controller がこの Ingress を担当するかを指定する。

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # デフォルト IngressClass
spec:
  controller: k8s.io/ingress-nginx
```

`ingressClassName` を省略した場合、デフォルト IngressClass の Controller が担当する。

---

## 5. 主な Ingress Controller

| Controller | 提供元 | 特徴 |
|---|---|---|
| ingress-nginx | Kubernetes SIG | nginx ベース。最も普及 |
| AWS Load Balancer Controller | AWS | ALB / NLB を直接作成 |
| Traefik | Traefik Labs | 動的設定更新・Let's Encrypt 自動対応 |
| Istio Gateway | Istio | Service Mesh との統合 |
| Kong | Kong Inc | API Gateway 機能（認証・レート制限等）|

**Kubernetes 本体に Ingress Controller は含まれない**。
型定義は `staging/src/k8s.io/api/networking/v1/` にあるが、
Controller 実装は `kubernetes/ingress-nginx` など外部リポジトリ。

---

## 6. TLS 終端

Ingress で HTTPS を終端し、バックエンドへは HTTP で転送できる。

```yaml
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-secret

---
# TLS Secret の構造
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

**cert-manager との連携**:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod  # 証明書を自動取得・更新
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert   # cert-manager が Secret を自動作成・更新
```

---

## 7. Gateway API（次世代 Ingress）

Ingress の限界を超えるために設計された新しい API 群（1.28 GA）。

### Ingress の限界

```
・annotations で Controller 固有設定を書かざるを得ない（移植性が低い）
・HTTP のみ（TCP/UDP ルーティングができない）
・ルール表現力が低い（ヘッダーベースルーティング等ができない）
・複数チームでの役割分担が難しい
```

### Gateway API の役割分担

```
インフラチーム:
  GatewayClass → 「どの Controller を使うか」を定義
  Gateway      → 「何番ポートで何を受け付けるか」を定義

アプリチーム:
  HTTPRoute    → 「どのパス/ヘッダーをどの Service に転送するか」を定義
```

```yaml
# GatewayClass
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: k8s.io/ingress-nginx

---
# Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      certificateRefs:
      - name: api-tls-cert

---
# HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: user-route
spec:
  parentRefs:
  - name: prod-gateway
  hostnames:
  - api.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /users
    - headers:              # ヘッダーベースルーティング（Ingress にはない機能）
      - name: X-Version
        value: v2
    backendRefs:
    - name: user-service-v2
      port: 8080
      weight: 80            # 重み付きトラフィック分割
    - name: user-service-v1
      port: 8080
      weight: 20
```

### Gateway API のルートの種類

| リソース | 用途 |
|---|---|
| `HTTPRoute` | HTTP/HTTPS のルーティング |
| `GRPCRoute` | gRPC のルーティング |
| `TCPRoute` | TCP のルーティング |
| `TLSRoute` | TLS（SNI ベース）のルーティング |
| `UDPRoute` | UDP のルーティング |

---

## 8. Ingress vs Gateway API

| | Ingress | Gateway API |
|---|---|---|
| 安定性 | GA（1.19）| GA（1.28）|
| 対応プロトコル | HTTP/HTTPS | HTTP/gRPC/TCP/TLS/UDP |
| ヘッダールーティング | Controller 依存（annotation）| ネイティブサポート |
| 重み付きトラフィック | Controller 依存 | ネイティブサポート |
| 役割分担 | 難しい | GatewayClass/Gateway/Route で明確 |
| 推奨 | 既存クラスターの継続利用 | 新規採用推奨 |

---

## 9. コードリーディングの起点

```
staging/src/k8s.io/api/networking/v1/types.go
  :249 Ingress の型定義
  :285 IngressSpec / IngressRule / IngressBackend
  :399 IngressRule（Host + pathType + backend）

# Ingress Controller の実装（外部リポジトリ）
github.com/kubernetes/ingress-nginx/
  └── internal/ingress/controller/ ← nginx 設定の生成

# Gateway API の型定義（外部リポジトリ）
github.com/kubernetes-sigs/gateway-api/
  └── apis/v1/httproute_types.go ← HTTPRoute の型定義
  └── apis/v1/gateway_types.go   ← Gateway の型定義
```
