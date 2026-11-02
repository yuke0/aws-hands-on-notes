# Step 5: 高度な設定・運用管理（CloudFront・Bedrock・Budgets）

インフラの構築に加えて、CloudFrontを通じたAPIの配信、Bedrockの検証、および実運用に向けたセキュリティ・コスト管理のCLI手順です。

## 1. CloudFront を使ったAPIの配信設定

CloudFront経由で `/api/*` のリクエストをEC2（Caddy）へ安全にルーティングするため、キャッシュビヘイビアとオリジンリクエストポリシーを設定します。

実際にCloudFront経由でAPIが正しくルーティングされているか確認します。

curl -s -X POST https://<your-cloudfront-domain>/api/chat \
  -H 'Content-Type: application/json' \
  -H "X-Api-Key: $KEY" \
  -d '{"message":"こんにちは"}'

---

## 2. Amazon Bedrock のモデル検証とテスト

シドニーリージョン（`ap-southeast-2`）で利用可能な推論プロファイルを確認し、CLIから直接AIモデルを呼び出して動作確認を行います。

利用可能な推論プロファイルの確認：
aws bedrock list-inference-profiles --region ap-southeast-2 \
  --query 'inferenceProfileSummaries[].[inferenceProfileId,status]' --output text

Bedrock Runtime (Converse API) を使ったテスト呼び出し：
aws bedrock-runtime converse --region ap-southeast-2 \
  --model-id au.anthropic.claude-haiku-4-5-20251001-v1:0 \
  --messages '[{"role":"user","content":[{"text":"こんにちは"}]}]' \
  --query 'output.message.content[0].text' --output text

---

## 3. AWS Budgets によるコスト管理（予算アラート）

予期せぬ料金発生を防ぐため、AWS CLIから直接予算（月額5USD）とアラート通知を設定します。

### 予算定義ファイル (`budget.json`)
{
  "BudgetName": "monthly-5usd",
  "BudgetLimit": { "Amount": "5", "Unit": "USD" },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}

### 通知設定ファイル (`notifications.json`)
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "your-email@gmail.com" }]
  },
  {
    "Notification": {
      "NotificationType": "FORECASTED",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 100,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "your-email@gmail.com" }]
  }
]

### 予算の作成実行
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws budgets create-budget --account-id $ACCOUNT_ID \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
