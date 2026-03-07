# Kubernetes ネットワーク

## 1. 全体像

Kubernetes のネットワークは大きく3つの層に分かれる。

```
+--------------------------------------------------+
|  Pod ネットワーク（CNI）                           |
|  Pod ↔ Pod の通信（クラスター内どこでも直接通信可）  |
+--------------------------------------------------+
|  Service ネットワーク                              |
|  Pod の集合に安定した IP と名前を提供               |
+--------------------------------------------------+
|  外部ネットワーク                                  |
|  NodePort / LoadBalancer / Ingress で外部公開       |
+--------------------------------------------------+
```

---

## 2. なぜ Service が必要か

Pod は起動のたびに IP アドレスが変わる。

```
Pod A（IP: 10.0.0.1）が死ぬ
  ↓ 再起動
Pod A（IP: 10.0.0.5）← IP が変わった
```

Pod の IP を直接使うと、再起動のたびに接続先が変わってしまう。

Service は Pod の前に置く**安定した仮想 IP（ClusterIP）**。

```
クライアント → Service（ClusterIP: 10.96.0.1）→ Pod（どれか1つ）
                ↑
                IP は変わらない。Pod が入れ替わっても OK
```

---

## 3. Service の型

### ClusterIP（デフォルト）

クラスター内からだけアクセスできる仮想 IP。

```
クラスター内の Pod → ClusterIP:Port → 対象 Pod（ロードバランス）
外部からはアクセス不可
```

### NodePort

各 Node の特定ポートを Listen し、Node に到達できるクライアントからアクセスできるようにする。

```
外部クライアント → NodeのIP:30080 → ClusterIP → 対象 Pod
                   ↑
                   全 Node の同じポートが開く（30000〜32767）
```

**「外部」の範囲はネットワーク設定次第**:
Kubernetes がやるのは「Node の :30080 を Listen する」だけ。
その手前のファイアウォール・セキュリティグループは Kubernetes の責務の外。

```
Kubernetes の責務: Node の :30080 を Listen する
Kubernetes の外側: ファイアウォール / セキュリティグループの設定

→ SG で全開放すればインターネット全体からアクセス可
→ SG で社内 IP のみ許可すれば社内からのみアクセス可
```

### LoadBalancer

`type: LoadBalancer` を指定するだけで、クラウドのロードバランサを**自動で作成・設定**する（自動プロビジョニング）。

```
外部クライアント → クラウド LB（外部IP）→ NodePort → ClusterIP → Pod
```

NodePort を内部で使いつつ、外部には単一 IP を提供。

```bash
# apply するだけでクラウド LB が自動作成される
kubectl apply -f service.yaml

kubectl get svc
# NAME    TYPE           EXTERNAL-IP      PORT(S)
# my-svc  LoadBalancer   203.0.113.1      80:30080/TCP
#                        ↑ クラウドが自動で割り当てた外部 IP
```

クラウドごとの実装:

| クラウド | 作られるもの |
|---|---|
| AWS | ALB / NLB |
| GCP | Cloud Load Balancer |
| Azure | Azure Load Balancer |

**オンプレでは動かない**: クラウド API がないため外部 IP が `<pending>` のまま止まる。
代替として **MetalLB** などを使う。

### ExternalName

クラスター外のドメイン名に DNS の CNAME レコードで転送するだけ。
Pod の通信を受けた CoreDNS が「このドメイン名に聞いてください」と別名転送する。

```
クラスター内 → my-service → CNAME → external.example.com → IP を返す → Pod が接続
```

**DNS レコードの種類（補足）**:

```
A レコード    : ドメイン名 → IP アドレス（直接）
CNAME レコード: ドメイン名 → 別のドメイン名（転送・委ねる）
```

CNAME は DNS の機能の一種。ExternalName Service は CNAME を使って
クラスター外のホスト名に接続先を委ねる仕組み。

**クラスター外の DNS とは**: Route53（AWS）・Cloud DNS（GCP）・社内 DNS サーバーなど、
クラスターの外で管理されている DNS に登録されているドメイン名のこと。
Kubernetes は「このドメイン名を調べてください」と転送するだけで、
そのドメインがどこに登録されているかは関知しない。

```
CoreDNS（クラスター内）→ Route53 など（クラスター外）→ IP を返す
```

