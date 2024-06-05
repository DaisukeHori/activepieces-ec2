# DockerとActivepiecesのセットアップガイド

このガイドは、EC2インスタンスのUbuntu24.04上でDockerとActivepiecesをセットアップし、再起動後に自動的に起動するように設定するためのステップバイステップの手順を示しています。

## 前提条件

- EC2インスタンスでUbuntu24.04を開始していてClIを操作できる状態であること
- EC2上でSSH,80,443,8080のTCPポートを0.0.0.0/0（任意の場所）からアクセスできるようにセキュリティグループを設定しておくこと
- ドメインを取得すること
- EC2のパブリック IPv4 DNS　（ec2-xx-xx-xxx-xxx.ap-northeast-1.compute.amazonaws.com#こんな感じ）をサブドメイン（一般的にはactivepieces.xxxx.com）にCNAMEで紐づけておくこと

### 余談：

- t3nano:V2Core0.5GB:$5/月
- t3micro:V2Core1GB:$10/月
- t3small:V2Core2GB:$20/月
- t3midium:V2Core4GB:$40/月

### インストール後のサーバの状態

インストール完了後は下記のようになります。のでメモリは1GBでギリギリ。2GBである程度は。4GBあれば余裕がありそうです。
```sh
sudo apt-get install sysstat
mpstat
top - 06:11:46 up 38 min,  1 user,  load average: 0.00, 0.00, 0.09
Tasks: 134 total,   1 running, 133 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.2 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.2 st 
MiB Mem :   3836.9 total,    173.9 free,    908.7 used,   3076.5 buff/cache     
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   2928.3 avail Mem 
```
SSDは下記のようになります。10GBでは多分足りません。15GB以上ないときつそうです。
```sh
ubuntu@ip-172-31-17-43:~/activepieces$ df -h
Filesystem       Size  Used Avail Use% Mounted on
/dev/root         29G  7.2G   21G  26% /
tmpfs            1.9G     0  1.9G   0% /dev/shm
tmpfs            768M  1.2M  767M   1% /run
tmpfs            5.0M     0  5.0M   0% /run/lock
efivarfs         128K  3.6K  120K   3% /sys/firmware/efi/efivars
/dev/nvme0n1p16  881M   76M  744M  10% /boot
/dev/nvme0n1p15  105M  6.1M   99M   6% /boot/efi
tmpfs            384M   12K  384M   1% /run/user/1000
```

### まとめてシステム全体のステータスを確認する

`glances` コマンドを使用すると、システム全体のステータスを一目で確認できます。
```
sudo apt-get install glances
glances
```
`glances`はCPU、メモリ、ディスクの使用状況、ネットワークの状態などをリアルタイムで表示します。


