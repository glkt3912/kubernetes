# Pod Security Admission（PSA）

## 1. なぜ必要か

デフォルトでは Pod は特権的な操作（ホストのネットワーク使用・root で実行等）が可能。
マルチテナントクラスターや本番環境では、Pod が実行できる操作を制限する必要がある。

```
制限なし（デフォルト）:
  Pod が hostPID: true で実行 → 他 Pod のプロセスが見える
  Pod が privileged: true    → Node のカーネル操作が可能
  Pod が root で実行          → コンテナエスケープのリスク

PSA で制限:
  Namespace のラベルでセキュリティポリシーを適用
  → 違反する Pod の作成を自動で拒否または警告
```

---

## 2. Pod Security Standards（3つのプロファイル）

```
privileged  → 制限なし（信頼できるシステムコンポーネント向け）
    ↓
baseline    → 既知の特権昇格を防ぐ最低限の制限（一般アプリ向け）
    ↓
restricted  → 最も厳しい制限（セキュリティ重視の本番環境向け）
```

| プロファイル | 禁止される主な設定 |
|---|---|
| `privileged` | なし（全許可）|
| `baseline` | privileged container / hostPID / hostIPC / hostNetwork / hostPort / allowPrivilegeEscalation（一部）|
| `restricted` | baseline の全制限 + root での実行禁止 / seccompProfile 必須 / capabilities drop ALL 必須 |

---

## 3. PSA の動作モード

各プロファイルは3つのモードで動作する。

| モード | 動作 |
|---|---|
| `enforce` | 違反する Pod の作成を**拒否**する |
| `audit` | 違反しても作成は許可するが、監査ログに記録する |
| `warn` | 違反しても作成は許可するが、**警告**をユーザーに返す |

```yaml
# Namespace のラベルで設定
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    # enforce: 違反する Pod を拒否
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    # warn: 警告のみ（移行期間中に使う）
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest

    # audit: 監査ログに記録
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

---

## 4. 内部実装

PSA は Kubernetes 本体に組み込まれた Admission Plugin として動作する。

```go
// plugin/pkg/admission/security/podsecurity/admission.go
const PluginName = "PodSecurity"
```

```
Pod / Deployment / StatefulSet 等の作成・更新リクエスト
        |
        v  PodSecurity Admission Plugin
  Pod の spec を取得（Deployment の場合は spec.template.spec）
        |
        v  Namespace のラベルを確認
  enforce/audit/warn のレベルとプロファイルを読み取る
        |
        v  評価（kubernetes/pod-security-admission リポジトリの policy パッケージ）
  各チェック項目を評価（privileged / hostPID / seccomp 等）
        |
  enforce 違反 → 拒否（403）
  warn 違反    → 許可 + Warning ヘッダーを返す
  audit 違反   → 許可 + 監査ログに記録
```

---

## 5. restricted プロファイルの要件

最も厳しい `restricted` を満たすために必要な Pod 設定:

```yaml
spec:
  securityContext:
    runAsNonRoot: true          # root で実行禁止
    seccompProfile:
      type: RuntimeDefault      # seccomp プロファイル必須

  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false  # 特権昇格禁止
      capabilities:
        drop: ["ALL"]           # 全 capabilities を drop
        add: ["NET_BIND_SERVICE"]  # 必要なものだけ追加（省略可）
      runAsNonRoot: true
      runAsUser: 1000           # 非 root UID
```

---

## 6. PSA への移行手順

既存クラスターへの導入は段階的に行う。

```
1. warn モードで導入（既存 Pod への影響なし）
   → kubectl apply で警告が出る Pod を把握する

2. audit モードを追加（監査ログで違反数を把握）
   → 修正が必要な Pod をリストアップ

3. Pod の securityContext を修正（違反を解消）

4. enforce モードに昇格
   → 違反する Pod の新規作成が拒否される

# 段階的移行の例
kubectl label namespace my-app \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

---

## 7. PSP（PodSecurityPolicy）との違い

PSP は 1.25 で削除された旧機能。PSA はその代替。

| | PSP（削除済み）| PSA |
|---|---|---|
| 設定場所 | クラスタースコープのリソース | Namespace のラベル |
| 適用方法 | RBAC で Pod に紐付ける | Namespace 単位で一括適用 |
| 複雑さ | 高（RBAC との組み合わせが難しい）| 低（ラベルを貼るだけ）|
| 柔軟性 | 高（カスタムポリシー可）| 低（3プロファイル固定）|

柔軟なポリシーが必要な場合は OPA/Gatekeeper や Kyverno などの外部ツールを使う。

---

## 8. OPA / Kyverno との比較

PSA で対応できない要件（イメージのレジストリ制限・ラベル強制等）には外部ポリシーエンジンを使う。

| | PSA | OPA/Gatekeeper | Kyverno |
|---|---|---|---|
| 組み込み | あり（Kubernetes 本体）| なし（別途インストール）| なし |
| ポリシー表現 | 3プロファイル固定 | Rego 言語 | YAML |
| イメージ制限 | ✗ | ✓ | ✓ |
| ラベル強制 | ✗ | ✓ | ✓ |
| 変更（Mutating）| ✗ | ✓ | ✓ |

---

## 9. コードリーディングの起点

```
plugin/pkg/admission/security/podsecurity/admission.go
  └── PodSecurity Admission Plugin のエントリポイント

# Pod Security Standards の評価ロジック（外部リポジトリ）
github.com/kubernetes/pod-security-admission/
  └── policy/check*.go ← 各チェック項目の実装
      （privileged / hostPID / seccomp / capabilities 等）

staging/src/k8s.io/api/core/v1/types.go
  └── PodSecurityContext / SecurityContext ← セキュリティ設定の型定義
```
