# Step 2: Amazon Bedrock との連携

Amazon Bedrock を利用して、FastAPIからAIモデル（Claude）を呼び出す仕組みを構築します。

## 1. 利用可能なモデルの確認
シドニーリージョンで利用可能な推論プロファイル（Inference Profiles）を確認します。

aws bedrock list-inference-profiles --region ap-southeast-2 \
  --query 'inferenceProfileSummaries[].[inferenceProfileId,status]' --output text

選定モデル: `au.anthropic.claude-haiku-4-5-20251001-v1:0`

## 2. FastAPIでのBedrockクライアント実装
`boto3` の `bedrock-runtime` クライアントを使用して、`converse` APIを呼び出すサンプルコードです。

import boto3
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

REGION = "ap-southeast-2"
MODEL_ID = "au.anthropic.claude-haiku-4-5-20251001-v1:0"

app = FastAPI()
bedrock = boto3.client("bedrock-runtime", region_name=REGION)

class Chat(BaseModel):
    message: str = Field(min_length=1, max_length=500)

@app.post("/api/chat")
def chat(c: Chat):
    try:
        res = bedrock.converse(
            modelId=MODEL_ID,
            messages=[{"role": "user", "content": [{"text": c.message}]}],
            inferenceConfig={"maxTokens": 300},
        )
    except Exception as e:
        raise HTTPException(status_code=502, detail=str(e))
    return {"reply": res["output"]["message"]["content"][0]["text"]}
