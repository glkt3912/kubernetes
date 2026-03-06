# コントローラパターン実装ガイド

## 1. コントローラとは何か

コントローラは **「クラスタのあるべき状態（desired state）を現在の状態（current state）に近づけ続けるループ処理」** を実装したコンポーネントである。
Kubernetes では Deployment、ReplicaSet、Job、StatefulSet など数十のコントローラが `kube-controller-manager` の中で並行して動いており、
それぞれが独立した goroutine として実行される。
すべてのコントローラは **Informer → WorkQueue → reconcile** という共通のパターンで実装されている。

---

## 2. 全体処理フロー

```
  Informer（Deployment / ReplicaSet / Pod）
      │
      │ OnAdd / OnUpdate / OnDelete
      │  ↓
      │ enqueue("default/my-deployment")  ← キーだけ積む
      v
  ┌──────────────────────────────────────────┐
  │           WorkQueue                      │
  │  [default/my-app, default/other, ...]    │
  │  （rate-limiting + deduplication）        │
  └──────────────────────────────────────────┘
      │  queue.Get()
      │
      v
  ┌──────────────────────────────────────────┐
  │  worker goroutine × N（並行数）           │
  │                                          │
  │  processNextWorkItem()                   │
  │    └── syncDeployment(ctx, key)          │
  │          │                               │
  │          ├─ Lister でキャッシュ読み取り   │
  │          ├─ あるべき状態 vs 現在状態 比較 │
  │          ├─ 差分を解消（API 呼び出し）    │
  │          └─ 成功 → Forget(key)           │
  │             失敗 → AddRateLimited(key)   │
  └──────────────────────────────────────────┘
```

---

## 3. Deployment Controller を読む

Deployment Controller は「Deployment が管理する ReplicaSet と Pod を常に spec 通りの状態に保つ」コントローラである。
コードは `pkg/controller/deployment/` 以下に分割されている。

```
deployment_controller.go  ← 構造体定義・初期化・Run・worker・syncDeployment
sync.go                    ← syncStatusOnly, sync（スケーリング・一時停止）
rolling.go                 ← ローリングアップデート戦略
recreate.go                ← Recreate 戦略
rollback.go                ← ロールバック処理
progress.go                ← デプロイ進捗の条件更新
```

### 3-1. 構造体（DeploymentController）

```go
// pkg/controller/deployment/deployment_controller.go:67
type DeploymentController struct {
    client    clientset.Interface      // apiserver への書き込み用クライアント

    dLister   appslisters.DeploymentLister  // Deployment キャッシュ（読み取り専用）
    rsLister  appslisters.ReplicaSetLister  // ReplicaSet キャッシュ
    podLister corelisters.PodLister         // Pod キャッシュ

    dListerSynced  cache.InformerSynced  // 初回 List 完了フラグ
    rsListerSynced cache.InformerSynced
    podListerSynced cache.InformerSynced

    queue workqueue.TypedRateLimitingInterface[string]  // reconcile キュー
}
```

**設計のポイント**:
- `client` は **書き込み専用**（Create/Update/Delete）
- `dLister` / `rsLister` / `podLister` は **読み取り専用**（Indexer のキャッシュ参照）
- 読み書きを明確に分離することで、reconcile 内のアクセスパターンが明快になる

### 3-2. 初期化（NewDeploymentController）

`NewDeploymentController()` では3種類の Informer に EventHandler を登録する。

```go
// pkg/controller/deployment/deployment_controller.go:123
dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { dc.addDeployment(logger, obj) },
    UpdateFunc: func(old, new interface{}) { dc.updateDeployment(logger, old, new) },
    DeleteFunc: func(obj interface{}) { dc.deleteDeployment(logger, obj) },
})
rsInformer.Informer().AddEventHandler(...)  // ReplicaSet 変化も監視
podInformer.Informer().AddEventHandler(...)  // Pod 削除も監視（Recreate 戦略用）
```

**なぜ ReplicaSet と Pod の Informer も監視するのか**:

