# GC と OwnerReference

## 1. 一言で言うと

**Kubernetes の Garbage Collection（GC）は、オーナーが消えた依存オブジェクトを自動削除する仕組み**。

```
Deployment（オーナー）を削除
  ↓
GC が OwnerReference を辿って連鎖削除
  ↓
ReplicaSet → Pod → （コンテナ停止）
```

「誰がこのオブジェクトを作ったか」を `metadata.ownerReferences` に記録しておくことで、
オーナー消滅時に依存オブジェクトも一緒に消せる。

---

## 2. OwnerReference の型定義

```
staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go
```

```go
type OwnerReference struct {
    APIVersion         string    // オーナーの APIVersion（例: "apps/v1"）
    Kind               string    // オーナーの Kind（例: "Deployment"）
    Name               string    // オーナーの名前
    UID                types.UID // オーナーの UID（名前変更に対して安全）
    Controller         *bool     // true = このオーナーが「管理コントローラ」
    BlockOwnerDeletion *bool     // true = オーナーの foreground 削除をブロック
}
```

### UID で参照する理由

名前ではなく **UID** を使う設計理由は、同名オブジェクトの再作成に対する安全性。

```
1. ReplicaSet "rs-v1" が作成される（UID: aaa）
2. Deployment が "rs-v1" を削除して再作成（UID: bbb）
3. Pod の ownerRef.UID=aaa はもはや存在しない UID を指す
   → GC はこの Pod を孤立オブジェクトと判断して削除できる
```

---

## 3. GarbageCollector の構造

```
pkg/controller/garbagecollector/garbagecollector.go
```

```
GarbageCollector
  ├── dependencyGraphBuilder（GraphBuilder）
  │     全リソースの Add/Update/Delete を Watch
  │     UID をキーにした依存グラフ（uidToNode）を構築
  │
  ├── attemptToDelete queue
  │     「削除を試みるべきオブジェクト」のキュー
  │
  └── attemptToOrphan queue
        「依存オブジェクトを孤立させてから削除するオブジェクト」のキュー
```

### node（グラフのノード）

```
pkg/controller/garbagecollector/graph.go
```

```go
type node struct {
    identity           objectReference    // 識別子（UID + GVR + Namespace/Name）
    dependents         map[*node]struct{} // 自分を参照している子オブジェクト群
    owners             []metav1.OwnerReference // 自分のオーナー一覧
    beingDeleted       bool
    deletingDependents bool               // foreground 削除中フラグ
    virtual            bool               // informer で未観測のオーナー（仮想ノード）
}
```

---

## 4. 依存グラフの構築

### GraphBuilder の役割

```
全種類のリソースを Informer で監視
  ↓
Add イベント: ノードを追加、ownerReferences を解析して親子エッジを張る
Update イベント: ownerReferences が変化したらエッジを更新
Delete イベント: ノードを削除、依存オブジェクトを attemptToDelete に積む
```

### 仮想ノード（virtual node）

ownerReference が指すオーナーが Informer のキャッシュにまだ存在しない場合、
GC は **仮想ノード** を生成してグラフに追加する。
後で実オブジェクトのイベントが届いたら `markObserved()` で仮想状態を解除する。

```
Pod が作成される（ownerRef → ReplicaSet "rs-abc"）
  ↓
rs-abc がまだ Watch に現れていない
  ↓
仮想ノード "rs-abc" を作成してグラフに登録
  ↓
rs-abc の Add イベント到着 → 仮想ノードを実ノードに昇格
```

---

## 5. 削除の 3 パターン

### 5-1. Background 削除（デフォルト）

```
kubectl delete deployment my-app
  ↓
Deployment をただちに削除
  ↓
GC がバックグラウンドで依存オブジェクトを非同期削除
```

ユーザーから見ると即座に Deployment が消えるが、ReplicaSet/Pod は少し後に消える。

### 5-2. Foreground 削除

```
kubectl delete deployment my-app --cascade=foreground
  ↓
Deployment に DeletionTimestamp セット
Deployment に finalizer: foregroundDeletion 追加
  ↓
GC が依存オブジェクトをすべて削除完了するまで待機
  ↓
依存がなくなったら finalizer を除去 → Deployment が実際に削除される
```

**用途**: 依存オブジェクトが確実に消えてから親が消えることを保証したい場合。

### 5-3. Orphan 削除

