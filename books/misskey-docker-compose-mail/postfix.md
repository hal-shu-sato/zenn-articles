---
title: "PostfixによるSMTPサーバーの構築"
---

# 目標

Postfixでメールを送信する

# 前提

前提条件は、こちらをご覧ください。
https://zenn.dev/hal_shu_sato/books/misskey-docker-compose-mail/viewer/introduction

このチャプターは、以下の公式ガイドに沿います。
https://ubuntu.com/server/docs/mail-postfix

また、こちらの記事を参考しています。
https://qiita.com/mizuki_takahashi/items/1b33e1f679359827c17d
https://zenn.dev/uchidaryo/books/ubuntu-2204-server-book/viewer/postfix-dovecot

# Postfixのインストール

以下のコマンドでインストールします。

```bash
$ sudo apt install postfix
```

インストール中のプロンプトは、No configurationで大丈夫です。

# Postfixの設定

設定の詳しい説明（日本語訳）はこちらにあります。
https://www.postfix-jp.info/trans-2.3/jhtml/BASIC_CONFIGURATION_README.html

まず、テンプレートをコピーして開きます。

```bash
$ cd /etc/postfix/
$ sudo cp main.cf.proto main.cf
$ sudo vi main.cf
```

`#`から始まる行はコメント行です。

まずホスト名を設定します。

```diff:/etc/postfix/main.cf
-#myhostname = host.domain.tld
+myhostname = subdomain.domain.tld
```

ドメイン名も設定します。

```diff:/etc/postfix/main.cf
-#mydomain = domain.tld
+mydomain = domain.tld
```

送信するメールのドメインを指定します。
私は`noreply@subdomain.domain.tld`を使うので、`$myhostname`を指定します（`$hogehoge`は変数`hogehoge`を参照します）。

```diff:/etc/postfix/main.cf
 #myorigin = /etc/mailname
-#myorigin = $myhostname
+myorigin = $myhostname
 #myorigin = $mydomain
```

メールを受信するネットワークを指定します。
`all`はすべてのネットワークから受信します。
外部から受信しない場合は、2行目や3行目の`#`を外します。

```diff:/etc/postfix/main.cf
-#inet_interfaces = all
+inet_interfaces = all
 #inet_interfaces = $myhostname
 #inet_interfaces = $myhostname, localhost
```

受信するメールの宛先ドメインを指定します。
Postfixは、ここに列挙したドメインに送られたメールに対して、受信の処理をします。

```diff:/etc/postfix/main.cf
 #mydestination = $myhostname, localhost.$mydomain, localhost
-#mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
+mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
 #mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain,
 #       mail.$mydomain, www.$mydomain, ftp.$mydomain
```

メールのリレーを行うリレー元を指定します。
私は、IPv4のループバックアドレスに加えて、IPv6のループバックアドレス、Dockerからの通信を念頭に置いて、ローカルIPアドレスを追加しました。

```diff:/etc/postfix/main.cf
 #mynetworks = 168.100.3.0/28, 127.0.0.0/8
 #mynetworks = $config_directory/mynetworks
 #mynetworks = hash:/etc/postfix/network_table
-mynetworks = 127.0.0.0/8
+mynetworks = 127.0.0.0/8 [::1]/128 172.16.0.0/12 [fe80::]/64
```

メール格納フォーマットを`Maildir`に変更します。

```diff:/etc/postfix/main.cf
 #home_mailbox = Mailbox
-#home_mailbox = Maildir/
+home_mailbox = Maildir/
```

SMTPバナーを変更します。
SMTP接続時に表示するバナーで、メールソフトやバージョンが表示されると攻撃されてしまう可能性があるため、表示しないようにします。
なおRFCの要求で、`$myhostname`から始める必要があります。

```diff:/etc/postfix/main.cf
 #smtpd_banner = $myhostname ESMTP $mail_name
 #smtpd_banner = $myhostname ESMTP $mail_name ($mail_version)
-smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
+#smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
+smtpd_banner = $myhostname ESMTP
```

不必要な設定をコメントアウトして、デフォルト設定を使用します。

```diff:/etc/postfix/main.cf
-sendmail_path =
+#sendmail_path =
```

```diff:/etc/postfix/main.cf
-newaliases_path =
+#newaliases_path =
```

```diff:/etc/postfix/main.cf
-mailq_path =
+#mailq_path =
```

```diff:/etc/postfix/main.cf
-setgid_group =
+#setgid_group =
```

```diff:/etc/postfix/main.cf
-html_directory =
+#html_directory =
```

```diff:/etc/postfix/main.cf
-manpage_directory =
+#manpage_directory =
```

```diff:/etc/postfix/main.cf
-sample_directory =
+#sample_directory =
```

```diff:/etc/postfix/main.cf
-readme_directory =
+#readme_directory =
```

使用するインターネットプロトコル（IP）を指定します。
IPv4とIPv6を両方使う場合は、`all`を指定します。

```diff:/etc/postfix/main.cf
-inet_protocols = ipv4
+inet_protocols = all
```

メール一通のサイズとメールボックスのサイズを制限します。
それぞれ10MBと1GBにしています。

```diff:/etc/postfix/main.cf
+message_size_limit = 10485760
+mailbox_size_limit = 1073741824
```

ログに使用するfacilityを指定します。
デフォルトも`mail`なので、指定する必要はありません。

```diff:/etc/postfix/main.cf
+syslog_facility = mail
```

VRFYコマンド（ユーザーの存在確認）を無効化して、内部ユーザーを隠蔽します。

```diff:/etc/postfix/main.cf
+disable_vrfy_command = yes
```

設定ファイルを保存した後、エイリアスデータベースを更新して、Postfixを再起動します。

```bash
$ sudo newaliases
$ sudo systemctl restart postfix.service
```

# 参考

https://centossrv.com/postfix.shtml
https://qiita.com/rocinante-ein/items/9c31cb0e36fb2d01d343
