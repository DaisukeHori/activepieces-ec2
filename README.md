# DockerとActivepiecesのセットアップガイド

このガイドは、EC2インスタンスのUbuntu24.04上でDockerとActivepiecesをセットアップし、再起動後に自動的に起動するように設定するためのステップバイステップの手順を示しています。

## 前提条件

- EC2インスタンスでUbuntu24.04を開始していてClIを操作できる状態であること
- EC2上でSSH,80,443,5678のTCPポートを0.0.0.0/0（任意の場所）からアクセスできるようにセキュリティグループを設定しておくこと
- ドメインを取得すること
- EC2のパブリック IPv4 DNS　（ec2-xx-xx-xxx-xxx.ap-northeast-1.compute.amazonaws.com#こんな感じ）をサブドメイン（一般的にはactivepieces.xxxx.com）にCNAMEで紐づけておくこと

余談：

- t3nano:V2Core0.5GB:$5/月
- t3micro:V2Core1GB:$10/月
- t3small:V2Core2GB:$20/月
- t3midium:V2Core4GB:$40/月

## ステップ1: 古いDocker関連パッケージの削除

以下のコマンドを実行して、古いDocker関連パッケージを削除します。

```sh
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove -y $pkg; done
```

## ステップ2: Dockerリポジトリの設定とインストール

1. 必要なパッケージのインストール

```sh
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

2. DockerのGPGキーを追加

```sh
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

3. Dockerリポジトリの追加

```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

4. Dockerと関連パッケージのインストール

最新バージョンのインストールコマンドを入れます。
```sh
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

5. Dockerの動作確認

```sh
sudo docker run hello-world
```

###ここまでの参考

https://docs.docker.com/engine/install/ubuntu/


## ステップ3: ユーザーをDockerグループに追加

```sh
sudo usermod -aG docker ${USER}
```

この変更を反映するために、再ログインするか次のコマンドを実行します：

```sh
su - ${USER}　#なんか知らんがこれ動かないし、動かなくても動いた。
#あるいは↓
newgrp docker
```

## ステップ4: GitとActivepiecesのインストール

1. Gitのインストール

```sh
sudo apt-get install -y git
```

2. Activepiecesリポジトリのクローン

```sh
git clone https://github.com/activepieces/activepieces.git
cd activepieces
```

3. デプロイスクリプトの実行

```sh
sh tools/deploy.sh
```

## ステップ5: SSL証明書の取得とNginxの設定

1. Nginxのインストール

```sh
sudo apt-get install -y nginx
```

2. Certbotのインストール

```sh
sudo apt-get install -y certbot python3-certbot-nginx
```

3. SSL証明書の取得

以下のコマンドを実行して、ドメイン用のSSL証明書を取得します。`activepieces.revol-one.com`をあなたのドメイン名に置き換えてください。

```sh
sudo certbot --nginx -d activepieces.revol-one.com
```

4. Nginxの設定ファイルを編集

```sh
sudo tee /etc/nginx/sites-available/default <<EOF
server {
    listen 80;
    listen [::]:80;

    server_name activepieces.revol-one.com;

    return 301 https://\$server_name\$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name activepieces.revol-one.com;

    ssl_certificate /etc/letsencrypt/live/activepieces.revol-one.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/activepieces.revol-one.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host \$host;
        proxy_cache_bypass \$http_upgrade;
    }

    location ~ /\.ht {
        deny all;
    }
}

server {
    if (\$host = activepieces.revol-one.com) {
        return 301 https://\$host\$request_uri;
    }

    listen 80;
    listen [::]:80;

    server_name activepieces.revol-one.com;
    return 404;
}
EOF

# Nginxを再起動して設定を反映
sudo systemctl restart nginx
```

## ステップ6: ActivepiecesのDockerコンテナの実行

1. ActivepiecesのDockerコンテナを実行

```sh
docker run -d --restart unless-stopped -p 8080:80 -v ~/.activepieces:/root/.activepieces -e AP_QUEUE_MODE=MEMORY -e AP_DB_TYPE=SQLITE3 -e AP_FRONTEND_URL="http://localhost:8080" --name activepieces_container activepieces/activepieces:latest
```

## ステップ7: サーバ再起動後の自動起動設定

1. Docker Composeのサービスファイルを作成

```sh
sudo tee /etc/systemd/system/docker-compose-activepieces.service <<EOF
[Unit]
Description=Docker Compose Activepieces Service
After=network.target docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/home/ubuntu/activepieces
ExecStart=/usr/bin/docker compose -p activepieces up -d
ExecStop=/usr/bin/docker compose -p activepieces down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF
```

2. サービスを有効化して起動

```sh
sudo systemctl daemon-reload
sudo systemctl enable docker-compose-activepieces
sudo systemctl start docker-compose-activepieces
```

## ステップ8: セキュリティグループの設定

EC2インスタンスのセキュリティグループを設定し、ポート443を許可します。

1. EC2ダッシュボードから対象のインスタンスを選択
2. 「セキュリティ」タブを選択し、関連付けられているセキュリティグループをクリック
3. 「インバウンドルールの編集」を選択
4. 新しいルールを追加し、以下のように設定
   - タイプ: HTTPS
   - プロトコル: TCP
   - ポート範囲: 443
   - ソース: 任意の場所（0.0.0.0/0）または必要に応じたIP範囲

## ステップ9: Activepiecesの設定変更（必要に応じて）

`AP_FRONTEND_URL`をHTTPSで設定します：

```sh
# Docker Compose環境変数ファイルを編集
sudo nano ~/activepieces/.env

# 以下の行を追加または修正
AP_FRONTEND_URL="https://activepieces.revol-one.com"
```

## 完了

上記の設定を行った後、Webブラウザから `https://activepieces.revol-one.com` にアクセスしてActivepiecesのインターフェースにSSLでアクセスできるはずです。