**何が嬉しいか**: 接続先をコードに直書きせず、環境ごとに Service だけ変えられる。

```yaml
# 本番
spec:
  type: ExternalName
  externalName: prod-db.example.com

# 開発（コードは変えない）
spec:
  type: ExternalName
  externalName: dev-db.example.com
```

---

## 4. Service の型定義

```
staging/src/k8s.io/api/core/v1/types.go
```

```go
type ServiceSpec struct {
    Type      ServiceType           // ClusterIP / NodePort / LoadBalancer / ExternalName
    Selector  map[string]string     // どの Pod を対象にするか（ラベルで指定）
    ClusterIP string                // 仮想 IP（自動割り当て）
    Ports     []ServicePort         // 公開するポート
}

type ServiceType string
const (
    ServiceTypeClusterIP    ServiceType = "ClusterIP"
    ServiceTypeNodePort     ServiceType = "NodePort"
    ServiceTypeLoadBalancer ServiceType = "LoadBalancer"
    ServiceTypeExternalName ServiceType = "ExternalName"
)
```

---

## 5. Endpoint / EndpointSlice

Service の「実際の転送先 Pod の IP リスト」が EndpointSlice。

```
Service: my-service (ClusterIP: 10.96.0.1)
  ↓
EndpointSlice:
  - 10.0.0.1:8080  (Pod A)
  - 10.0.0.2:8080  (Pod B)
  - 10.0.0.3:8080  (Pod C)
```

Pod が増減すると EndpointSlice が自動更新される。

```
Pod が死ぬ → Endpoint Controller が検知 → EndpointSlice から削除
Pod が増える → Endpoint Controller が検知 → EndpointSlice に追加
```

---

## 6. kube-proxy の役割

**kube-proxy** は各 Node で動き、Service → Pod への転送ルールを管理する。

```
apiserver の Service/EndpointSlice を Watch
        |
        v  変更があるたびに
  iptables（または ipvs）のルールを更新
        |
        v  パケットが来たとき
  ルールに従って Pod に転送
```

kube-proxy は「パケットを転送するプロセス」ではない。
**iptables のルールを書く係**。実際の転送は Linux カーネルが行う。

名前に「proxy」とあるが、通信を仲介するわけではない。

```
一般的なプロキシ:
  クライアント → プロキシプロセス → サーバー（全パケットがプロセスを通る）

kube-proxy の実際:
  kube-proxy → iptables にルールを書く → 終わり
  パケットが来たとき: カーネルがルールに従って直接 Pod に転送
                      （kube-proxy はパケットに関与しない）
```

**向き先は kube-proxy が決めるのではない**: Service の `selector` にマッチする Pod が
EndpointSlice に記録され、kube-proxy はそれを読んで iptables に反映するだけ。

```
Service の selector → Endpoint Controller → EndpointSlice（Pod IP リスト）
                                                    ↓
                                             kube-proxy が読む
                                                    ↓
                                             iptables にルールを書く
```

### iptables モードの仕組み

**iptables** は Linux カーネルに組み込まれたパケットフィルタリングの仕組み。
「チェーン」はルールをまとめたリストで、パケットが上から順に通過する。

kube-proxy は Service が増減するたびに以下のチェーンを生成・更新する。

```go
// pkg/proxy/iptables/proxier.go
const (
    kubeServicesChain    = "KUBE-SERVICES"    // Service へのパケット入口
    kubeNodePortsChain   = "KUBE-NODEPORTS"   // NodePort へのパケット
    kubePostroutingChain = "KUBE-POSTROUTING" // SNAT（送信元IP変換）
)
```

```
パケット(宛先: ClusterIP:Port)
        |
        v  KUBE-SERVICES チェーン（Service の一覧）
  宛先 ClusterIP にマッチするルールを探す
        |
        v  KUBE-SVC-XXXXX チェーン（この Service の転送先選択）
  複数 Pod のうちランダムに1つ選ぶ（均等な確率で分岐）
  ※ 重み調整はデフォルトでは不可。Pod 数で間接的に制御するのみ
        |
        v  KUBE-SEP-XXXXX チェーン（Pod ごとの専用ルール）
  DNAT: 宛先 IP を Pod の実 IP に書き換える
        |
        v
  Pod に届く（カーネルが転送。kube-proxy はここに関与しない）
```