```
kubectl delete deployment my-app --cascade=orphan
  ↓
ReplicaSet の ownerReferences から Deployment への参照を削除
  ↓
Deployment のみ削除。ReplicaSet は残る（孤立）
```

**用途**: ReplicaSet を引き継いで別の管理下に置く場合など。

---

## 6. GC の削除判定ロジック

```
pkg/controller/garbagecollector/garbagecollector.go: attemptToDeleteItem()
```

```
対象オブジェクトの ownerReferences を取得
  ↓
各 ownerRef を classifyReferences() で分類:
  ├── solid: オーナーが存在し、削除待ちでもない
  ├── dangling: オーナーが存在しない
  └── waitingForDependentsDeletion: オーナーが削除中（foreground）

判定:
  ├── solid が 1 つ以上ある → 削除しない（オーナーが生きている）
  ├── 全て dangling → オブジェクトを削除
  └── waitingForDependentsDeletion → foreground 削除の連鎖
```

### isDangling チェック

```
absentOwnerCache に記録済み → dangling
  ↓ なければ
API Server に GET リクエスト
  ├── 404 → dangling（キャッシュに記録）
  ├── UID が一致しない → dangling
  └── 存在する → dangling でない
```

パフォーマンスのため、存在しないオーナーは `absentOwnerCache` に記録する。

---

## 7. Finalizers の仕組み

Finalizers はオブジェクトの削除を「意図的に遅らせる」仕組み。

```
オブジェクトに finalizers が設定されている場合:
  kubectl delete → DeletionTimestamp がセット（論理削除）
  ↓
  実際の削除はされない。オブジェクトは残り続ける
  ↓
  コントローラが処理完了後に finalizer を除去
  ↓
  finalizers が空になった瞬間、etcd から物理削除される
```

### DeletionTimestamp のセマンティクス

```go
// metadata.deletionTimestamp がセットされている = 「削除待ち」
// finalizers が [] になるまで物理削除されない
```

**設計の理由**: 外部リソースのクリーンアップ（クラウドのロードバランサ削除など）を
Kubernetes のオブジェクトライフサイクルと同期させるため。

### 代表的な Finalizers

| Finalizer | 設定者 | 役割 |
|---|---|---|
| `kubernetes.io/pvc-protection` | PVC Protection Controller | 使用中 PVC の削除阻止 |
| `foregroundDeletion` | GC | foreground 削除の連鎖処理 |
| `orphan` | GC | orphan 削除時の ownerRef 除去 |
| カスタム Finalizer | ユーザー実装コントローラ | 外部リソースのクリーンアップ |

---

## 8. GC のリソース監視

GC は kube-apiserver の Discovery API から「削除可能なリソース」一覧を取得し、
全種類のリソースを Metadata-only Informer で監視する。

```
pkg/controller/garbagecollector/garbagecollector.go: Sync()
  ↓
Discovery API で GVR 一覧を取得
  ↓
新しいリソースが増えていれば resyncMonitors()
  ↓
RestMapper をリセットして新しい CRD にも対応
```

**Metadata-only Informer**: オブジェクト全体ではなく `metadata`（UID / ownerReferences / labels）だけを
Watch する。GC に必要な情報だけを取得し、API Server とメモリへの負荷を最小化する。

---

## 9. 削除の連鎖図（Background 削除の例）

```
① kubectl delete deployment my-app
        ↓ API Server が Deployment を削除
② GraphBuilder が Delete イベントを受信
        ↓ Deployment ノードを削除グラフから除去
        ↓ 子の ReplicaSet ノードを attemptToDelete に投入
③ GC worker が ReplicaSet を処理
        ↓ isDangling チェック: Deployment UID は存在しない → dangling
        ↓ ReplicaSet を DELETE
④ GraphBuilder が ReplicaSet Delete イベントを受信
        ↓ 子の Pod ノードを attemptToDelete に投入
⑤ GC worker が Pod を処理
        ↓ isDangling チェック: ReplicaSet UID は存在しない → dangling
        ↓ Pod を DELETE
```

---

## 10. コードリーディングの起点

| 処理 | ファイル |
|---|---|
| GarbageCollector 本体 | `pkg/controller/garbagecollector/garbagecollector.go` |
| 依存グラフ構築 | `pkg/controller/garbagecollector/graph.go` |
| GraphBuilder（Watch 処理） | `pkg/controller/garbagecollector/graph_builder.go` |
| OwnerReference 型定義 | `staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go` |
| Patch 処理 | `pkg/controller/garbagecollector/patch.go` |
