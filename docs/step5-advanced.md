# Step 5: 高度な設定・運用管理

CloudFront を通じた API 配信、Bedrock の動作確認、実運用に向けたコスト管理の CLI 手順です。
{ .lead }

## 1. CloudFront を使った API の配信設定

CloudFront 経由で `/api/*` を EC2（Caddy）へ安全にルーティングするため、キャッシュビヘイビアとオリジンリクエストポリシーを設定します。

| 項目 | 設定値 |
| --- | --- |
| パスパターン | `/api/*` |
| オリジン | EC2 のパブリック DNS（HTTP / 80） |
| 許可するメソッド | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE |
| キャッシュポリシー | `CachingDisabled` |
| オリジンリクエストポリシー | `AllViewerExceptHostHeader` |

!!! tip "ポイント"
    - POST を通すには、許可メソッドに `POST` を含める必要があります。
    - `X-Api-Key` をオリジンへ届けるため、オリジンリクエストポリシーでヘッダーを転送します。
    - API のレスポンスは `CachingDisabled` でキャッシュさせません。

CloudFront 経由で API が正しくルーティングされているか確認します。

```bash
curl -s -X POST https://<your-cloudfront-domain>/api/chat \
  -H 'Content-Type: application/json' \
  -H "X-Api-Key: $KEY" \
  -d '{"message":"こんにちは"}'
```

## 2. Amazon Bedrock のモデル検証とテスト

利用可能な推論プロファイルを確認し、CLI から直接モデルを呼び出して動作確認します。

```bash title="利用可能な推論プロファイルの確認"
aws bedrock list-inference-profiles --region ap-southeast-2 \
  --query 'inferenceProfileSummaries[].[inferenceProfileId,status]' --output text
```

```bash title="Converse API でテスト呼び出し"
aws bedrock-runtime converse --region ap-southeast-2 \
  --model-id au.anthropic.claude-haiku-4-5-20251001-v1:0 \
  --messages '[{"role":"user","content":[{"text":"こんにちは"}]}]' \
  --query 'output.message.content[0].text' --output text
```

## 3. AWS Budgets によるコスト管理

予期せぬ課金を防ぐため、CLI から予算（月額 5 USD）とアラート通知を設定します。

### 予算定義ファイル

```json title="budget.json"
{
  "BudgetName": "monthly-5usd",
  "BudgetLimit": { "Amount": "5", "Unit": "USD" },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
```

### 通知設定ファイル

実績が 80% を超えたとき、予測が 100% を超えそうなときの 2 段階でメール通知します。

```json title="notifications.json"
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      { "SubscriptionType": "EMAIL", "Address": "your-email@example.com" }
    ]
  },
  {
    "Notification": {
      "NotificationType": "FORECASTED",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 100,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      { "SubscriptionType": "EMAIL", "Address": "your-email@example.com" }
    ]
  }
]
```

### 予算の作成

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws budgets create-budget --account-id $ACCOUNT_ID \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```