Service が1つ追加されるたびに KUBE-SERVICES にルールが1行追加され、
対応する KUBE-SVC / KUBE-SEP チェーンが新規作成される。

### ロードバランシングの範囲

kube-proxy のロードバランシングは**均等なランダム分散**のみ。

| 機能 | kube-proxy | Ingress | Service Mesh（Istio など）|
|---|---|---|---|
| ランダム分散 | ○ | ○ | ○ |
| 重み付き分散 | × | △ | ○ |
| ヘッダーベース振り分け | × | ○ | ○ |
| リトライ / レート制限 | × | △ | ○ |

重み調整や高度なトラフィック制御は Ingress / Service Mesh の領域。

---

## 7. CNI（Container Network Interface）

Pod 間の通信を実現するネットワークプラグインの仕様。
Kubernetes 本体には含まれず、別途インストールする。

複数 Node にまたがる Pod 間通信をどう実現するかは CNI プラグインごとに異なる。

| プラグイン | 方式 | 特徴 |
|---|---|---|
| Flannel | overlay | パケットを包んで送る。シンプルでどこでも動く |
| Calico | BGP | ルーターに Pod の経路を直接教える。高速 |
| Cilium | eBPF | カーネルに直接プログラムを埋め込む。最高速・高機能 |
| WeaveNet | mesh | 全 Node が全対全で直接接続 |

**overlay（Flannel）**: パケットをカプセル化して Node 間を通す。ルーターへの設定不要。

```
Pod A のパケット（宛先: Pod B の IP）
  ↓ Node IP で包む（カプセル化）
Node 間のネットワークで転送
  ↓ 包みを解く（デカプセル化）
Pod B に届く
```

**BGP（Calico）**: ルーターに「この IP 帯はこの Node にある」と通知する。包まない分高速。

**eBPF（Cilium）**: iptables を使わずカーネル内で直接処理。kube-proxy の代替も可能。

**mesh（WeaveNet）**: 全 Node がお互いに直接接続した網目状の overlay。

CNI の責務:

```
Pod 起動時:
  1. veth pair（仮想ネットワークケーブル）を作る
  2. Pod 側を Pod の Network Namespace に入れる
  3. Node 側を bridge または直接ルーティング
  4. Pod に IP アドレスを割り当てる

Pod 削除時:
  1. veth pair を削除
  2. IP アドレスを解放
```

---

## 8. NetworkPolicy（通信制限）

デフォルトでは Pod 間の通信はすべて許可。
NetworkPolicy でホワイトリスト方式に制限できる。

```go
// pkg/apis/networking/types.go
type NetworkPolicy struct {
    Spec NetworkPolicySpec
}

type NetworkPolicySpec struct {
    PodSelector metav1.LabelSelector  // どの Pod に適用するか
    Ingress     []NetworkPolicyIngressRule  // 受信ルール
    Egress      []NetworkPolicyEgressRule   // 送信ルール
}
```

```yaml
# frontend Pod への通信を backend Pod からのみ許可する例
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-only
spec:
  podSelector:
    matchLabels:
      app: frontend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```

**デフォルトはブラックリスト（全許可）**:
NetworkPolicy を何も設定しないと Pod 間の通信は全て許可される。

**NetworkPolicy を適用するとホワイトリストになる**:
適用した Pod への通信は一旦全て拒否され、明示的に書いたものだけ許可される。

```
NetworkPolicy なし → 全許可（ブラックリスト）
NetworkPolicy あり → 全拒否 + 書いたものだけ許可（ホワイトリスト）

backend → frontend  ✓（ingress に明示した）
other   → frontend  ✗（書いていない = 拒否）
```

NetworkPolicy を適用していない Pod はブラックリスト（全許可）のまま変わらない。

NetworkPolicy の実施は **CNI プラグイン**が担う（kube-proxy ではない）。
Flannel は NetworkPolicy を実装していない。Calico / Cilium は実装している。

---

## 9. DNS（Service の名前解決）

Kubernetes は **CoreDNS** を内蔵し、Service 名で Pod から接続できる。

```
my-service.my-namespace.svc.cluster.local
↑          ↑             ↑   ↑
Service名  Namespace名   固定 クラスタードメイン
```

同じ Namespace なら Service 名だけで接続できる。

```
# 同一 Namespace
curl http://my-service:8080

# 別 Namespace
curl http://my-service.other-namespace:8080
```

