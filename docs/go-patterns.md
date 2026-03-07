# Kubernetes コードの Go パターン集

Kubernetes のコードを読むときに頻出する Go のパターンをまとめる。
「文法は知っているが、実際のコードでどう使われているかわからない」という場面向けの解説。

---

## 1. goroutine：軽量な並行処理

### goroutine とは

`go` キーワードを先頭に付けて関数を呼ぶと、その関数が **別スレッドのような存在（goroutine）** として並行実行される。
OS スレッドより遥かに軽量（初期スタック数 KB）で、Kubernetes では数百〜数千の goroutine が同時に動いている。

```go
// 同期（順番に実行）
doA()
doB()  // doA が終わるまで待つ

// 非同期（並行実行）
go doA()
doB()  // doA の完了を待たずに doB も動く
```

### Kubernetes での使われ方

**Scheduler の Binding Cycle**（`pkg/scheduler/schedule_one.go`）:

```go
// Scheduling Cycle（直列）が終わったら
// Binding Cycle を goroutine で非同期起動
go sched.runBindingCycle(ctx, state, fwk, scheduleResult, assumedPodInfo, ...)
// ↑ の完了を待たずに、すぐ次の Pod のスケジューリングへ進む
```

**Deployment Controller の worker 起動**（`pkg/controller/deployment/deployment_controller.go:193`）:

```go
for i := 0; i < workers; i++ {
    wg.Go(func() {
        wait.UntilWithContext(ctx, dc.worker, time.Second)
    })
}
```

`workers` 個の goroutine を起動し、それぞれが独立して `dc.worker()` を実行し続ける。

### goroutine のライフサイクル

goroutine は **関数が return するまで生き続ける**。
`wait.UntilWithContext` のようなループ関数は `ctx` がキャンセルされるまで return しないため、
goroutine もその間ずっと動き続ける。

```
goroutine の終わり方:
  1. 関数が return する
  2. ctx がキャンセルされ、ループが終わる
  3. panic（利用可能な回復処理がなければプロセスごとクラッシュ）
```

---

## 2. channel：goroutine 間の通信

### channel とは

goroutine 間でデータを安全に受け渡すためのパイプ。
`chan` キーワードで宣言し、`<-` 演算子で送受信する。

```go
ch := make(chan string)     // バッファなし channel
ch := make(chan string, 10) // バッファあり channel（10個まで溜められる）

// 送信（goroutine A）
ch <- "hello"

// 受信（goroutine B）
msg := <-ch
```

**バッファなし**: 送信側は受信側が受け取るまでブロックする（待ち合わせ）
**バッファあり**: バッファが満杯になるまで送信側はブロックしない

### Kubernetes での使われ方

**終了シグナルの伝達**（`pkg/scheduler/scheduler.go`）:

```go
func (sched *Scheduler) Run(ctx context.Context) {
    go wait.UntilWithContext(ctx, sched.ScheduleOne, 0)
    <-ctx.Done()  // ctx がキャンセルされるまでここで待機
    sched.SchedulingQueue.Close()
}
```

`ctx.Done()` は `context` がキャンセルされたときに閉じられる channel。
`<-ctx.Done()` は channel が閉じられるまでブロックし、閉じられたら処理を続ける。

**sharedProcessor のイベント配信**（`staging/src/k8s.io/client-go/tools/cache/shared_informer.go`）:

```go
// リスナーごとにバッファ付き channel を持つ
type processorListener struct {
    addCh chan interface{}  // Informer → listener への中継チャネル
}
```

---

## 3. select：複数 channel の待ち受け

### select とは

複数の channel 操作を同時に待ち受け、**準備できた channel から処理する**。
`switch` 文に似ているが、条件ではなく channel の準備状態で分岐する。

```go
select {
case msg := <-ch1:
    // ch1 からデータが来たとき
    fmt.Println("ch1:", msg)
case msg := <-ch2:
    // ch2 からデータが来たとき
    fmt.Println("ch2:", msg)
case <-ctx.Done():
    // context がキャンセルされたとき
    return
}
```

複数の channel が同時に準備できた場合は**ランダムに1つを選ぶ**。

### Kubernetes での使われ方

