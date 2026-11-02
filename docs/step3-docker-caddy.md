# Step 3: Docker & Caddy リバースプロキシ

軽量なWebサーバーである **Caddy** をリバースプロキシとして使い、静的サイトとFastAPIコンテナを同居させます。

## 1. Caddyfile の設定
`/api/*` へのリクエストはバックエンド（`api-a:8000`）へ、それ以外はフロントエンド（`site-a:80`）へルーティングします。

:80 {
    handle /api/* {
        reverse_proxy api-a:8000
    }
    handle {
        reverse_proxy site-a:80
    }
}

## 2. Docker Compose 構成 (`docker-compose.yml`)
環境変数（`API_KEY`）を安全にコンテナへ渡すように設計しています。

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
