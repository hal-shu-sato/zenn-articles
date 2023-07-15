---
title: "Docker Composeでコンテナ内からホストに接続しようとして詰まった話"
emoji: "🐋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dockercompose"]
published: true
published_at: 2023-07-14 22:00
---

# 何が起きたの？

Docker Composeを利用してMisskeyのサーバーを構築していたところ、ホスト側に建てたPostfix+Dovecotのサーバーとの通信につまずきました。

# エラー内容

Misskeyのコントロールパネル内のメールサーバーで、

- ホスト：127.0.0.1（ループバックアドレス）
- ポート：25（SMTP）や587（SUBMISSION）

を設定して保存後、配信テストをしたところエラーが……。

```json
Endpoint: admin/send-email
Info: {"e":{"message":"connect ECONNREFUSED 127.0.0.1:25","code":"Error","id":"06f9227e-f0c0-4d71-8ab0-6f71a43f47e1"}}
Date: yyyy-MM-ddTHH:mm:ss.fffZ
```

```json
Endpoint: admin/send-email
Info: {"e":{"message":"connect ECONNREFUSED 127.0.0.1:587","code":"Error","id":"06f9227e-f0c0-4d71-8ab0-6f71a43f47e1"}}
Date: yyyy-MM-ddTHH:mm:ss.fffZ
```

# 理由

どうしてだあああと調べていたところ、そもそもDockerコンテナは仮想環境らしく、別のネットワークを使っているため、ループバックしてもコンテナ自体に返ってしまうらしい。
そこで調べたところ、この記事が。

https://qiita.com/ijufumi/items/badde64d530e6bade382

ホストのIPを直接指定すればいいらしいけど、それは環境を移行したりIPが変わったりすると意味ないよなあ……。
しかも、`host.docker.internal`がホストIPを指すのは、Docker Desktop限定の機能らしい……。

# 解決

もう少し調べたところ、今度はこの記事が。

https://gotohayato.com/content/561/

なるほど、`host-gateway`が環境変数的にホストIPになってくれるんだ。
ということで、Docker Composeの設定ファイル内の`services.web`に以下の設定を追加。

```yaml:docker-compose.yml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

そして、

- ホスト：host.docker.internal
- ポート：25（SMTP）や587（SUBMISSION）

としたところ、無事につながり、メールが送信されました。
