---
title: "Docker Composeを利用したMisskeyの構築"
---

# 目標

Docker ComposeでMisskeyサーバーを構築する

# 前提

前提条件は、こちらをご覧ください。
https://zenn.dev/hal_shu_sato/books/misskey-docker-compose-mail/viewer/introduction

このチャプターは、以下の公式ガイドに従った記事に沿います。
https://misskey-hub.net/docs/install/docker.html
https://qiita.com/Soli0222/items/1a8f854706528b63a8e2

また、後に行うCDNの設定ですが、DNSの反映に時間がかかるため、早く公開したい方は先にやっておくことをオススメします。
https://zenn.dev/hal_shu_sato/books/misskey-docker-compose-mail/viewer/cloudflare

# スワップ領域の設定

Misskeyは常時1GB程度のメモリを利用するため、メモリが1GBの場合、メモリ不足になりやすいです。
スワップの設定をしておけば解決できるので、やっておくといいと思います。
https://chatnoirlibre.com/ubuntu-22-04-lts-swap/

## スワップ領域の確認

以下のコマンドを実行します。

```bash
$ sudo swapon --show
```

表示がなければ、スワップ領域はありません。
以下の表示があれば、すでにスワップ領域があります。
この場合はすでに5.3GBの領域が確保されていますので、これ以降の設定は不要でしょう。

```bash
NAME      TYPE SIZE   USED PRIO
/swapfile file 5.3G 174.2M   -2
```

`free`コマンドでメモリについて確認することもできます。
`Swap:`が表示されれば、スワップ領域が確保されています。

```bash
$ free
               total        used        free      shared  buff/cache   available
Mem:         2011080      708080      188444       17792     1114556     1099972
Swap:        5529596      178412     5351184
```

## スワップファイルの作成

まず、ドライブの空き容量を確認します。

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        52G   16G   33G  33% /
```

十分に空き容量があれば、スワップファイルを作成します。
今回は、ルートに4GBの`/swapfile`を作成します。
以下のコマンドで作成します。

```bash
$ sudo fallocate -l 4G /swapfile
```

以下のコマンドで確認します。
`MMM dd HH:mm`は更新日時です。

```bash
$ ls -lh /swapfile
-rw-r--r-- 1 root root 4G MMM dd HH:mm /swapfile
```

安全のため、パーミッションを変更しておきます。

```bash
$ sudo chmod 600 /swapfile
$ ls -lh /swapfile
-rw------- 1 root root 4G MMM dd HH:mm /swapfile
```

## スワップファイルの有効化

スワップ領域を作成して、有効化します。

```bash
$ sudo mkswap /swapfile
$ sudo swapon /swapfile
```

確認して以下のように表示されれば成功です。

```bash
$ sudo swapon --show
NAME      TYPE SIZE USED PRIO
/swapfile file   4G  xxM   -2
```

## 起動時のスワップ領域設定

起動時にスワップ領域を設定するようにします。

```bash
$ sudo vi /etc/fstab
```

して、

```
/swapfile none swap defaults 0 0
```

を追加します。

`fstab`については、こちらが参考になります。
https://qiita.com/kihoair/items/03635447591358210772

# Docker, Docker Composeの導入

まずは、DockerとDocker Composeを導入します。

こちらのガイドに従います。
https://docs.docker.com/engine/install/ubuntu/

## 旧バージョンのアンインストール

新環境で構築する方には、念のためのステップになります。

```bash
$ for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

## レポジトリをセットアップ

Dockerのレポジトリを使えるようにします。

必要なパッケージをインストールします。

```bash
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl gnupg
```

Docker公式のGPG鍵を追加します。

```bash
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

レポジトリをセットアップします

```bash
$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Docker Engineのインストール

Docker Engine, containerd, Docker Composeをインストールします。

```bash
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

`hello-world`イメージを実行して、インストールできたかを確認します。

```bash
$ sudo docker run hello-world
```

# Misskeyの構築

## レポジトリの取得

GitHubからレポジトリをクローンします。

```bash
$ git clone -b master https://github.com/misskey-dev/misskey.git
$ cd misskey
$ git checkout master
```

## 設定

各設定ファイルのテンプレートをコピーします。

```bash
$ cp .config/docker_example.yml .config/default.yml
$ cp .config/docker_example.env .config/docker.env
$ cp ./docker-compose.yml.example ./docker-compose.yml
```

そしていくつか設定を変更します。
まずは、`.config/default.yml`から。

```yaml:.config/default.yml
#━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Misskey configuration
#━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

#   ┌─────┐
#───┘ URL └─────────────────────────────────────────────────────

# Final accessible URL seen by a user.
url: https://subdomain.domain.tld/ # 利用するドメインを設定

# ONCE YOU HAVE STARTED THE INSTANCE, DO NOT CHANGE THE
# URL SETTINGS AFTER THAT!
```

```yaml:.config/default.yml
#   ┌──────────────────────────┐
#───┘ PostgreSQL configuration └────────────────────────────────

db:
  host: db
  port: 5432

  # Database name
  db: misskey

  # Auth
  user: misskey # 自由ですが、私は分かりやすくmisskeyにしました。
  pass: password # お好きなパスワード
```

この設定を、`.config/docker.env`に書き込みます。

```env:.config/docker.env
# db settings
POSTGRES_PASSWORD=password
POSTGRES_USER=misskey
POSTGRES_DB=misskey
```

Dockerイメージをビルドするにはある程度のスペックが必要ですが、スペックが足りない場合はDocker Hubからイメージを取得することもできます。

```yaml:./docker-compose.yml
version: "3"

services:
  web:
    build: . # ビルドする場合（どちらか一方）
    image: misskey/misskey:latest # イメージを使う場合（どちらか一方）
    restart: always
    links:
      - db
      - redis
    # - meilisearch
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "3000:3000" # ポートを変更する場合はこちら
    networks:
      - internal_network
      - external_network
    volumes:
      - ./files:/misskey/files
      - ./.config:/misskey/.config:ro
```

## ビルド、初期化と起動

ビルドする場合はビルドします。

```bash
$ sudo docker compose build
```

データベースを初期化して、起動します。

```bash
$ sudo docker compose run --rm web pnpm run init
$ sudo docker compose up -d
```

これでMisskeyが構築され、`http://localhost:3000`でアクセスできます！

# 管理者アカウントの作成

`http://localhost:3000`にアクセスすると、管理者アカウントの作成画面が表示されます。
構築に成功したら、すぐに作っておきましょう。

# ファイアウォールの有効化

環境によっては、ファイアウォールが設定されておらず、外部から3000番ポートにアクセスできる状態にあることがあります。
とりあえずですが、ファイアウォールを設定しておきましょう。
SSH接続をしている場合は、慎重に設定してください。

ufwを利用して、ファイアウォールを設定します。

まず、現在の状態を確認します。
以下は、Vultrのデフォルト設定と思われます。

```bash
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
```

詳細は以下のコマンドで確認します。

```bash
$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

もしStatusがinactiveになっていれば、ファイアウォールを有効化します。

```bash
$ sudo ufw enable
```

TCP22番（SSH）ポートを許可、または制限（30秒に6回接続した場合ブロック）をかけます。

```bash
$ sudo ufw allow 22/tcp
$ sudo ufw limit 22/tcp
```

そして、デフォルトで通信を拒否します。

```bash
$ sudo ufw default deny
```

ufwサービスを有効化して、再起動後も実行します。

```bash
$ sudo systemctl enable ufw
```
