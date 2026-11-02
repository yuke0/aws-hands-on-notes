# Step 4: セキュリティ・運用管理

本番運用を想定したAPI保護、流量制限（レートリミット）、およびセキュリティ対策の設定です。

## 1. APIキー認証（HMAC比較）
ヘッダーに含まれる `X-Api-Key` を安全に検証し、不正なアクセスを防ぎます。

import hmac
import os
from fastapi import Header, HTTPException

API_KEY = os.environ.get("API_KEY", "")

def verify_api_key(x_api_key: str = Header(default="")):
    if not API_KEY or not hmac.compare_digest(x_api_key, API_KEY):
        raise HTTPException(status_code=401, detail="unauthorized")

## 2. レートリミットの導入
不正アクセスや過剰なリクエストからAPIを守るための流量制限設定の適用。
