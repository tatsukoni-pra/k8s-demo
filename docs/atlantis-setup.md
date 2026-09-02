# Atlantis on EKS セットアップ手順

Atlantis を EKS 上で稼働させるために実施した作業の記録。

## 前提条件

以下は既に構築済みであること。

- EKS クラスタ（`eks-test-cluster`）
- ALB + HTTPS Listener（ACM 証明書設定済み）
- Target Group（`tg-atlantis`、port 4141）
- Route53 レコード（`atlantis.awsometatsukoni.com` → ALB）
- OIDC Provider（IRSA 用）
- AWS Load Balancer Controller
- External Secrets Operator + ClusterSecretStore
- ArgoCD

## 実施した作業

### 1. EBS CSI Driver アドオンの有効化

Atlantis は StatefulSet で PVC（10Gi, gp2）を使用するため、EBS CSI Driver が必要。

**対象リポジトリ**: `eks-terraform`

**ファイル**: `tmp_eks_resources.tf`

```hcl
resource "aws_eks_addon" "aws_ebs_csi_driver" {
  cluster_name             = aws_eks_cluster.eks_test_cluster.name
  addon_name               = "aws-ebs-csi-driver"
  service_account_role_arn = aws_iam_role.aws_ebs_csi_driver.arn
}
```

> IAM Role（`aws-ebs-csi-driver-role`）は既に作成済みだったため、アドオンのコメントアウトを解除して `terraform apply`。

---

### 2. Atlantis 用 IRSA（IAM Role for ServiceAccount）の作成

Atlantis Pod が `terraform plan/apply` で AWS API を叩くために、IRSA を設定。

**対象リポジトリ**: `eks-terraform`

**ファイル**: `iam_dependon_eks.tf`

#### IAM Policy（`AtlantisPolicy`）

| 権限 | 用途 |
|------|------|
| `sns:*` | Terraform で管理する SNS リソースの操作 |
| `s3:GetObject/PutObject/DeleteObject/ListBucket` on `atlantis-demo-202609` | Terraform の S3 バックエンド（tfstate） |
| `sts:GetCallerIdentity` | `terraform plan` 時の認証確認 |

#### IAM Role（`AtlantisRole`）

- IRSA 用の trust policy で `atlantis` namespace の `sa-atlantis` のみが引き受け可能
- `AtlantisPolicy` をアタッチ

#### ServiceAccount へのアノテーション追加

**対象リポジトリ**: `k8s-demo`

**ファイル**: `infra/atlantis/service_account.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-atlantis
  namespace: atlantis
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::083636136646:role/AtlantisRole
```

---

### 3. GitHub App の作成と認証設定

PAT ではなく GitHub App 方式で認証を設定。

#### 3-1. GitHub App の作成

**設定ページ**: `https://github.com/settings/apps/new`

| 項目 | 値 |
|------|-----|
| App name | atlantis-tatsukoni-pra |
| Homepage URL | `https://atlantis.awsometatsukoni.com` |
| Webhook URL | `https://atlantis.awsometatsukoni.com/events` |
| Webhook secret | 任意の文字列（`openssl rand -hex 32` 等で生成） |
| Webhook Active | 有効 |

**Repository permissions**:

| Permission | Access |
|-----------|--------|
| Administration | Read-only |
| Checks | Read & Write |
| Commit statuses | Read & Write |
| Contents | Read & Write |
| Issues | Read & Write |
| Metadata | Read-only |
| Pull requests | Read & Write |
| Webhooks | Read & Write |
| Members | Read-only |
| Actions | Read-only |

**Subscribe to events**: GitHub App 方式では自動設定されるため、手動設定不要。

**Identifying and authorizing users**: すべて空欄/無効のまま。

#### 3-2. GitHub App を対象リポジトリにインストール

App 設定ページ → 「Install App」→ 対象 Organization/リポジトリを選択。

#### 3-3. 控えた情報

| 値 | 用途 |
|----|------|
| App ID | values.yaml の `githubApp.id` |
| Installation ID | values.yaml の `githubApp.installationId` |
| Slug | values.yaml の `githubApp.slug`（App 設定ページ URL の末尾） |
| Private Key（PEM） | Secrets Manager に格納 |
| Webhook Secret | Secrets Manager に格納 |

---

### 4. Secrets Manager にシークレットを格納

**対象リポジトリ**: `eks-terraform`

**ファイル**: `secretsmanager.tf`

```hcl
resource "aws_secretsmanager_secret" "atlantis" {
  name = "eks-test-cluster/atlantis/secret"
}
```

`terraform apply` 後、AWS CLI でシークレット値を格納：

```bash
aws secretsmanager put-secret-value \
  --secret-id eks-test-cluster/atlantis/secret \
  --secret-string "$(jq -n \
    --arg key "$(cat /path/to/private-key.pem)" \
    --arg webhook "YOUR_WEBHOOK_SECRET" \
    '{"github-app-privateKey": $key, "github-webhook-secret": $webhook}'
  )"
```

> **注意**: PEM ファイルの改行を正しく保持するため、`jq -n --arg` を使用すること。手動で JSON を書くと改行が壊れる。

| Secrets Manager キー名 | 内容 |
|------------------------|------|
| `github-app-privateKey` | GitHub App の Private Key（PEM） |
| `github-webhook-secret` | Webhook Secret |

---

### 5. ExternalSecret の作成

Secrets Manager → K8s Secret として Pod に注入。

**対象リポジトリ**: `k8s-demo`

**ファイル**: `infra/atlantis/external-secret.yaml`

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: atlantis-vcs-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: cluster-secret-store
    kind: ClusterSecretStore
  target:
    creationPolicy: Owner
  data:
    - secretKey: github_secret
      remoteRef:
        key: eks-test-cluster/atlantis/secret
        property: github-webhook-secret
    - secretKey: key.pem
      remoteRef:
        key: eks-test-cluster/atlantis/secret
        property: github-app-privateKey
```

Helm chart の `vcsSecretName` を指定すると、以下が自動で行われる：
- `github_secret` → 環境変数 `ATLANTIS_GH_WEBHOOK_SECRET` に注入
- `key.pem` → `/var/github-app/key.pem` にボリュームマウント

---

### 6. values.yaml の設定

**対象リポジトリ**: `k8s-demo`

**ファイル**: `infra/atlantis/values.yaml`

GitHub App 認証への変更ポイント：

```yaml
githubApp:
  id: "4789286"          # 文字列としてクォートすること（浮動小数点になるのを防ぐ）
  installationId: "158140158"  # 同上
  slug: atlantis-tatsukoni-pra  # App 設定ページ URL の末尾と一致させること

vcsSecretName: "atlantis-vcs-secret"  # ExternalSecret が作成する Secret 名
```

---

## トラブルシューティング

### `id` / `installationId` が浮動小数点になる

```
ATLANTIS_GH_APP_ID: 4.789286e+06
```

**原因**: YAML が数値として解釈する。
**対策**: 文字列としてクォートする（`"4789286"`）。

### `could not parse private key: invalid key`

**原因**: Secrets Manager に格納した PEM の改行が壊れている。
**対策**: `jq -n --arg` を使って PEM ファイルから直接格納する。

### `GET https://api.github.com/apps/<slug>: 404 Not Found`

**原因**: `githubApp.slug` が実際の GitHub App のスラッグと一致していない。
**対策**: `https://github.com/settings/apps/<slug>` の URL 末尾を確認して一致させる。
