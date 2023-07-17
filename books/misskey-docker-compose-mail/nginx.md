---
title: "nginxによるWeb公開"
---

# 目標

nginxでMisskeyをWebに公開する

# 前提

前提条件は、こちらをご覧ください。
https://zenn.dev/hal_shu_sato/books/misskey-docker-compose-mail/viewer/introduction

このチャプターは、以下の公式ガイドに従った記事に沿います。
https://misskey-hub.net/docs/admin/nginx.html
https://qiita.com/Soli0222/items/1a8f854706528b63a8e2
https://misskey-hub.net/docs/install/ubuntu-manual.html

# nginxの導入

以下の公式ガイドにしたがって、nginxをインストールします。
https://nginx.org/en/linux_packages.html#Ubuntu

## レポジトリのセットアップ

nginxのレポジトリを使えるようにします。

必要なパッケージをインストールします。

```bash
$ sudo apt update
$ sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring
```

nginxのGPG鍵をインポートします。

```bash
$ curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

正しい鍵がダウンロードされたかを確認します。
もし、出力が以下（公式サイトを確認することをオススメします）と異なれば、ファイルを消去してもう一度やり直します。

```bash
$ gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
pub   rsa2048 2011-08-19 [SC] [expires: 2024-06-14]
      573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
uid                      nginx signing key <signing-key@nginx.com>
```

安定版nginxのレポジトリを使うときは、以下のコマンドを実行します。

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

mainline版nginxのレポジトリを使うときは、以下のコマンドを実行します。

```bash
$ echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
```

nginxレポジトリを優先させるために、nginxレポジトリをピン留めします。

```bash
$ echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
```

## nginxのインストール

nginxをインストールします。

```bash
$ sudo apt update
$ sudo apt install nginx
```

# nginxの設定

`/etc/nginx/conf.d/misskey.conf`に、以下のページのconfigをコピーします。

https://misskey-hub.net/docs/admin/nginx.html

4か所の`example.tld`は、利用するドメインに変更します。
また、CDNを利用するので、`# If it's behind another reverse proxy or CDN, remove the following.`を含む4行を削除します。

設定ファイルを保存したら、設定ファイルのチェックをして、nginxを再起動します。

```bash
$ sudo nginx -t
$ sudo systemctl restart nginx
```

# ファイアウォールの設定

念のためファイアウォールの設定について再掲した後、HTTP/HTTPSポートの設定をします。

## ファイアウォールの有効化（再掲）

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

## ファイアウォール設定の追加

Misskeyへのアクセスで使うTCP80番（HTTP）ポートとTCP443番（HTTPS）ポートを許可します。

```bash
$ sudo ufw allow 80/tcp
$ sudo ufw allow 443/tcp
```
