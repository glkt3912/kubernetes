# CoreDNS と Kubernetes の DNS 解決

## 1. なぜ DNS が必要か

Pod の IP は起動のたびに変わる。Service の ClusterIP も再作成で変わりうる。
**名前（Service 名）で接続できれば、IP を気にしなくてよい**。

```
# IP で接続（脆弱）
curl http://10.96.0.1:8080   ← Service を再作成したら IP が変わって壊れる

# DNS 名で接続（堅牢）
curl http://my-service:8080  ← CoreDNS が IP を引いてくれる
```

---

## 2. CoreDNS の全体像

CoreDNS は `kube-system` Namespace で **Deployment** として動く。

```
+------------------------------------------------------+
|  Pod（どの Namespace でも）                            |
|                                                      |
|  curl http://my-service                              |
|    ↓ /etc/resolv.conf に書かれた nameserver に問い合わせ |
+------------------------------------------------------+
                    ↓ UDP/TCP port 53
+------------------------------------------------------+
|  CoreDNS Pod（kube-system）                           |
|                                                      |
|  kubernetes plugin が Service/Pod を Watch            |
|    ↓ kubernetes API から Service 情報を取得            |
|    ↓ 名前 → ClusterIP を返す                          |
|                                                      |
|  未知のドメイン → forward で外部 DNS に転送            |
+------------------------------------------------------+
                    ↓ 外部 DNS（/etc/resolv.conf の上流）
            8.8.8.8（Google DNS）等
```

---

## 3. DNS 名の命名規則

### Service の DNS 名

```
<service-name>.<namespace>.svc.<cluster-domain>

例:
  my-service.default.svc.cluster.local
  my-service.prod.svc.cluster.local
```

### 短縮形でアクセスできる仕組み（search ドメイン）

Pod の `/etc/resolv.conf` には `search` ドメインが設定されている。

```
# Pod の /etc/resolv.conf（kubelet が生成）
nameserver 10.96.0.10          ← CoreDNS Service の ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`search` ドメインがあるため、短縮名でも解決できる。

```
curl http://my-service
  ↓ my-service → まず my-service.default.svc.cluster.local を試す
  ↓ CoreDNS が解決 → ClusterIP を返す

curl http://my-service.prod
  ↓ my-service.prod.svc.cluster.local を試す
  ↓ CoreDNS が解決
```

### ndots:5 の意味

`options ndots:5` は「ドット数が 5 未満の名前は search ドメインを補完してから解決する」設定。

```
my-service           ← ドット 0個 → search ドメインを補完
my-service.prod      ← ドット 1個 → search ドメインを補完
a.b.c.d.e            ← ドット 4個 → search ドメインを補完
a.b.c.d.e.f          ← ドット 5個 → そのまま解決（補完しない）
```

**外部ドメインアクセス時の注意**:

```
curl http://api.example.com
  ↓ ドット数が 2 (<5) なので search ドメインを補完して順に試す
  ↓ api.example.com.default.svc.cluster.local → 失敗
  ↓ api.example.com.svc.cluster.local → 失敗
  ↓ api.example.com.cluster.local → 失敗
  ↓ api.example.com → 外部 DNS で解決（ここまで無駄な試みが走る）

対策: api.example.com. ← 末尾にドットをつけると補完しない（絶対名）
```

### StatefulSet の Pod の DNS 名

StatefulSet の各 Pod は個別に DNS 名を持つ（Headless Service 経由）。

```
<pod-name>.<service-name>.<namespace>.svc.<cluster-domain>

例（web という StatefulSet、headless という Headless Service）:
  web-0.headless.default.svc.cluster.local  → web-0 の Pod IP
  web-1.headless.default.svc.cluster.local  → web-1 の Pod IP
  web-2.headless.default.svc.cluster.local  → web-2 の Pod IP
```

通常の Service と違い、Headless Service は ClusterIP を持たず、
DNS が直接 Pod の IP を返す。

---

## 4. CoreDNS の設定（Corefile）

CoreDNS の設定は `kube-system/coredns` ConfigMap の `Corefile` フィールドに記述される。

```
# cluster/addons/dns/coredns/coredns.yaml.base
.:53 {
    errors                    # エラーをログ出力
    health {                  # /health エンドポイント（liveness probe）
        lameduck 5s
    }
    ready                     # /ready エンドポイント（readiness probe）
    kubernetes cluster.local in-addr.arpa ip6.arpa {  # Kubernetes DNS 解決
        pods insecure         # Pod の逆引き解決を許可
        fallthrough in-addr.arpa ip6.arpa  # 逆引き失敗時は上流に転送
        ttl 30
    }
    prometheus :9153          # Prometheus メトリクスを 9153 番で公開
    forward . /etc/resolv.conf {   # 未知のドメインは Node の resolv.conf の DNS に転送
        max_concurrent 1000
    }
    cache 30                  # 30 秒キャッシュ
    loop                      # DNS ループ検出
    reload                    # Corefile の変更を自動リロード
    loadbalance               # 複数 A レコードをランダムに並び替え（簡易 LB）
}
```

プラグインチェーン形式で処理が行われる。

```
DNS クエリ受信
    ↓ errors プラグイン（エラーハンドリング）
    ↓ health プラグイン（ヘルスチェック用エンドポイント）
    ↓ ready プラグイン（準備完了チェック）
    ↓ kubernetes プラグイン（Service/Pod の名前解決）
        → cluster.local で終わるクエリはここで解決
        → 解決できなければ次へ
    ↓ forward プラグイン（外部 DNS への転送）