```
ip-172-31-17-43 (Ubuntu 24.04 64bit / Linux 6.8.0-1008-aws)                                                                                                                               Uptime: 0:47:19

Intel(R) Xeon(R) Platinum 8259CL CPU @ 2.50GHz                           CPU -     1.9%  idle    98.5%  ctx_sw    242         MEM -   25.0%  active   1.05G         SWAP -   0.0%         LOAD -  2core
CPU  [|                                                          1.9%]   user      1.0%  irq      0.0%  inter     168         total   3.75G  inacti   2.05G         total       0         1 min    0.06
MEM  [||||||||||||||                                            25.0%]   system    0.2%  nice     0.0%  sw_int     80         used     960M  buffer    217M         used        0         5 min    0.14
SWAP [                                                           0.0%]   iowait    0.0%  steal    0.2%                        free    2.81G  cached   2.73G         free        0         15 min   0.12

NETWORK                  Rx/s   Tx/s   TASKS 135 (261 thr), 1 run, 82 slp, 52 oth Threads sorted automatically by CPU consumption
br-bac391a8e680            0b     0b
ens5                      9Kb   90Kb   CPU%   MEM%  VIRT  RES       PID USER          TIME+ THR  NI S  R/s W/s  Command ('e' to pin | 'k' to kill)
lo                       384b   384b   >2.9   1.3   204M  48.1M    6488 ubuntu         0:08 1     0 R    0 0    python3 /usr/bin/glances
veth44355c7                0b     0b    0.0   5.9   21.2G 225M     4071 root           0:36 12    0 S    ? ?    node --enable-source-maps dist/packages/server/api/main.js
vethac6295a                0b     0b    0.0   3.4   563M  131M     4679 root           0:01 6     0 S    ? ?    fwupd
vethdc13e85                0b     0b    0.0   2.4   2.30G 92.6M    2716 root           1:47 14    0 S    ? ?    dockerd -H fd:// --containerd=/run/containerd/containerd.sock
                                        0.0   1.2   1.72G 45.9M    2579 root           0:01 8     0 S    ? ?    containerd
DefaultGateway                   6ms    0.0   1.1   199M  43.3M    6318 root           0:00 1     0 S    ? ?    python3 /usr/bin/glances -s -B 127.0.0.1
                                        0.0   0.8   1.90G 30.7M     668 root           0:01 11    0 S    ? ?    snapd
DISK I/O                  R/s    W/s    0.0   0.7   282M  26.5M     181 root           0:00 7     0 S    ? ?    multipathd -d -s
nvme0n1                     0      0    0.0   0.7   208M  26.2M    3873 999            0:00 1     0 S    ? ?    postgres
nvme0n1p1                   0      0    0.0   0.6   107M  22.4M     684 root           0:00 2     0 S    ? ?    python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
nvme0n1p14                  0      0    0.0   0.5   31.7M 20.1M     660 root           0:00 1     0 S    ? ?    python3 /usr/bin/networkd-dispatcher --run-startup-triggers
nvme0n1p15                  0      0    0.0   0.5   364M  20.0M    6446 root           0:00 4     0 S    ? ?    packagekitd
nvme0n1p16                  0      0    0.0   0.5   1.75G 17.9M     996 root           0:00 9     0 S    ? ?    amazon-ssm-agent
                                        0.0   0.4   208M  16.5M    4109 999            0:00 1     0 S    ? ?    postgres: checkpointer 
FILE SYS                 Used  Total    0.0   0.4   49.4M 13.9M     139 root           0:00 1     0 S    ? ?    systemd-journald
/ (nvme0n1p1)           7.79G  28.0G    0.0   0.4   22.2M 13.5M       1 root           0:06 1     0 S    ? ?    init
                                        0.0   0.3   458M  13.4M     670 root           0:00 6     0 S    ? ?    udisksd
SENSORS                                 0.0   0.3   21.1M 12.6M     335 systemd-r      0:00 1     0 S    ? ?    systemd-resolved
Composite                      -273C    0.0   0.3   383M  12.2M     785 root           0:00 4     0 S    ? ?    ModemManager
                                        0.0   0.3   1.18G 11.6M    4001 root           0:00 11    0 S    ? ?    containerd-shim-runc-v2 -namespace moby -id 855a848c8d462885806a68583a357f9cc12b9f9d96bc6
                                        0.0   0.3   1.18G 11.6M    3841 root           0:00 11    0 S    ? ?    containerd-shim-runc-v2 -namespace moby -id 9f0ee4ae11a097c41dc4be20ae7e0bb3411bfd1b0c95a
                                        0.0   0.3   1.18G 11.5M    3822 root           0:00 11    0 S    ? ?    containerd-shim-runc-v2 -namespace moby -id 2424340cb7471d4f3a042ac9beef3efe30c50b88adaea
                                        0.0   0.3   19.6M 11.0M    1421 ubuntu         0:00 1     0 S    ? ?    systemd --user
                                        0.0   0.3   21.9M 9.62M     533 systemd-n      0:00 1     0 S    ? ?    systemd-networkd
                                        0.0   0.2   208M  9.46M    4111 999            0:00 1     0 S    ? ?    postgres: walwriter 
                                        0.0   0.2   24.0M 9.27M    4627 www-data       0:00 1     0 S    ? ?    nginx: worker process
                                        0.0   0.2   23.9M 9.27M    4628 www-data       0:00 1     0 S    ? ?    nginx: worker process
                                        0.0   0.2   374M  9.25M     662 polkitd        0:00 4     0 S    ? ?    polkitd --no-debug
                                        0.0   0.2   55.2M 8.88M    3880 999            0:03 5     0 S    ? ?    redis-server *:6379
                                        0.0   0.2   17.6M 8.50M     635 root           0:00 1    -1 S    ? ?    systemd-logind
                                        0.0   0.2   209M  8.34M    4112 999            0:00 1     0 S    ? ?    postgres: autovacuum launcher 
                                        0.0   0.2   26.0M 8.16M     195 root           0:01 1     0 S    ? ?    systemd-udevd
                                        0.0   0.2   14.4M 8.11M    1074 root           0:00 1     0 S    ? ?    sshd: ubuntu [priv]
                                        0.0   0.2   11.7M 7.88M    1073 root           0:00 1     0 S    ? ?    sshd: /usr/sbin/sshd -D -o AuthorizedKeysCommand /usr/share/ec2-instance-connect/eic_run_
                                        0.0   0.2   209M  7.21M    4114 999            0:00 1     0 S    ? ?    postgres: logical replication launcher 
                                        0.0   0.2   14.6M 6.91M    1526 ubuntu         0:00 1     0 S    ? ?    sshd: ubuntu@pts/0
                                        0.0   0.2   208M  6.71M    4110 999            0:00 1     0 S    ? ?    postgres: background writer 
                                        0.0   0.2   8.39M 5.88M    4070 root           0:00 1     0 S    ? ?    nginx: master process nginx -g daemon off;
                                        0.0   0.2   217M  5.88M     750 syslog         0:00 4     0 S    ? ?    rsyslogd -n -iNONE
                                        0.0   0.1   9.70M 5.50M     641 messagebu      0:00 1     0 S    ? ?    @dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --sysl
                                        0.0   0.1   66.4M 5.46M    4113 999            0:00 1     0 S    ? ?    postgres: stats collector 
                                        0.0   0.1   8.85M 5.12M    3493 ubuntu         0:00 1     0 S    0 0    bash
```



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

