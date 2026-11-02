# Step 3: Docker & Caddy リバースプロキシ

軽量な Web サーバー **Caddy** をリバースプロキシにして、静的サイトと FastAPI コンテナを同居させます。
{ .lead }

## 1. Caddyfile の設定

`/api/*` はバックエンド（`api-a:8000`）へ、それ以外はフロントエンド（`site-a:80`）へ振り分けます。

```caddyfile title="Caddyfile"
:80 {
    handle /api/* {
        reverse_proxy api-a:8000
    }
    handle {
        reverse_proxy site-a:80
    }
}
```

!!! info "HTTPS について"
    Caddy は `:80`（HTTPのみ）で待ち受けています。HTTPS の終端は前段の CloudFront が担当します（Step 5）。

## 2. Docker Compose 構成

環境変数（`API_KEY`）を、イメージに焼き込まずコンテナへ渡す構成です。

```yaml title="docker-compose.yml"
services:
  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

  site-a:
    image: ghcr.io/yuke0/site-a:latest
    restart: unless-stopped

  api-a:
    image: ghcr.io/yuke0/api-a:latest
    restart: unless-stopped
    environment:
      - API_KEY=${API_KEY}

volumes:
  caddy_data:
  caddy_config:
```

## 3. APIキーの受け渡し

`${API_KEY}` は、`docker-compose.yml` と同じ場所の `.env` から読み込まれます。

```bash
printf 'API_KEY=%s\n' "$(openssl rand -hex 32)" > .env
chmod 600 .env
docker compose up -d
```

!!! danger ".env は Git に入れない"
    `.env` は必ず `.gitignore` に追加してください。APIキーをリポジトリや本書に書かないでください。