```

---

## 5. kubelet が Pod の /etc/resolv.conf を生成する仕組み

Pod 起動時、kubelet が CoreDNS の IP を `nameserver` として書き込む。

```go
// pkg/kubelet/network/dns/dns.go:59
type Configurer struct {
    clusterDNS    []net.IP  // CoreDNS Service の IP（kubelet の --cluster-dns フラグ）
    ClusterDomain string    // クラスタードメイン（--cluster-domain）
    ResolverConfig string   // Node の resolv.conf のパス
}

// GetPodDNS: Pod の DNS 設定を返す
func (c *Configurer) GetPodDNS(ctx context.Context, pod *v1.Pod) (*runtimeapi.DNSConfig, error) {
    // dnsPolicy が ClusterFirst の場合（デフォルト）:
    dnsConfig.Servers = []string{clusterDNS IP}            // nameserver に CoreDNS を設定
    dnsConfig.Searches = generateSearchesForDNSClusterFirst() // search ドメインを生成
    dnsConfig.Options = []string{"ndots:5"}                // ndots:5 を設定
}
```

生成される `/etc/resolv.conf` の内容:

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`defaultDNSOptions = []string{"ndots:5"}` がハードコードされている
（`pkg/kubelet/network/dns/dns.go:44`）。

---

## 6. Pod の dnsPolicy

Pod の `spec.dnsPolicy` で DNS 設定の動作を変更できる。

| dnsPolicy | 動作 |
|---|---|
| `ClusterFirst`（デフォルト）| CoreDNS が nameserver。クラスター内の名前解決を優先 |
| `ClusterFirstWithHostNet` | hostNetwork: true の Pod でも CoreDNS を使う |
| `Default` | Node の `/etc/resolv.conf` をそのまま使う（CoreDNS を使わない）|
| `None` | dnsConfig で完全にカスタム設定 |

```yaml
# カスタム DNS の例
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - my-company.internal
    options:
    - name: ndots
      value: "2"
```

---

## 7. CoreDNS のデプロイ構成

```yaml
# cluster/addons/dns/coredns/coredns.yaml.base（抜粋）

# CoreDNS Deployment
kind: Deployment
spec:
  template:
    spec:
      priorityClassName: system-cluster-critical  # 高優先度（Node が逼迫しても退去されにくい）
      affinity:
        podAntiAffinity:                          # 複数レプリカを別 Node に分散
          preferredDuringSchedulingIgnoredDuringExecution: ...
      tolerations:
      - key: CriticalAddonsOnly                   # CriticalAddons Taint の Node にも配置できる
        operator: Exists

# CoreDNS Service（固定の ClusterIP で常に同じ IP を提供）
kind: Service
metadata:
  name: kube-dns     # ← "kube-dns" のまま（後方互換性）
spec:
  clusterIP: 10.96.0.10   # 固定（--service-cluster-ip-range の先頭付近）
  ports:
  - port: 53               # DNS クエリ（UDP/TCP）
  - port: 9153             # Prometheus メトリクス
```

CoreDNS の ClusterIP（`10.96.0.10` など）が kubelet の `--cluster-dns` フラグで
全 Node に伝わり、Pod の `/etc/resolv.conf` に書き込まれる。

---

## 8. NodeLocal DNSCache

大規模クラスターでの CoreDNS への負荷集中を緩和するアドオン。

```
通常（CoreDNS のみ）:
  Pod → CoreDNS Pod（数台）← 全クエリが集中

NodeLocal DNSCache:
  Pod → 同じ Node の DNSCache（DaemonSet）← キャッシュヒットは高速・CoreDNS 不要
                ↓ キャッシュミスのみ
              CoreDNS Pod（負荷が大幅に減る）
```

```yaml
# cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
# DaemonSet として全 Node に配置される
kind: DaemonSet
metadata:
  name: node-local-dns
  namespace: kube-system
```

---

## 9. トラブルシュート

### DNS 解決できない

```bash
# CoreDNS Pod の状態確認
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS のログ確認
kubectl logs -n kube-system -l k8s-app=kube-dns

# Pod 内から DNS を確認
kubectl run test --rm -it --image=busybox -- nslookup kubernetes.default
# → Server: 10.96.0.10（CoreDNS の IP）
# → kubernetes.default.svc.cluster.local → 10.96.0.1（kube-apiserver の ClusterIP）
```

### 外部 DNS の解決が遅い

`ndots:5` のせいで外部ドメインに余分なクエリが走る場合。

```yaml
# dnsConfig で ndots を下げる
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"
```

または CoreDNS の `autopath` プラグインを使って search ドメイン補完を最適化する。

### CoreDNS の設定を変更する

```bash
# Corefile を編集
kubectl edit configmap coredns -n kube-system

# reload プラグインが自動で反映する（Pod 再起動不要）
```

---

## 10. コードリーディングの起点

```
pkg/kubelet/network/dns/dns.go
  └── Configurer / GetPodDNS() ← kubelet が Pod の /etc/resolv.conf を生成する処理

cluster/addons/dns/coredns/coredns.yaml.base
  └── CoreDNS の Deployment / Service / ConfigMap（Corefile）定義

cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
  └── NodeLocal DNSCache の DaemonSet 定義
```

CoreDNS 本体のコード（外部リポジトリ）:

```
github.com/coredns/coredns/plugin/kubernetes/
  └── kubernetes.go ← Service/Pod の名前解決を担う kubernetes プラグイン

github.com/coredns/coredns/plugin/forward/
  └── forward.go ← 外部 DNS への転送プラグイン
```
