# ストレージ（PV / PVC）

## 1. なぜ PV / PVC が必要か

Pod はそのままではストレージを持てない。Pod が死ぬとデータも消える。

```
Pod（死ぬとデータ消える）
  └── コンテナ内のファイル → Pod 削除で消える
```

データを永続化するために **PersistentVolume（PV）** と
**PersistentVolumeClaim（PVC）** を使う。

```
Pod → PVC（「この容量のストレージが欲しい」）
         ↓ バインド
      PV（実際のストレージ）
         ↓
      外部ストレージ（AWS EBS / GCP PD / NFS など）
```

---

## 2. PV と PVC の役割の分担

**なぜ分けるのか**: 管理者とユーザーの関心事が異なるため。

```
管理者が気にすること        ユーザーが気にすること
  どのクラウドを使うか        何 GB 欲しいか
  どのゾーンに置くか          読み書きできるか
  コスト・性能               マウントして使えるか
  バックアップ設定
```

PV と PVC を分けることで、ユーザーは実装の詳細（どのクラウド・どこのゾーン）を
知らなくてよくなる（関心事の分離）。

```
分けない場合:
  Pod に直接 AWS EBS の情報を書く → AWS 依存・環境ごとに書き直しが必要

分けた場合:
  管理者が PV を用意 → ユーザーは「50Gi 欲しい」と PVC に書くだけ
  PV Controller が条件に合う PV を自動マッチング
```

```
管理者が用意する          ユーザーが要求する
PersistentVolume    ←→   PersistentVolumeClaim
（実ストレージの定義）      （ストレージの要求）
```

| | PV | PVC |
|---|---|---|
| 誰が作る | 管理者（または自動）| ユーザー（アプリ開発者）|
| 何を表す | 実際のストレージ | ストレージの要求 |
| スコープ | クラスター全体 | Namespace |

---

## 3. 型定義

```
pkg/apis/core/types.go
```

```go
type PersistentVolumeSpec struct {
    Capacity                      ResourceList          // 容量（例: 10Gi）
    PersistentVolumeSource        PersistentVolumeSource // 実体（EBS / NFS など）
    AccessModes                   []PersistentVolumeAccessMode
    PersistentVolumeReclaimPolicy PersistentVolumeReclaimPolicy
    StorageClassName              string
    ClaimRef                      *ObjectReference      // バインドした PVC への参照
}
```

### AccessMode（アクセスモード）

| モード | 意味 |
|---|---|
| `ReadWriteOnce` | 1つの Node から読み書き可（最も一般的）|
| `ReadOnlyMany` | 複数 Node から読み取りのみ |
| `ReadWriteMany` | 複数 Node から読み書き可（NFS など）|
| `ReadWriteOncePod` | 1つの Pod からのみ読み書き可 |

### なぜ4モードあるか

2つの軸の組み合わせで決まる。

```
軸1: 何台の Node からアクセスできるか
  Once（1台）  vs  Many（複数台）

軸2: 読み書きか、読み取りだけか
  ReadWrite    vs  ReadOnly
```

これで4通りの組み合わせができる（ReadOnlyOnce は需要がないため省略）。

```
                  1 Node       複数 Node
読み書き    ReadWriteOnce    ReadWriteMany
読み取りのみ    (省略)        ReadOnlyMany
```

**物理的な制約が背景にある**:

```
AWS EBS（ブロックストレージ）:
  ディスクを直接マウントする仕組み → 1つの Node にしか繋げない
  → ReadWriteOnce しか使えない

NFS（ネットワークファイルシステム）:
  ネットワーク越しにファイルを共有 → 複数 Node から同時アクセス可
  → ReadWriteMany / ReadOnlyMany が使える
```

`ReadWriteOncePod`（Kubernetes 1.22 以降）は `ReadWriteOnce` のさらに厳しい版。
`ReadWriteOnce` は「1 Node から複数 Pod が読み書き可」だが、
`ReadWriteOncePod` は「厳密に1 Pod だけ」に制限する。

```
ReadWriteOnce:
  Node1 の Pod A ─┐
  Node1 の Pod B ─┤→ PV（OK: 同一 Node の複数 Pod は許可）
  Node2 の Pod C ─╳ → 拒否（別 Node はダメ）

ReadWriteOncePod:
  Pod A → PV（OK: この Pod だけ）
  Pod B → PV（拒否: 他の Pod はダメ）
```

**PVC と StorageClass での指定**:
PVC に AccessMode を書いても、実際にその制約を守れるかはストレージ側の実装次第。
Kubernetes は PVC と PV の AccessMode を照合してバインドするが、
物理的に不可能な組み合わせ（EBS で ReadWriteMany など）は動かない。

### PV のフェーズ

```
Available  → Bound → Released → Failed
（未使用）   （使用中）（PVC 削除後）（エラー）
```

---

## 4. PV と PVC のバインディング

