# Step 2: Amazon Bedrock との連携

Amazon Bedrock を使い、FastAPI から AI モデル（Claude）を呼び出す仕組みを構築します。
{ .lead }

## 1. 利用可能なモデルの確認

シドニーリージョンで利用できる推論プロファイル（Inference Profiles）を確認します。

```bash
aws bedrock list-inference-profiles --region ap-southeast-2 \
  --query 'inferenceProfileSummaries[].[inferenceProfileId,status]' --output text
```

!!! success "選定モデル"
    `au.anthropic.claude-haiku-4-5-20251001-v1:0`

## 2. FastAPI での Bedrock クライアント実装

`boto3` の `bedrock-runtime` クライアントで `converse` API を呼び出すサンプルです。

```python title="main.py"
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
```

!!! tip "コスト対策"
    入力は `max_length=500`、出力は `maxTokens=300` に制限しています。想定外の高額請求を防ぐ最初の防波堤です。Step 5 の AWS Budgets と併用します。

!!! warning "本番運用での注意"
    `detail=str(e)` は内部のエラーメッセージをそのままクライアントに返します。本番ではログに出力し、クライアントには固定メッセージ（例: `"upstream error"`）を返すのが安全です。

## 3. ローカルでの動作確認

```bash
uvicorn main:app --port 8000
```

```bash
curl -s -X POST http://localhost:8000/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"こんにちは"}'
```
