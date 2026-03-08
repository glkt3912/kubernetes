# Pod Lifecycle（Pod のライフサイクル）

## 1. Pod のフェーズ（Phase）

Pod の全体的な状態を表す。`kubectl get pod` の `STATUS` 列に相当。

```go
// pkg/apis/core/types.go:3112
const (
    PodPending   PodPhase = "Pending"
    PodRunning   PodPhase = "Running"
    PodSucceeded PodPhase = "Succeeded"
    PodFailed    PodPhase = "Failed"
    PodUnknown   PodPhase = "Unknown"
)
```

| フェーズ | 意味 |
|---|---|
| `Pending` | apiserver に受理されたが、まだ Node に配置されていない。またはイメージ Pull 中 |
| `Running` | Node に配置済み。少なくとも1つのコンテナが起動中または再起動中 |
| `Succeeded` | 全コンテナが正常終了（exit 0）。再起動しない |
| `Failed` | 全コンテナが終了し、少なくとも1つが異常終了（exit ≠ 0）|
| `Unknown` | Node と通信できず状態不明。Node 障害時に発生 |

---

## 2. Pod の起動シーケンス

```
kubectl apply (または Deployment Controller が Pod を作成)
        |
        v  kube-apiserver に Pod オブジェクトが作成される
        |  Phase: Pending
        |
        v  kube-scheduler が Node を選択
        |  Pod.spec.nodeName に Node 名をセット
        |
        v  kubelet が Watch で検知
        |
        v  ① Volume のマウント準備
        |
        v  ② Init Container を順番に実行（すべて成功するまで）
        |
        v  ③ コンテナイメージの Pull
        |
        v  ④ 通常コンテナを起動
        |     └── PostStart フック実行（コンテナと非同期だが完了まで Ready にならない）
        |
        v  ⑤ Startup Probe（設定時）
        |     → 成功するまで liveness/readiness は評価しない
        |
        v  ⑥ Readiness Probe
        |     → 成功すると Service の Endpoints に追加され、トラフィックが来る
        |
        v  ⑦ Liveness Probe（継続的に実行）
              → 失敗するとコンテナを再起動（restartPolicy に従う）
```

---

## 3. Init Container

### 役割

通常コンテナが起動する**前に**、順番に実行される初期化専用コンテナ。

```
全 Init Container が成功 → 通常コンテナが起動
Init Container が失敗   → restartPolicy に従って再試行
```

```yaml
spec:
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nc -z my-db 5432; do sleep 2; done']
    # → DB の 5432 番ポートが開くまで待ち続ける

  - name: init-migration
    image: my-app:latest
    command: ['./migrate', '--up']
    # → DB マイグレーションを実行

  containers:
  - name: app
    image: my-app:latest
    # → Init Container が全て成功してから起動する
```

### Init Container の特徴

```
・順番に実行される（並列ではない）
・1つ失敗すると Pod は再起動または Pending のまま停止
・通常コンテナとは別のイメージを使える（ツール類を本番イメージに含めなくて済む）
・完了した Init Container は再起動しない（Completed 状態で残る）
・通常コンテナの resources とは独立して設定できる
```

### Sidecar Container（1.29+ Beta）

`initContainers` に `restartPolicy: Always` を設定すると **Sidecar Container** になる。

```yaml
initContainers:
- name: log-agent
  image: fluent-bit:latest
  restartPolicy: Always   # ← Sidecar Container として動作
```

```
・Init Container の順番で起動するが、完了を待たない（Ready になれば次へ）
・通常コンテナと同じ期間動き続ける
・Pod が終了するとき、通常コンテナの後に終了する
・従来の Sidecar パターン（Pod 内の別コンテナ）の公式版
```

---

## 4. Lifecycle フック

コンテナの起動直後・終了直前に任意の処理を実行できる。

```go
// pkg/apis/core/types.go:2861
type Lifecycle struct {
    PostStart *LifecycleHandler  // 起動直後
    PreStop   *LifecycleHandler  // 終了直前
}
```

### PostStart

コンテナが起動した**直後**に実行される。

```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo 'started' >> /var/log/app.log"]
```