PVC が作られると **PersistentVolume Controller** がマッチする PV を探してバインドする。

```
PVC 作成（要求: 10Gi、ReadWriteOnce）
        |
        v  PV Controller が探す
  条件を満たす PV はあるか？
    ある → バインド（PV.ClaimRef = PVC、PVC.VolumeName = PV）
    ない → Pending のまま待つ
```

バインドは**双方向ポインタ**で表現される。

```go
// PV 側
pv.Spec.ClaimRef = &ObjectReference{Name: "my-pvc", Namespace: "default"}

// PVC 側
pvc.Spec.VolumeName = "my-pv"
```

これにより「どの PVC がどの PV を使っているか」が明確になる。

---

## 5. ReclaimPolicy（解放後の動作）

PVC が削除されて PV が解放されたとき、PV をどう扱うか。

| ポリシー | 動作 |
|---|---|
| `Retain` | PV を残す（データ保持）。管理者が手動で対処 |
| `Delete` | PV と実ストレージを自動削除 |
| `Recycle` | データを消して再利用（非推奨）|

```
PVC を削除
        |
        v
  PV のフェーズ: Bound → Released
        |
        v  ReclaimPolicy に従って
  Retain → PV は Released のまま残る（手動で対処）
  Delete → PV と実ストレージが自動削除される
```

---

## 6. StorageClass と動的プロビジョニング

PV を事前に手動で作るのは手間がかかる。
**StorageClass** を使うと PVC 作成時に PV を自動で作れる（動的プロビジョニング）。

```
staging/src/k8s.io/api/storage/v1/types.go
```

```go
type StorageClass struct {
    Provisioner        string                         // どのドライバが作るか
    Parameters         map[string]string              // ドライバへのパラメータ
    ReclaimPolicy      *PersistentVolumeReclaimPolicy // 解放後の動作
    VolumeBindingMode  *VolumeBindingMode             // いつバインドするか
}
```

```yaml
# StorageClass の例（AWS EBS）
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com   # CSI ドライバ
parameters:
  type: gp3
reclaimPolicy: Delete
```

```yaml
# PVC で StorageClass を指定
spec:
  storageClassName: fast   # ← この StorageClass を使う
  resources:
    requests:
      storage: 10Gi
```

```
PVC 作成（storageClassName: fast）
        |
        v  PV Controller が StorageClass を確認
  Provisioner（ebs.csi.aws.com）を呼ぶ
        |
        v
  AWS EBS ボリュームが自動作成される
        |
        v
  PV が自動作成 → PVC にバインド
```

### VolumeBindingMode

| モード | タイミング |
|---|---|
| `Immediate` | PVC 作成直後に PV を作成・バインド |
| `WaitForFirstConsumer` | Pod が作成されて Node が決まってから PV を作成 |

`WaitForFirstConsumer` は「Pod が動く Node と同じゾーンにストレージを作る」ために使う。

---

## 7. CSI（Container Storage Interface）

ストレージドライバのプラグイン仕様。CNI の Storage 版。

```
Kubernetes → CSI ドライバ → 実ストレージ
                ↑
           各クラウド・ストレージベンダーが実装
           AWS EBS CSI / GCP PD CSI / Rook-Ceph など
```

以前は in-tree プラグイン（Kubernetes 本体に組み込み）で実装していたが、
現在は全て CSI に移行中。CSI は Kubernetes の外で独立して動く。

---

## 8. Pod からの使い方

```yaml
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc    # PVC を参照

  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data     # コンテナ内のマウントパス
```

---

## 9. ライフサイクル全体図

```
管理者が StorageClass を作成
        |
        v
ユーザーが PVC を作成
        |
        v  PV Controller
  StorageClass の Provisioner を呼ぶ
        |
        v
  実ストレージ（EBS など）が作成される
        |
        v
  PV が自動作成 → PVC にバインド（Bound）
        |
        v
  Pod が PVC をマウントして起動
        |
        v  （Pod 削除）
  ストレージはそのまま残る（Pod とは独立）
        |
        v  PVC を削除
  ReclaimPolicy: Delete → PV と実ストレージも削除
  ReclaimPolicy: Retain → PV は Released のまま残る
```

---

## 10. コードリーディングの起点

```
pkg/apis/core/types.go
  └── PersistentVolume / PersistentVolumeClaim / PersistentVolumeSpec の型定義

staging/src/k8s.io/api/storage/v1/types.go
  └── StorageClass の型定義（Provisioner / ReclaimPolicy / VolumeBindingMode）

pkg/controller/volume/persistentvolume/pv_controller.go
  └── PV と PVC のバインディングロジック（Space Shuttle Style で書かれた重要コード）

pkg/controller/volume/persistentvolume/pv_controller_base.go
  └── PV Controller の Watch・ループ起動

pkg/controller/volume/attachdetach/attach_detach_controller.go
  └── PV を Node にアタッチ・デタッチする制御
```