```
Deployment を監視するだけでは気づけないケース:

  ReplicaSet が誰かに削除された
    → ReplicaSet の DeleteFunc → 親の Deployment キーをエンキュー

  Pod が削除された（Recreate 戦略中）
    → Pod の DeleteFunc → 全 Pod がゼロになったら Deployment をエンキュー
```

イベントを受け取るのは子リソース（RS/Pod）だが、**キューに入れるのは常に親の Deployment のキー**。
reconcile は常に Deployment 単位で行われる。

### 3-3. エンキュー処理

```go
// pkg/controller/deployment/deployment_controller.go:414
func (dc *DeploymentController) enqueue(deployment *apps.Deployment) {
    key, err := controller.KeyFunc(deployment)  // → "default/my-deployment"
    dc.queue.Add(key)
}
```

`controller.KeyFunc` は `cache.MetaNamespaceKeyFunc` の薄いラッパーで、
`namespace/name` 形式のキー文字列を生成する。
WorkQueue の `Add()` は重複排除を自動的に行うため、
短時間に複数のイベントが来ても同じキーは1つしかキューに残らない。

### 3-4. Run：Worker の起動

```go
// pkg/controller/deployment/deployment_controller.go:171
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
    // 1. 全 Informer の初回 List 完了を待つ（必須）
    if !cache.WaitForNamedCacheSyncWithContext(ctx, dc.dListerSynced, ...) {
        return
    }
    // 2. N 個の worker goroutine を起動
    for i := 0; i < workers; i++ {
        wg.Go(func() {
            wait.UntilWithContext(ctx, dc.worker, time.Second)
        })
    }
    <-ctx.Done()
}
```

**`WaitForNamedCacheSyncWithContext` が必須な理由**:
Informer の初回 List が完了する前は Indexer（キャッシュ）が空または不完全。
この状態で reconcile を走らせると「Deployment が存在しない」と誤判断し、
不要な ReplicaSet 削除などの誤操作を引き起こす可能性がある。

`wait.UntilWithContext` は worker がパニックやエラーで終了しても
`time.Second` 間隔で自動的に再起動する安全網となっている。

### 3-5. worker → processNextWorkItem

```go
// pkg/controller/deployment/deployment_controller.go:496
func (dc *DeploymentController) worker(ctx context.Context) {
    for dc.processNextWorkItem(ctx) {}
}

func (dc *DeploymentController) processNextWorkItem(ctx context.Context) bool {
    key, quit := dc.queue.Get()   // ブロッキング取得
    if quit { return false }
    defer dc.queue.Done(key)       // 処理完了を通知（必須）

    err := dc.syncHandler(ctx, key)  // = syncDeployment()
    dc.handleErr(ctx, err, key)
    return true
}
```

**`queue.Done(key)` が必須な理由**:
WorkQueue は `Get()` したキーを「処理中」としてマークする。
`Done()` を呼ばないと、同じキーが再度 `Add()` されても取り出せない状態のまま残る。
`defer` で確実に呼ぶのが定石。

### 3-6. エラーハンドリング（handleErr）

```go
// pkg/controller/deployment/deployment_controller.go:514
func (dc *DeploymentController) handleErr(ctx context.Context, err error, key string) {
    if err == nil {
        dc.queue.Forget(key)   // リトライカウントをリセット
        return
    }
    if dc.queue.NumRequeues(key) < maxRetries {  // maxRetries = 15
        dc.queue.AddRateLimited(key)  // 指数バックオフでリトライ
        return
    }
    // 15 回失敗したらキューから除外（次の変更イベントで再試行される）
    dc.queue.Forget(key)
}
```

**リトライのバックオフ間隔**（`DefaultTypedControllerRateLimiter`）:

```
1回目: 5ms
2回目: 10ms
3回目: 20ms
...
15回目: 約82秒
```

`AddRateLimited` は内部でレート制限付きの遅延 Add を行う。
同じキーが連続して失敗しても、apiserver を過負荷にしない設計になっている。

### 3-7. syncDeployment：reconcile 本体

