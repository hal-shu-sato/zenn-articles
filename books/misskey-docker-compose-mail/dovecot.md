---
title: "DovecotによるPOP3, IMAPサーバーの構築"
---

# 目標

Dovecotで受信したメールにアクセスできるようにする
Postfixと連携してSMTP認証をする

# 前提

前提条件は、こちらをご覧ください。
https://zenn.dev/hal_shu_sato/books/misskey-docker-compose-mail/viewer/introduction

このチャプターは、以下の記事に沿います。
https://qiita.com/mizuki_takahashi/items/1b33e1f679359827c17d
https://zenn.dev/uchidaryo/books/ubuntu-2204-server-book/viewer/postfix-dovecot

# Dovecotのインストール

まず、Dovecotをインストールします。

```bash
$ sudo apt install dovecot-core dovecot-imapd dovecot-pop3d
```

# Dovecotの設定

Dovecotの設定は`/etc/dovecot/`以下にあります。

## dovecot.conf

リッスンする送信元を指定します。
IPv4のみで使用可能にする場合は、`*`のみにします。

```diff:/etc/dovecot/dovecot.conf
-#listen = *, ::
+listen = *, ::
```

## conf.d/10-auth.conf

プレーンテキスト認証を許可します。

```diff:/etc/dovecot/conf.d/10-auth.conf
-#disable_plaintext_auth = yes
+disable_plaintext_auth = no
```

`login`での認証を許可します（古いOutlooksやMicrosoft phonesで使われている模様）。

```diff:/etc/dovecot/conf.d/10-auth.conf
- auth_mechanisms = plain
+ auth_mechanisms = plain login
```

## conf.d/10-mail.conf

メール格納フォーマットをMaildir形式にします。

```diff:/etc/dovecot/conf.d/10-mail.conf
-mail_location = mbox:~/mail:INBOX=/var/mail/%u
+mail_location = maildir:~/Maildir
```

## conf.d/10-master.conf

IMAPとPOP3を有効化します。

```diff:/etc/dovecot/conf.d/10-master.conf
 service imap-login {
   inet_listener imap {
-    #port = 143
+    port = 143
   }
```

```diff:/etc/dovecot/conf.d/10-master.conf
 service pop3-login {
   inet_listener pop3 {
-    #port = 110
+    port = 110
   }
```

PostfixでSMTP認証をできるようにします。

```diff:/etc/dovecot/conf.d/10-master.conf
-  #unix_listener /var/spool/postfix/private/auth {
-  #  mode = 0666
-  #}
+  unix_listener /var/spool/postfix/private/auth {
+    mode = 0666
+    user = postfix
+    group = postfix
+  }
---
```

## conf.d/10-ssl.conf

SSLを無効化します。

```diff:/etc/dovecot/conf.d/10-ssl.conf
-ssl = yes
+ssl = no
```

設定を保存した後、Dovecotを再起動します。

```bash
$ sudo systemctl restart dovecot.service
```

# Postfixの設定

SMTP認証の連携のための設定をします。
上から、

- SASL認証の有効化
- SASLの種類をDovecotにする
- SASL認証に使うパスを設定
- SASL認証で匿名認証を拒否する
- SASL認証のローカルドメインを`$myhostname`にする
- 古いSASL認証クライアントを許可するか
- SMTP認証の挙動（上から適用）
  - 自ネットワーク内は許可
  - SASL認証済みは許可
  - 認証されていない宛先のものは拒否

の設定です。

```diff:/etc/postfix/main.cf
+smtpd_sasl_auth_enable = yes
+smtpd_sasl_type = dovecot
+smtpd_sasl_path = private/auth
+smtpd_sasl_security_options = noanonymous
+smtpd_sasl_local_domain = $myhostname
+#broken_sasl_auth_clients = yes
+smtpd_recipient_restrictions =
+    permit_mynetworks
+    permit_sasl_authenticated
+    reject_unauth_destination
```

TCP587番（SUBMISSION）ポートでのSASL認証を有効化します。

```diff:/etc/postfix/master.cf
 # Choose one: enable submission for loopback clients only, or for any client.
 #127.0.0.1:submission inet n -   y       -       -       smtpd
 submission inet n       -       y       -       -       smtpd
 #  -o syslog_name=postfix/submission
 #  -o smtpd_tls_security_level=encrypt
-#  -o smtpd_sasl_auth_enable=yes
+  -o smtpd_sasl_auth_enable=yes
 #  -o smtpd_tls_auth_only=yes
 #  -o smtpd_reject_unlisted_recipient=no
 #  -o smtpd_client_restrictions=$mua_client_restrictions
 #  -o smtpd_helo_restrictions=$mua_helo_restrictions
 #  -o smtpd_sender_restrictions=$mua_sender_restrictions
-#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
+  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
 #  -o milter_macro_daemon_name=ORIGINATING
```

設定を保存した後、Postfixを再起動します。

```bash
$ sudo systemctl restart postfix.service
```

これで、TCP110番（POP3）ポートやTCP143番（IMAP）ポートで受信メールを確認したり、メールを送信するときに簡単な認証をするようになりました。
もちろんファイアウォールで許可すれば、外部からアクセスすることも可能です。

# メールアカウントを追加する

現在の設定ではUbuntuのユーザーがそのままメールアカウントとなっており、認証もそれによって行われています。
メールアカウントのためのユーザーを追加します。

## ユーザー作成時にMaildirを作成するようにする

まず、ユーザーを作成したときにホームディレクトリ内にMaildirを作成するようにします。

```bash
$ sudo mkdir -p /etc/skel/Maildir/{new,cur,tmp}
```

## ユーザーを追加

実際にユーザーを追加します。
ユーザーの追加には、`adduser`（インタラクティブでオススメ）と`useradd`が使えます。

### ログインしてメールを確認できるユーザーを追加する場合

```bash
$ sudo adduser --shell /bin/bash support
# または
$ sudo useradd -m -s /bin/bash support
$ sudo passwd support
```

### ログインできない送信専用のユーザーを追加する場合

```bash
$ sudo adduser --no-create-home --shell /usr/sbin/nologin noreply
# または
$ sudo useradd -s /usr/sbin/nologin noreply
$ sudo passwd noreply
```

# Misskeyの設定

実際にMisskeyできちんとメールを送れるようにしましょう。
今回もTCP587番（SUBMISSION）ポートを使います。

コントロールパネルの設定>メールサーバーを開いて

- メールアドレス：noreply@subdomain.domain.tld
- ホスト：host.docker.internal
- ポート：587
- ユーザー名：noreply
- パスワード：（ユーザーのパスワード）

を入力して保存した後、内部のアドレス（support@subdomain.domain.tldなど受信できるユーザー宛）や外部のアドレスに向けて配信テストをします。
上手くいけばチェックが出て、送信はできると思います。

Postfix側の動作は、ログが`/etc/log/mail.log`に出力されるので、エラーが起きていなければ送信されていると思います。

受信については、内部に向けて送ったものは、送ったユーザーの`~/Maildir/new/`にファイルが生成されていれば送れている可能性が高いです。
`cat`や`vi`などで見られると思います。
外部に向けて送ったものも、ここまでの設定ができていれば届いているはずです。

ここまでできていたら、設定>全般から、管理者のメールアドレスを`support@subdomain.domain.tld`などにしておきましょう。

# TLS暗号化

実際にTLS暗号化をした後、追加します。

# 参考

https://centossrv.com/postfix.shtml
