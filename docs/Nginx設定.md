# Nginx 設定方針

## 基本構成

nginx コンテナ (vmbr0: `192.168.x.100`, vmbr1: `10.10.10.100`) で以下を担当します。

1. **SSL終端** — Let's Encrypt (Certbot) で証明書を取得・自動更新
2. **HTTP → HTTPS リダイレクト** — 全HTTPアクセスをHTTPSに転送
3. **パスベースのルーティング** — URLパスに応じて各サイトに転送
4. **WebSocket プロキシ** — Dashboard, cod-web のリアルタイム通信に対応
5. **セキュリティヘッダー** — XSS, クリックジャッキング等の対策
6. **Gzip圧縮** — 転送量の削減

---

## 設定ファイル構成

```
/etc/nginx/
├── nginx.conf                 # メイン設定
├── sites-available/
│   └── shiratama.conf         # サイト設定（メイン）
├── sites-enabled/
│   └── shiratama.conf → ../sites-available/shiratama.conf
├── snippets/
│   ├── ssl-params.conf        # SSL共通設定
│   ├── security-headers.conf  # セキュリティヘッダー
│   └── proxy-params.conf      # プロキシ共通設定
└── certbot/                   # Let's Encrypt 証明書
```

---

## HTTP → HTTPS リダイレクト

```nginx
server {
    listen 80;
    server_name shiratama.mydns.jp;

    # Certbot の HTTP-01 チャレンジ用
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # 上記以外はすべてHTTPSにリダイレクト
    location / {
        return 301 https://$host$request_uri;
    }
}
```

---

## HTTPS サーバーブロック（メイン設定）

> **重要:** `nginx.conf` の `http {}` ブロック内に以下を追加してください:
> ```nginx
> map $http_upgrade $connection_upgrade {
>     default upgrade;
>     ''      close;
> }
> ```

```nginx
server {
    listen 443 ssl http2;
    server_name shiratama.mydns.jp;

    # SSL証明書 (Let's Encrypt)
    ssl_certificate     /etc/letsencrypt/live/shiratama.mydns.jp/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/shiratama.mydns.jp/privkey.pem;

    # SSL共通設定を読み込み
    include /etc/nginx/snippets/ssl-params.conf;

    # セキュリティヘッダー
    include /etc/nginx/snippets/security-headers.conf;

    # Gzip圧縮
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml text/javascript image/svg+xml;
    gzip_min_length 1000;

    # クライアントボディサイズ制限 (ファイルアップロード用)
    client_max_body_size 50M;

    # ─── ルーティング ───

    # Portfolio (ルート)
    location / {
        include /etc/nginx/snippets/proxy-params.conf;
        proxy_pass http://10.10.10.101:3000/;
    }

    # DropMod
    location /dropmod/ {
        include /etc/nginx/snippets/proxy-params.conf;
        proxy_pass http://10.10.10.101:3001/;
    }

    # Flashcard
    location /flashcard/ {
        include /etc/nginx/snippets/proxy-params.conf;
        proxy_pass http://10.10.10.101:3002/;
    }

    # ytdl
    location /ytdl/ {
        include /etc/nginx/snippets/proxy-params.conf;
        proxy_pass http://10.10.10.101:3003/;
    }

    # Dashboard (WebSocket対応)
    location /admin/ {
        include /etc/nginx/snippets/proxy-params.conf;
        proxy_pass http://10.10.10.101:3004/;

        # Socket.IO (WebSocket) サポート
        location /admin/socket.io/ {
            include /etc/nginx/snippets/websocket-params.conf;
            proxy_pass http://10.10.10.101:3004/socket.io/;
        }
    }

    # PalmIDE
    location /palmide/ {
        include /etc/nginx/snippets/proxy-params.conf;
        proxy_pass http://10.10.10.101:3005/;
    }

    # cod-web (WebSocket対応)
    location /cod/ {
        include /etc/nginx/snippets/websocket-params.conf;
        proxy_pass http://10.10.10.103:8080/;
    }
}
```

---

## スニペット（共通設定）

### proxy-params.conf

```nginx
proxy_http_version 1.1;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# タイムアウト設定
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

### websocket-params.conf

```nginx
# WebSocket接続用（nginx.conf の http ブロックに map を追加）
# map $http_upgrade $connection_upgrade {
#     default upgrade;
#     ''      close;
# }

proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# WebSocket用タイムアウト（24時間）
proxy_read_timeout 86400s;
proxy_send_timeout 86400s;

# リアルタイム通信のためバッファリング無効
proxy_buffering off;
proxy_cache off;
```

### security-headers.conf

```nginx
# XSS保護
add_header X-XSS-Protection "1; mode=block" always;

# クリックジャッキング対策
add_header X-Frame-Options "SAMEORIGIN" always;

# MIMEタイプスニッフィング防止
add_header X-Content-Type-Options "nosniff" always;

# Referrer Policy
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# HSTS (HTTP Strict Transport Security)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### ssl-params.conf

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
```

---

## SSL証明書の取得・更新

### 初回取得

```bash
# Certbot で証明書を取得（HTTP-01 チャレンジ）
certbot certonly --webroot -w /var/www/certbot \
    -d shiratama.mydns.jp \
    --non-interactive --agree-tos \
    --email <メールアドレス>
```

> **注意:** MyDNS.jp はDDNSのため、DNS-01チャレンジは使えません。HTTP-01チャレンジ（webroot方式）を使用します。

### 自動更新

```bash
# cron で1日2回チェック、有効期限が近づいていれば自動更新
0 3,15 * * * certbot renew --quiet --post-hook "nginx -s reload"
```

---

## 各サイトの Next.js 側設定（basePath）

パスベースでルーティングする場合、**各Next.jsアプリ側で `basePath` の設定が必要**です。これを忘れると画像やCSSが読み込めません。

```js
// next.config.js (各サイト共通の考え方)
module.exports = {
  basePath: '/dropmod',  // サイトごとのパス
  output: 'standalone',  // 本番用
}
```

| サイト | `basePath` |
|--------|-----------|
| Portfolio | `''`（ルートなので不要） |
| DropMod | `'/dropmod'` |
| Flashcard | `'/flashcard'` |
| ytdl | `'/ytdl'` |
| Dashboard | `'/admin'` |
| PalmIDE | `'/palmide'` |
| cod-web | `'/cod'` |

---

## メモ

- **Portfolio** は SSG (静的生成) なので、Nginx で静的ファイルを直接配信することも可能。ただし、app コンテナ内にあるため、現在はプロキシ方式。将来的にボリューム共有で直接配信に変更してもよい。
- **ytdl** のダウンロード機能は大きなファイルを扱うため、`client_max_body_size` と `proxy_read_timeout` を大きめに設定する必要があるかも。
- **cod-web** の WebSocket はゲームのリアルタイム通信に使用。遅延を最小化するため、`proxy_buffering off` を追加検討。