```go
// pkg/controller/deployment/deployment_controller.go:589
func (dc *DeploymentController) syncDeployment(ctx context.Context, key string) error {
    namespace, name, _ := cache.SplitMetaNamespaceKey(key)

    // 1. Lister（キャッシュ）から Deployment を取得
    deployment, err := dc.dLister.Deployments(namespace).Get(name)
    if errors.IsNotFound(err) {
        return nil  // 削除済みなら何もしない
    }

    d := deployment.DeepCopy()  // キャッシュを直接変更しないようコピー

    // 2. 管理下の ReplicaSet 一覧を取得（キャッシュ経由）
    rsList, _ := dc.getReplicaSetsForDeployment(ctx, d)

    // 3. 状態に応じて処理を分岐
    if d.DeletionTimestamp != nil {
        return dc.syncStatusOnly(ctx, d, rsList)  // 削除中はステータス更新のみ
    }
    if d.Spec.Paused {
        return dc.sync(ctx, d, rsList)            // 一時停止中はスケーリングのみ
    }
    if getRollbackTo(d) != nil {
        return dc.rollback(ctx, d, rsList)        // ロールバック
    }
    if scalingEvent, _ := dc.isScalingEvent(ctx, d, rsList); scalingEvent {
        return dc.sync(ctx, d, rsList)            // レプリカ数変更
    }

    // 4. 通常のロールアウト戦略を実行
    switch d.Spec.Strategy.Type {
    case apps.RollingUpdateDeploymentStrategyType:
        return dc.rolloutRolling(ctx, d, rsList)
    case apps.RecreateDeploymentStrategyType:
        return dc.rolloutRecreate(ctx, d, rsList, podMap)
    }
}
```

**`DeepCopy()` が必要な理由**:
`dLister.Get()` が返すのは Indexer（インメモリキャッシュ）内のポインタ。
直接変更するとキャッシュが汚染され、他の goroutine からキャッシュを読んでいるコードに影響する。
`DeepCopy()` でコピーを作ってから変更するのが Kubernetes コントローラの鉄則。

---

## 4. Deployment Controller のオーナーシップモデル

Deployment Controller は **Deployment → ReplicaSet → Pod** という2段階のオーナーシップを管理する。

```
Deployment: my-app (Spec.Replicas: 3)
  │
  ├── OwnerReference
  ▼
ReplicaSet: my-app-7d9f8b  (podTemplateHash=7d9f8b, Spec.Replicas: 3)
  │
  ├── OwnerReference × 3
  ▼
  Pod: my-app-7d9f8b-xxxxx
  Pod: my-app-7d9f8b-yyyyy
  Pod: my-app-7d9f8b-zzzzz
```

**Deployment Controller が直接管理するのは ReplicaSet だけ**。
Pod の増減は ReplicaSet Controller に委譲される。
ローリングアップデート時は新しい ReplicaSet を作り、
旧 RS の replicas を徐々に減らしながら新 RS の replicas を増やす。

**ControllerRef（OwnerReference）**:
各リソースの `metadata.ownerReferences` に「誰が親か」が記録されている。
`addReplicaSet()` ハンドラが RS の OwnerReference を辿って親 Deployment を特定し、
その Deployment をエンキューする仕組みがここに使われている。

---

## 5. よくある疑問 Q&A

**Q: worker を複数起動（workers > 1）すると同じ Deployment が並行処理されないのか？**

WorkQueue は `Get()` されたキーを「処理中」としてロックする。
同じキーが再度 `Add()` されても、`Done()` が呼ばれるまで他の worker から取り出せない。
→ 同一キーの並行処理は発生しない（コメントにも "never invoked concurrently with the same key" とある）。
異なる Deployment は別々の worker で並行処理される。

**Q: reconcile の中で apiserver から最新データを取得すべきか？**

原則としてキャッシュ（Lister）を使う。
ただし書き込み操作（Update/Patch）後に競合（conflict）エラーが返ったときは、
最新バージョンを取り直して再試行する。
`syncDeployment` の冒頭の `dLister.Get()` もキャッシュ参照。

**Q: maxRetries（15回）を超えたらどうなる？**