コントローラや goroutine の終了処理でよく現れるパターン:

```go
for {
    select {
    case item := <-workCh:
        process(item)
    case <-stopCh:
        return  // 終了シグナルを受けたら goroutine を終了
    }
}
```

`ctx.Done()` と組み合わせて「仕事をし続けるが、終了指示が来たら止まる」というループを作る。

---

## 4. interface：Duck Typing によるポリモーフィズム

### interface とは

**「このメソッドを持っていれば、この型として使える」** という契約。
Java の interface と似ているが、Go では**明示的な `implements` 宣言が不要**。
メソッドシグネチャが一致していれば、自動的にその interface を満たしていることになる（Duck Typing）。

```go
type Animal interface {
    Sound() string
}

type Dog struct{}
func (d Dog) Sound() string { return "woof" }

type Cat struct{}
func (Cat) Sound() string { return "meow" }

// Dog も Cat も Animal interface を自動的に満たす
// "implements Animal" の宣言は不要
var a Animal = Dog{}
```

### Kubernetes での使われ方

**Scheduling Framework のプラグインシステム**:

```go
// 拡張点ごとにインターフェースが定義されている
type FilterPlugin interface {
    Filter(ctx, state, pod, nodeInfo) *Status
}
type ScorePlugin interface {
    Score(ctx, state, pod, nodeInfo) (int64, *Status)
}

// プラグインは必要なインターフェースだけ実装する
type NodeResourcesFit struct{ ... }
func (n *NodeResourcesFit) Filter(...) *Status { ... }  // FilterPlugin を満たす
func (n *NodeResourcesFit) Score(...) (int64, *Status) { ... }  // ScorePlugin も満たす
```

**型アサーション**で「このプラグインはどの拡張点を実装しているか」を確認する:

```go
// 型アサーション: interface の実態が特定の型かを確認
if fp, ok := plugin.(FilterPlugin); ok {
    // FilterPlugin を実装していれば Filter 拡張点に登録
    filterPlugins = append(filterPlugins, fp)
}
if sp, ok := plugin.(ScorePlugin); ok {
    // ScorePlugin も実装していれば Score 拡張点にも登録
    scorePlugins = append(scorePlugins, sp)
}
```

`ok` が `false` のとき panic しないのがポイント（`plugin.(FilterPlugin)` だけにすると失敗時に panic）。

**WorkQueue のインターフェース**（`staging/src/k8s.io/client-go/util/workqueue/queue.go:30`）:

```go
type TypedInterface[T comparable] interface {
    Add(item T)
    Get() (item T, shutdown bool)
    Done(item T)
    ShutDown()
}
```

コントローラは具体的な実装（rate-limited queue / delay queue 等）を意識せず、
このインターフェース経由で操作する。テスト時にモックに差し替えやすくなる。

---

## 5. context：キャンセルとタイムアウトの伝播

### context とは

`context.Context` は **「この処理をいつキャンセルするか」という情報を持ち回す**ための型。
関数の第1引数として渡すのが Go の慣習。

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    // ctx を受け取った関数はキャンセルに対応できる
    doWork(ctx)
}()

cancel()  // キャンセルを発行 → ctx.Done() channel が閉じられる
```

**なぜ必要か**:
goroutine は外部から強制終了できない。
`context` を使うことで「もう止まってください」という意思を伝え、goroutine 自身が終了できる。

### Kubernetes での使われ方

**コントローラの Run() への伝達**（`pkg/controller/deployment/deployment_controller.go:171`）:

```go
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
    for i := 0; i < workers; i++ {
        wg.Go(func() {
            wait.UntilWithContext(ctx, dc.worker, time.Second)
            // ctx がキャンセルされると UntilWithContext が return
            // → goroutine が終了する
        })
    }
    <-ctx.Done()  // キャンセルを待つ
}
```

プロセスのシャットダウン時に最上位の `ctx` をキャンセルすると、
そこから派生した全 goroutine が連鎖的に終了する。

```
main の ctx キャンセル
  └── DeploymentController.Run() の ctx がキャンセル
        └── 全 worker goroutine が UntilWithContext を抜けて終了