以下のコマンドを実行して、ドメイン用のSSL証明書を取得します。`activepiecestest.revol-one.com`をあなたのドメイン名に置き換えてください。

```sh
sudo certbot --nginx -d activepiecestest.revol-one.com
```

途中は下記のような入力が求められます。
```sh
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): xxxx@xxx.co.jp ＃管理用？メールアドレス。確認連絡も来ないようなのでなんでもいい。               
```sh
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.4-April-3-2024.pdf. You must agree in
order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y　#だまって"Y"

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y　#だまって"Y"
Account registered.
Requesting a certificate for activepiecestest.revol-one.com
```



4. Nginxの設定ファイルを編集

```sh
sudo tee /etc/nginx/sites-available/default <<EOF
server {
    listen 80;
    listen [::]:80;

    server_name activepiecestest.revol-one.com;

    return 301 https://\$server_name\$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name activepiecestest.revol-one.com;

    ssl_certificate /etc/letsencrypt/live/activepiecestest.revol-one.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/activepiecestest.revol-one.com/privkey.pem;
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
    if (\$host = activepiecestest.revol-one.com) {
        return 301 https://\$host\$request_uri;
    }

    listen 80;
    listen [::]:80;

    server_name activepiecestest.revol-one.com;
    return 404;
}
EOF

# Nginxを再起動して設定を反映
sudo systemctl restart nginx
```


## ステップ6: サーバ再起動後の自動起動設定

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
sudo systemctl start docker-compose-activepieces #ここめちゃ時間かかるのでしばらくほっとく。コマンドラインで完了してもアクセスすると表示できない場合があるがほっとく
```

## ステップ7: セキュリティグループの設定

EC2インスタンスのセキュリティグループを設定し、ポート443を許可します。

1. EC2ダッシュボードから対象のインスタンスを選択
2. 「セキュリティ」タブを選択し、関連付けられているセキュリティグループをクリック
3. 「インバウンドルールの編集」を選択
4. 新しいルールを追加し、以下のように設定
   - タイプ: HTTPS
   - プロトコル: TCP
   - ポート範囲: 443
   - ソース: 任意の場所（0.0.0.0/0）または必要に応じたIP範囲

## ステップ8: Activepiecesの設定変更（必要に応じて）

`AP_FRONTEND_URL`をHTTPSで設定します：

```sh
# Docker Compose環境変数ファイルを編集
sudo nano ~/activepieces/.env

# 以下の行を追加または修正
AP_FRONTEND_URL="https://activepiecestest.revol-one.com"
```

## 完了

上記の設定を行った後、Webブラウザから `https://activepieces.revol-one.com` にアクセスしてActivepiecesのインターフェースにSSLでアクセスできるはずです。