### Namespace とは

リソースを分離する「仕切り」。同じ名前の Pod でも Namespace が違えば別物。

CoreDNS 自体も Pod として `kube-system` Namespace で動いている。

```bash
kubectl get pods -n kube-system
# NAME                READY
# coredns-xxx         1/1   ← CoreDNS
# kube-proxy-xxx      1/1
# kube-apiserver-xxx  1/1
```

**代表的な Namespace**:

| Namespace | 用途 |
|---|---|
| `default` | 何も指定しないと使われる |
| `kube-system` | Kubernetes のシステムコンポーネント（CoreDNS・kube-proxy など）|
| `kube-public` | 全ユーザーが読める公開情報 |
| `kube-node-lease` | Node の死活監視（NodeLease）|

`kube-system` を分けるのは「ユーザーのアプリと混在させず、誤操作を防ぐため」。

---

## 10. コードリーディングの起点

```
staging/src/k8s.io/api/core/v1/types.go:5942
  └── ServiceSpec の型定義（Type / Selector / ClusterIP / Ports）

pkg/proxy/types.go
  └── Provider インターフェース（kube-proxy の抽象）

pkg/proxy/iptables/proxier.go
  └── iptables モードの実装（KUBE-SERVICES チェーン生成）

pkg/proxy/servicechangetracker.go
  └── Service の変更を追跡して差分を計算

pkg/apis/networking/types.go
  └── NetworkPolicy の型定義

cmd/kube-proxy/
  └── kube-proxy のエントリポイント
```

---

## 11. IP アドレスと CIDR の基礎

### IP アドレスとは

ネットワーク上の「住所」。どこに届けるかを識別する番号。

```
192.168.1.5

4つの数字（0〜255）をドットで区切ったもの。
実体は 32 ビットの数値。

192    .168    .1      .5
11000000.10101000.00000001.00000101  ← 2進数表現
```

IP アドレスは2つの部分からなる：

```
192.168.1.5  のうち

192.168.1  → ネットワーク部（「この建物」を表す）
        .5 → ホスト部（「建物内の何号室」を表す）

どこで区切るかを示すのが CIDR。
```

### CIDR（サイダー）とは

「どこまでがネットワーク部か」を `/数字` で表す記法。

```
192.168.1.0/24

/24 = 先頭から 24 ビットがネットワーク部
    = 192.168.1. までが固定、最後の 1 バイトが変動

使えるアドレス数:
  32 - 24 = 8 ビットがホスト部
  2の8乗 = 256 個（先頭・末尾は特殊用途のため実際は 254 台）
```

`/` の数字とホスト数の関係：

```
/8  → ホスト部 24 ビット → 約 1677 万台（大規模）
/16 → ホスト部 16 ビット → 約 6.5 万台
/24 → ホスト部  8 ビット → 254 台（よく使う）
/32 → ホスト部  0 ビット → 1台だけ（特定の1台を指す）
```

### プライベート IP とパブリック IP

```
プライベート IP（組織内だけで使う、インターネットには出ない）:
  10.0.0.0/8       → 大規模組織・クラウド VPC
  172.16.0.0/12    → 中規模
  192.168.0.0/16   → 家庭用ルーター・小規模

パブリック IP（インターネット上で世界に一意）:
  上記以外のアドレス
```

### Kubernetes での使われ方

```
Pod ネットワーク: 10.244.0.0/16
  → 10.244.0.1 〜 10.244.255.254 の範囲で Pod に IP を割り当てる
  → 約 6.5 万 Pod 分の IP がある

Service ネットワーク: 10.96.0.0/12
  → この範囲内で ClusterIP を自動割り当て

Node ごとのサブネット（例）:
  Node1 → 10.244.1.0/24（この Node の Pod は 10.244.1.x を使う）
  Node2 → 10.244.2.0/24（この Node の Pod は 10.244.2.x を使う）
```

CNI プラグインが Pod 起動時にその Node のサブネットから空き IP を割り当てる。
Node ごとにサブネットを分けることで、IP の重複なく Pod 間通信が成立する。

```
外部公開の橋渡し:
  Pod / Node はプライベート IP を使う
       ↓
  LoadBalancer / NodePort でパブリック IP に橋渡し
       ↓
  インターネットからアクセス可能になる
```
