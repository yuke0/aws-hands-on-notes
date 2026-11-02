# Step 1: 基盤構築・IAM・ネットワーク

本ハンズオンにおけるAWSインフラの土台となる、EC2、セキュリティグループ、およびIAMロールの設定手順です。

## 1. ネットワークとセキュリティグループ
* **リージョン**: シドニー (`ap-southeast-2`)
* **EC2 インスタンス**: メタデータホップリミットの設定やセキュリティグループの整理を実施。

不要なSSHイングレスルールの撤回例：
aws ec2 revoke-security-group-ingress --region ap-southeast-2 \
  --group-id sg-007900fafc85e2eb9 --protocol tcp --port 22 --cidr 103.5.140.152/32

## 2. Amazon DynamoDB テーブルの作成
お問い合わせデータ（contacts）を保存するためのテーブルをオンデマンド（`PAY_PER_REQUEST`）で作成します。

aws dynamodb create-table --region ap-southeast-2 \
  --table-name contacts \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

## 3. EC2への権限付与（IAMロール）
EC2インスタンスからDynamoDBおよびBedrockを安全に操作できるよう、インスタンスプロファイルにカスタムポリシーをアタッチします。

ポリシードキュメント例 (`dyn-policy.json`)：
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["dynamodb:PutItem", "dynamodb:GetItem", "dynamodb:Scan"],
    "Resource": "arn:aws:dynamodb:ap-southeast-2:297361542320:table/contacts"
  }]
}