```

---

## 6. sync.WaitGroup：goroutine の完了待ち

### WaitGroup とは

複数の goroutine が**全て完了するのを待つ**ための同期プリミティブ。

```go
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    wg.Add(1)     // カウンタを +1
    go func() {
        defer wg.Done()  // 終了時にカウンタを -1
        doWork()
    }()
}

wg.Wait()  // カウンタが 0 になるまでブロック（全 goroutine の完了を待つ）
```

### Kubernetes での使われ方

**Deployment Controller のシャットダウン**（`pkg/controller/deployment/deployment_controller.go:182`）:

```go
var wg sync.WaitGroup
defer func() {
    dc.queue.ShutDown()
    wg.Wait()  // 全 worker goroutine の終了を待ってから return
}()

for i := 0; i < workers; i++ {
    wg.Go(func() {  // wg.Add(1) + go func() + defer wg.Done() をまとめた便利メソッド
        wait.UntilWithContext(ctx, dc.worker, time.Second)
    })
}
```

`defer` で `wg.Wait()` を登録しているため、`Run()` が return する直前に全 goroutine の終了を待つ。

---

## 7. sync.Mutex / sync.RWMutex：共有データの保護

### Mutex とは

複数の goroutine が同じデータに**同時にアクセスするとデータ競合（race condition）が起きる**。
Mutex（ミューテックス）はある goroutine がデータを使っている間、他の goroutine を待機させる。

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()         // ロック取得（他の goroutine はここで待機）
    defer c.mu.Unlock() // 処理が終わったら必ず解放
    c.count++
}
```

**RWMutex**（Read-Write Mutex）: 読み取りは複数 goroutine が同時に可能、書き込みは排他。

```go
var mu sync.RWMutex

// 読み取り（複数 goroutine が同時にできる）
mu.RLock()
defer mu.RUnlock()
return data

// 書き込み（1 goroutine だけ）
mu.Lock()
defer mu.Unlock()
data = newValue
```

### Kubernetes での使われ方

Kubernetes のキャッシュ（Indexer）は読み取りが多く書き込みが少ないため、`RWMutex` で保護されている。
`CycleState` も `sync.Map`（内部で RWMutex を使う）でスレッドセーフを実現している。

---

## 8. defer：確実な後処理

### defer とは

`defer` を付けた処理は**関数が return する直前に必ず実行される**。
エラーや panic が発生しても実行されるため、リソース解放に使う。

```go
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()  // どんな return でも必ず f.Close() が呼ばれる
    // ... ファイル処理
}
```

### Kubernetes での使われ方

**Mutex の解放**（最重要用途）:

```go
mu.Lock()
defer mu.Unlock()  // panic が起きても必ず解放される
// ... 保護されたコードブロック
```

**シャットダウン処理の登録**（`deployment_controller.go:171`）:

```go
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
    defer utilruntime.HandleCrash()  // panic 時の回復処理
    defer func() {
        dc.queue.ShutDown()
        wg.Wait()
    }()
    // ... 起動処理
}
```

`defer` は**後に書いたものから先に実行される（LIFO）**。
上記なら `queue.ShutDown()` → `HandleCrash()` の順で実行される。

---

## 9. エラーハンドリング：明示的な error 返却

### Go のエラー処理の特徴

Go には `try/catch` がない。代わりに関数の戻り値として `error` を返し、呼び出し元が必ず処理する。

```go
func doSomething() (Result, error) {
    if somethingBad {
        return Result{}, fmt.Errorf("something went wrong: %w", err)  // エラーをラップして返す
    }
    return result, nil  // 成功時は nil
}

// 呼び出し側
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err)  // さらに上位にラップして伝播
}
```

`%w` でエラーをラップすると、`errors.Is()` や `errors.As()` で元のエラーを判定できる。

### Kubernetes での使われ方

**WorkQueue でのリトライ**（コントローラの reconcile 内）:

```go
func (dc *DeploymentController) syncDeployment(ctx context.Context, key string) error {
    // ... 処理
    if err != nil {
        // エラーなら WorkQueue に再エンキュー（指数バックオフでリトライ）
        dc.queue.AddRateLimited(key)
        return err
    }
    // 成功なら再試行カウンタをリセット
    dc.queue.Forget(key)
    return nil
}
```