`queue.Forget(key)` が呼ばれリトライカウントがリセットされ、キューから除外される。
ただし次回そのリソースに変更イベントが来れば再びエンキューされるため、
「永遠に無視される」わけではない。

**Q: `enqueueAfter` と `enqueueRateLimited` の違いは？**

- `enqueueAfter(d, duration)`: 指定時間後に確実に1回エンキュー（ProgressDeadline チェックなど定期処理用）
- `enqueueRateLimited(d)`: エラーリトライ用のバックオフ付きエンキュー

**Q: Deployment が削除されたとき、ReplicaSet も自動で消えるのか？**

GarbageCollector Controller（`pkg/controller/garbagecollector/`）が OwnerReference を見て
親が削除されたら子も削除する。Deployment Controller 自身は削除時に `syncStatusOnly()` を呼ぶだけ。

---

## 6. コントローラ実装の骨格（テンプレート）

Kubernetes コントローラを自作する際の基本パターン：

```go
type MyController struct {
    client     clientset.Interface
    myLister   listers.MyResourceLister
    listerSynced cache.InformerSynced
    queue      workqueue.TypedRateLimitingInterface[string]
}

func NewMyController(informer informers.MyResourceInformer, client clientset.Interface) *MyController {
    c := &MyController{
        client: client,
        queue: workqueue.NewTypedRateLimitingQueue(...),
    }
    informer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    func(obj interface{}) { c.enqueue(obj) },
        UpdateFunc: func(_, obj interface{}) { c.enqueue(obj) },
        DeleteFunc: func(obj interface{}) { c.enqueue(obj) },
    })
    c.myLister = informer.Lister()
    c.listerSynced = informer.Informer().HasSynced
    return c
}

func (c *MyController) Run(ctx context.Context, workers int) {
    if !cache.WaitForNamedCacheSyncWithContext(ctx, c.listerSynced) { return }
    for i := 0; i < workers; i++ {
        go wait.UntilWithContext(ctx, c.worker, time.Second)
    }
    <-ctx.Done()
}

func (c *MyController) worker(ctx context.Context) {
    for {
        key, quit := c.queue.Get()
        if quit { return }
        if err := c.sync(ctx, key); err != nil {
            c.queue.AddRateLimited(key)
        } else {
            c.queue.Forget(key)
        }
        c.queue.Done(key)
    }
}

func (c *MyController) sync(ctx context.Context, key string) error {
    ns, name, _ := cache.SplitMetaNamespaceKey(key)
    obj, err := c.myLister.MyResources(ns).Get(name)
    if errors.IsNotFound(err) { return nil }

    // あるべき状態 vs 現在状態を比較して差分を解消
    return nil
}
```

---

## 7. 次に読むべきファイル

### Deployment Controller 本体

```
pkg/controller/deployment/deployment_controller.go:589
  └── syncDeployment() ← reconcile のエントリポイント

pkg/controller/deployment/rolling.go
  └── rolloutRolling() ← ローリングアップデートの実装
      → 新 RS の replicas を増やしながら旧 RS を縮小する処理
```

### WorkQueue の実装

```
staging/src/k8s.io/client-go/util/workqueue/queue.go:30
  └── TypedInterface ← WorkQueue のインターフェース定義

staging/src/k8s.io/client-go/util/workqueue/rate_limiting_queue.go
  └── TypedRateLimitingInterface ← AddRateLimited/Forget の定義

staging/src/k8s.io/client-go/util/workqueue/default_rate_limiters.go
  └── ItemExponentialFailureRateLimiter ← バックオフの実装
```

### OwnerReference / GC の仕組み

```
pkg/controller/garbagecollector/garbagecollector.go
  └── 親リソース削除時に子を自動削除する GarbageCollector

staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go
  └── OwnerReference 型定義
```

### 他のシンプルなコントローラ（学習用）

```
pkg/controller/namespace/namespace_controller.go
  └── Namespace 削除時のクリーンアップ処理（比較的シンプル）

pkg/controller/job/job_controller.go
  └── Job/CronJob の完了管理（状態遷移が明快）
```
