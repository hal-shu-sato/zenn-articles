---
title: "はじめに"
---

この本は、MisskeyをDocker Composeで構築する一連の流れを文書化したものです。
基本的に以下のガイドに従います。
https://misskey-hub.net/docs/install/docker.html

# 目標

この本では、

- Docker Compose上でMisskeyサーバーを構築する
- Web上に公開する
- Postfix + Dovecotによって、認証メールなどを送受信できるようにする

ことが目標です。

# 前提

この本では以下の前提で進めていきます。

- Ubuntu 22.04 LTS x64
- 独自ドメインを持っていること
- 以下のパッケージを利用すること
  - [Docker, Docker Compose](https://www.docker.com/)
  - [nginx](https://nginx.org/)
  - [Certbot](https://certbot.eff.org/)
  - [Postfix](https://www.postfix.org/)
  - [Dovecot](https://www.dovecot.org/)

## 自環境

また、私の環境を書いておきます。
この環境下での話であることをご了承ください。

- VPS: Vultr
- CPU: 1 vCPU
- RAM: 2GB (2048.00 MB)
- ストレージ: 55 GB SSD
- 帯域幅: 2.00 TB

# 書き方

この本では

- できるだけ公式の方法に沿うこと
- 利用するパッケージのインストールから説明すること
- 設定の意味を詳細に解説すること

を心がけて執筆します。
もし、分かりにくい解説があれば、お気軽に[Twitter](https://twitter.com/ato_lash_dev)や[GitHub](https://github.com/hal-shu-sato/zenn-articles)にお知らせください。