**apiserver エラーの判定**（`k8s.io/apimachinery/pkg/api/errors`）:

```go
import apierrors "k8s.io/apimachinery/pkg/api/errors"

_, err := client.Get(ctx, name, metav1.GetOptions{})
if apierrors.IsNotFound(err) {
    // 404 エラー → オブジェクトが存在しない（正常系として扱う場合もある）
    return nil
}
if err != nil {
    return err  // その他のエラー
}
```

---

## 10. 関数型フィールド：依存性注入とテスト容易性

### 関数を構造体フィールドに持つパターン

Go では関数も値として扱える。
構造体フィールドに関数を持たせることで、**テスト時に実装を差し替えられる**。

```go
type DeploymentController struct {
    // 実際の同期処理。テスト時は差し替えできる
    syncHandler func(ctx context.Context, dKey string) error
    // エンキュー処理。テスト時はモックに差し替えられる
    enqueueDeployment func(deployment *apps.Deployment)
}

// 本番時の初期化
dc.syncHandler = dc.syncDeployment
dc.enqueueDeployment = dc.enqueue

// テスト時は差し替え
dc.syncHandler = func(ctx context.Context, key string) error {
    recordedKeys = append(recordedKeys, key)
    return nil
}
```

これは Go における**依存性注入（Dependency Injection）** の典型パターン。
インターフェースを使わずに、関数の型だけで差し替えを実現している。

---

## 11. ジェネリクス（型パラメータ）

### ジェネリクスとは

Go 1.18 で導入された機能。**型を引数として受け取る**ことで、異なる型に対して同じロジックを使える。

```go
// T は comparable な任意の型
type TypedQueue[T comparable] struct {
    items []T
}

func (q *TypedQueue[T]) Add(item T) {
    q.items = append(q.items, item)
}

// 使用時に型を指定
stringQueue := TypedQueue[string]{}
intQueue := TypedQueue[int]{}
```

### Kubernetes での使われ方

**WorkQueue**（`staging/src/k8s.io/client-go/util/workqueue/queue.go:30`）:

```go
// T はキューに入れるアイテムの型（comparable = 比較可能な型）
type TypedInterface[T comparable] interface {
    Add(item T)
    Get() (item T, shutdown bool)
    Done(item T)
}
```

旧バージョンでは `interface{}` を使って任意の型を受け取っていたが、
ジェネリクス導入で型安全になった（間違った型を入れるとコンパイルエラーになる）。

---

## 12. wait パッケージ：ループとリトライ

### wait.UntilWithContext

**context がキャンセルされるまで、関数を一定間隔で繰り返し実行する**ユーティリティ。

```go
// ctx がキャンセルされるまで、1秒間隔で dc.worker を呼び続ける
wait.UntilWithContext(ctx, dc.worker, time.Second)
```

コントローラの worker ループや、定期的な処理の繰り返しに使う。

### wait.WaitForCacheSync

**Informer のキャッシュ同期完了を待つ**ユーティリティ。
コントローラは Informer のキャッシュが初期化される前に reconcile を開始してはいけない。

```go
// 全 Lister のキャッシュ同期を待つ（false が返ったら ctx がキャンセルされた）
if !cache.WaitForNamedCacheSyncWithContext(ctx,
    dc.dListerSynced,
    dc.rsListerSynced,
    dc.podListerSynced) {
    return  // キャンセルされたので終了
}
```

---

## 次に読むべきファイル

```
pkg/controller/deployment/deployment_controller.go:170
  └── Run() - goroutine・WaitGroup・context・defer の組み合わせ

staging/src/k8s.io/client-go/util/workqueue/queue.go:30
  └── TypedInterface - ジェネリクスを使ったインターフェース定義

pkg/scheduler/framework/parallelize/parallelism.go:66
  └── Until() - goroutine プールによる並列処理の実装

staging/src/k8s.io/client-go/tools/cache/shared_informer.go
  └── sharedProcessor - channel を使ったイベント配信

staging/src/k8s.io/kube-scheduler/framework/interface.go:419
  └── プラグインインターフェース群 - interface と型アサーションの典型例
```