```
コンテナプロセス起動
     ↓（ほぼ同時）
PostStart フック実行
     ↓ 完了まで、コンテナは Ready にならない（Probe も評価されない）
     ↓ 失敗するとコンテナは終了・再起動
通常の処理へ
```

**注意**: PostStart の実行はコンテナプロセスの起動と**非同期**。
`ENTRYPOINT` より先に実行される保証はない。

### PreStop

コンテナが終了する**直前**に実行される。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "nginx -s quit; sleep 5"]
    # または httpGet:
    #   path: /shutdown
    #   port: 8080
```

```
Pod 削除リクエスト
     ↓
PreStop フック実行（terminationGracePeriodSeconds のカウントダウン開始と同時）
     ↓ 完了を待つ（または terminationGracePeriodSeconds に達するまで）
SIGTERM をコンテナプロセスに送信
     ↓ terminationGracePeriodSeconds 内に終了しなければ
SIGKILL で強制終了
```

**重要**: `terminationGracePeriodSeconds`（デフォルト 30 秒）は PreStop フックの
実行と SIGTERM → 終了待ちの**合計**時間。

```
terminationGracePeriodSeconds: 30
  ├── PreStop フック: 10 秒
  └── SIGTERM 後のアプリ終了待ち: 20 秒（残り）
```

---

## 5. Probe（ヘルスチェック）

### 3種類の Probe

```go
// 3つの Probe はそれぞれ独立して設定する
type Container struct {
    LivenessProbe  *Probe  // 失敗 → コンテナ再起動
    ReadinessProbe *Probe  // 失敗 → Endpoints から除外（トラフィックが来なくなる）
    StartupProbe   *Probe  // 失敗 → コンテナ再起動。起動完了まで他 Probe を無効化
}
```

| Probe | 失敗時の動作 | 用途 |
|---|---|---|
| Liveness | コンテナを再起動 | デッドロック・無限ループの検出 |
| Readiness | Endpoints から除外 | 起動中・一時的な高負荷・依存サービス待ち |
| Startup | コンテナを再起動 | 起動が遅いアプリ（Liveness の初回チェックを遅らせる）|

### Probe の種類

```yaml
# exec: コマンドを実行し exit code で判定
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]

# httpGet: HTTP リクエストを送り ステータスコードで判定（2xx/3xx = 成功）
readinessProbe:
  httpGet:
    path: /ready
    port: 8080

# tcpSocket: TCP 接続が確立できれば成功
livenessProbe:
  tcpSocket:
    port: 5432

# grpc: gRPC Health Checking Protocol
livenessProbe:
  grpc:
    port: 50051
```

### Probe のパラメータ

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15  # 起動してから最初の Probe まで待つ秒数
  periodSeconds: 10        # Probe の間隔
  timeoutSeconds: 5        # タイムアウト
  successThreshold: 1      # 成功とみなすための連続成功回数（liveness は 1 固定）
  failureThreshold: 3      # 失敗とみなすための連続失敗回数
```

### Startup Probe の役割

起動に時間がかかるアプリで Liveness の誤検知を防ぐ。

```
# Startup Probe なしの問題
initialDelaySeconds: 30 秒 → 起動が 30 秒未満なら OK。31 秒かかると再起動ループ

# Startup Probe ありの解決策
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30    # 30 回失敗するまで Startup とみなさない
  periodSeconds: 10       # 10 秒ごとにチェック
  # → 最大 300 秒（5分）起動時間を確保できる

→ Startup Probe が成功してから Liveness/Readiness が動き出す
```

---

## 6. RestartPolicy

コンテナが終了したときの再起動方針。

```yaml
spec:
  restartPolicy: Always  # デフォルト
```

| ポリシー | 動作 |
|---|---|
| `Always` | 終了したら常に再起動（exit 0 でも再起動）。Deployment に使う |
| `OnFailure` | 異常終了（exit ≠ 0）のときだけ再起動。Job に使う |
| `Never` | 再起動しない |

```
CrashLoopBackOff:
  コンテナが繰り返し失敗して再起動する状態。
  kubelet は指数バックオフで再起動を遅らせる:
  10s → 20s → 40s → ... → 最大 5 分
```

---

## 7. QoS クラス（Quality of Service）

kubelet がメモリ不足時にどの Pod を先に退去させるかの優先度。

