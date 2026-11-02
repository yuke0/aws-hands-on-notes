# Step 4: セキュリティ・運用管理

本番運用を想定した、API保護・流量制限（レートリミット）・セキュリティ対策の設定です。
{ .lead }

## 1. APIキー認証（HMAC比較）

ヘッダーの `X-Api-Key` を検証し、不正なアクセスを防ぎます。`hmac.compare_digest` は一定時間で比較するため、タイミング攻撃を避けられます。

```python title="auth.py"
import hmac
import os

from fastapi import Header, HTTPException

API_KEY = os.environ.get("API_KEY", "")


def verify_api_key(x_api_key: str = Header(default="")):
    if not API_KEY or not hmac.compare_digest(x_api_key, API_KEY):
        raise HTTPException(status_code=401, detail="unauthorized")
```

エンドポイントには `dependencies` で適用します。

```python title="main.py（抜粋）"
from fastapi import Depends

from auth import verify_api_key


@app.post("/api/chat", dependencies=[Depends(verify_api_key)])
def chat(c: Chat):
    ...
```

!!! note "API_KEY が空のとき"
    `API_KEY` が未設定だと全リクエストを拒否する設計です。設定漏れで認証が無効になる事故を防ぎます。

## 2. レートリミットの導入

不正アクセスや過剰なリクエストから API を守ります。Bedrock は呼び出すたびに課金されるため、コスト保護にもなります。

=== "アプリ側 (slowapi)"

    ```bash
    pip install slowapi
    ```

    ```python title="main.py（抜粋）"
    from fastapi import Depends, FastAPI, Request
    from slowapi import Limiter, _rate_limit_exceeded_handler
    from slowapi.errors import RateLimitExceeded


    def client_ip(request: Request) -> str:
        xff = request.headers.get("x-forwarded-for", "")
        return xff.split(",")[0].strip() or request.client.host


    limiter = Limiter(key_func=client_ip)
    app = FastAPI()
    app.state.limiter = limiter
    app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)


    @app.post("/api/chat", dependencies=[Depends(verify_api_key)])
    @limiter.limit("10/minute")
    def chat(request: Request, c: Chat):
        ...
    ```

    !!! warning "注意"
        `request: Request` 引数が必須です。また `X-Forwarded-For` は CloudFront 経由のときだけ信頼できます。EC2 を直接公開していると偽装されます。

=== "CloudFront側 (AWS WAF)"

    CloudFront に AWS WAF のレートベースルールを関連付ける方法です。アプリに手を入れずに済みますが、CloudFront 用の WAF は `us-east-1` に作成します。Web ACL とルールに月額料金がかかるため、月額 5 USD の予算とのバランスを見て選んでください。