```go
// pkg/apis/core/types.go:4354
const (
    PodQOSGuaranteed PodQOSClass = "Guaranteed"
    PodQOSBurstable  PodQOSClass = "Burstable"
    PodQOSBestEffort PodQOSClass = "BestEffort"
)
```

| クラス | 条件 | OOMKill 優先度 |
|---|---|---|
| Guaranteed | 全コンテナで `requests == limits`（CPU・メモリ両方） | 最後（OOMKill されにくい）|
| Burstable | 一部でも requests/limits が設定されている | 中間 |
| BestEffort | requests も limits も設定なし | 最初（OOMKill されやすい）|

```yaml
# Guaranteed の例
resources:
  requests:
    cpu: "1"
    memory: 512Mi
  limits:
    cpu: "1"       # ← requests と同じ値
    memory: 512Mi  # ← requests と同じ値

# BestEffort の例（resources を一切書かない）
containers:
- name: app
  image: myapp:latest
  # resources: なし → BestEffort
```

---

## 8. Pod の終了シーケンス（Graceful Termination）

```
kubectl delete pod / Deployment スケールダウン / Node Eviction
        |
        v  apiserver: Pod の DeletionTimestamp をセット
        |             Phase は Running のまま（Terminating は Phase ではない）
        |
        v  kubelet が変化を検知
        |
        v  ① Endpoints から Pod を除外（Service が新しいトラフィックを転送しなくなる）
        |     ※ kube-proxy が EndpointSlice の変化を検知して iptables を更新
        |
        v  ② PreStop フックを実行
        |
        v  ③ SIGTERM をコンテナに送信
        |
        v  ④ terminationGracePeriodSeconds（デフォルト 30 秒）以内に終了を待つ
        |
        v  ⑤ 時間切れなら SIGKILL で強制終了
        |
        v  ⑥ kubelet が Pod オブジェクトを削除
```

**Endpoints 除外と SIGTERM のタイミングのズレに注意**:

```
iptables の更新（kube-proxy）と PreStop + SIGTERM はほぼ同時に始まる。
iptables 更新が遅れると、SIGTERM 後もトラフィックが来てしまう場合がある。

対策:
  PreStop で少し待つ（sleep 5 など）
  アプリが SIGTERM を受けてから接続を適切にドレインする
```

---

## 9. Ephemeral Container（エフェメラルコンテナ）

実行中の Pod にデバッグ用コンテナを**一時的に追加**する機能（1.23 GA）。

```bash
# 実行中 Pod にデバッグコンテナを追加
kubectl debug -it my-pod --image=busybox --target=app

# → Pod に ephemeral container が追加される
# → Pod を再起動せずにデバッグできる
# → Pod が削除されると消える（追加後の削除はできない）
```

```yaml
# Pod Status に ephemeral container の状態が追記される
status:
  ephemeralContainerStatuses:
  - name: debugger
    state:
      running:
        startedAt: "2026-03-08T00:00:00Z"
```

distroless イメージや scratch ベースのコンテナ（シェルが入っていない）の
デバッグに有効。

---

## 10. コードリーディングの起点

```
pkg/apis/core/types.go
  :3108 PodPhase の定義
  :3132 PodConditionType の定義
  :2861 Lifecycle（PostStart / PreStop）の定義
  :4348 PodQOSClass の定義

pkg/kubelet/kubelet.go
  └── syncPod() ← Pod の同期処理（Init Container → 通常コンテナの起動フロー）

pkg/kubelet/pod_workers.go
  └── podWorker → UpdatePod() ← Pod ごとの worker goroutine

pkg/kubelet/kuberuntime/kuberuntime_manager.go
  └── SyncPod() ← CRI 経由でコンテナを起動・停止
  └── killContainer() ← PreStop フックの実行 → SIGTERM → SIGKILL の順序

pkg/kubelet/prober/
  └── liveness/readiness/startup probe の実装
  └── prober_manager.go ← Probe の定期実行を管理

pkg/kubelet/lifecycle/handlers.go
  └── PostStart / PreStop フックの実行（exec / httpGet / tcpSocket）

pkg/api/v1/pod/util.go
  └── IsPodReady() / GetPodCondition() ← Pod の条件チェック
```
