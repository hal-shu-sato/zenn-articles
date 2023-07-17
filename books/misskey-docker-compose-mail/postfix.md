---
title: "PostfixによるSMTPサーバーの構築"
---

# 目標

Postfixでメールを送信する
認証されたメールを送れるようにする

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

# SPF・DKIM・DMARCの設定

それぞれ送信元ドメインを認証する設定です。
迷惑メールとして判定されにくくなります。
すべてDNSのTXTレコードに指定します。

## SPF (Sender Policy Framework)

SPFは、送信元ドメインをIPアドレスによって認証します。
DNSで`subdomain.domain.tld`のTXTレコードに設定します。
以下は、私が設定した例です。

```
v=spf1 ip4:xxx.xxx.xxx.xxx ip6:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx mx -all
```

各タグの簡単な説明を示します。

| タグ    | 説明と値                                                                                                                                                                         |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| v       | SPFバージョン。必須。必ず最初に`v=spf1`を指定する。                                                                                                                              |
| ip4     | IPv4アドレス。`ip4:xxx.xxx.xxx.xxx`や`ip4:xxx.xxx.0.0/16`のように指定する。                                                                                                      |
| ip6     | IPv6アドレス。`ip6:xxxx::xxxx`や`ip6:xxxx::/16`のように指定する。                                                                                                                |
| a       | Aレコード。`a:domain.tld`のように指定する。`a`のみの場合、SPFレコードを設定したドメインのAレコードを参照する。                                                                   |
| mx      | MXレコード。`mx:domain.tld`のように指定する。`mx`のみの場合や指定がない場合は、SPFレコードを設定したドメインのMXレコードを参照する。                                             |
| include | サードパーティーの送信元ドメイン。`include:mail.another.tld`のように指定する。                                                                                                   |
| all     | すべてのメール。指定を推奨。必ず最後に置き、以降は無視される。`~all`を指定すると、受信されるが不審なメールとしてマークされる。`-all`を指定すると、受信を拒否される可能性がある。 |

さらに詳しい説明はこちらをご覧ください。
https://support.google.com/a/answer/10683907

## DKIM (DomainKeys Identified Mail)

DKIMは、電子署名によってドメインを認証します。

まずはOpenDKIMをインストールします。

```bash
$ sudo apt install opendkim
```

そしてDKIMのためのキーを生成します。
生成先のディレクトリを指定するには`-D /path/to/dir/`、セレクタ名（鍵の判別用）を変える場合は`-s selector_name`を追加で指定します。

```bash
$ cd
$ sudo opendkim-genkey -d subdomain.domain.tld -b 2048
```

すると実行ディレクトリ（または指定したディレクトリ）に、default.txtとdefault.private（セレクタ名による）が生成されます。

default.privateの権限を変更して、`/etc/dkimkeys/`にコピーします。

```bash
$ chmod 600 default.private
$ sudo chown opendkim:opendkim default.private
$ sudo mv default.private /etc/dkimkeys/
```

default.txtの内容を`default._domainkey.subdomain.domain.tld`（セレクタ名による）のTXTレコードに設定します。
ファイル内で`"(改行)          "`のように改行されていますが、これはすべて取り除いて一行にします。

```:default._domainkey.subdomain.domain.tld TXTレコード
v=DKIM1; h=sha256; k=rsa; p=(公開鍵)
```

### Author Domain Signing Practices (ADSP)

また受信時の挙動を定めるため、Author Domain Signing Practices (ADSP)を指定します。
`_adsp._domainkey.subdomain.domain.tld`のTXTレコードに設定します。
私は以下を指定しました。

```:_adsp._domainkey.subdomain.domain.tld TXTレコード
dkim=unknown
```

値による挙動の説明は以下の通りです。

| 値          | 説明                                                                                                                                                               |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| all         | このドメインから送信されるメールは、すべてメール作成者署名が与えられる                                                                                             |
| unknown     | このドメインから送信されるメールのいくつか、またはすべてに、メール作成者署名が得られる                                                                             |
| discardable | このドメインから送信されるメールは、すべてメール作成者署名が与えられる。そして、もしメール作成者署名が得られない場合は、受信者はそのメールを破棄することが望まれる |

[DKIM (Domainkeys Identified Mail)](https://salt.iajapan.org/wpmu/anti_spam/admin/tech/explanation/dkim/)（迷惑メール対策委員会）より引用

### OpenDKIMの設定

OpenDKIMの設定をします。
元の設定を残し忘れたので、変更後のみ載せます。

`Mode`を`sv`（sign/verify; 送信時に署名/受信時に認証）にします。

```:/etc/opendkim.conf
Canonicalization        relaxed/simple
Mode                    sv
SubDomains              no
OversignHeaders         From
```

先ほど生成した鍵のセレクタと秘密鍵ファイルを指定します。

```:/etc/opendkim.conf
#Domain                 example.com
Selector                default
KeyFile         /etc/dkimkeys/default.private
```

OpenDKIMを実行するためのソケットファイルの場所をPostfixが使いやすい場所にします。

```diff:/etc/opendkim.conf
-Socket local:/run/opendkim/opendkim.sock
+#Socket local:/run/opendkim/opendkim.sock
 #Socket inet:8891@localhost
 #Socket inet:8891
-#Socket local:/var/spool/postfix/opendkim/opendkim.sock
+Socket local:/var/spool/postfix/opendkim/opendkim.sock
```

ヘッダーのソフトウェアバージョンを載せないように変更します。

```diff:/etc/opendkim.conf
+SoftwareHeader no
```

これらを保存した後、ソケットファイルのためにディレクトリを作成します。

```bash
$ cd /var/spool/postfix/
$ sudo mkdir opendkim
$ sudo chown opendkim:opendkim opendkim
$ sudo chmod 750 opendkim
```

### Postfixの設定

OpenDKIMの処理を挟むようにします。

```diff:/etc/postfix/main.cf
+smtpd_milters = local:opendkim/opendkim.sock
+non_smtpd_milters = $smtpd_milters
+milter_default_action = accept
```

また、ユーザー`postfix`がOpenDKIMのSocketを実行できるように、`opendkim`グループに追加します。

```bash
$ sudo adduser postfix opendkim
```

## DMARC (Domain-based Message Authentication, Reporting and Conformance)

DMARCはSPFやDKIMの認証に失敗したメールをどうするかを指定します。
`_dmarc.subdomain.domain.tld`のTXTレコードに以下の設定をします。

```
v=DMARC1; p=quarantine; adkim=s; aspf=s;
```

pの設定は以下のような挙動をします。

| 値         | SPFやDKIMに失敗した場合に |
| ---------- | ------------------------- |
| none       | 何もしない（通過する）    |
| quarantine | 隔離する                  |
| reject     | ブロックする              |

`adkim`と`aspf`は、DKIMやSPFの厳格さを定義します。
ドメイン名について、s(strict)は完全一致、r(relaxed)は部分一致となります。

詳しい設定はこちらをご覧ください。
https://support.google.com/a/answer/2466563
https://support.google.com/a/answer/10032169
https://www.cloudflare.com/ja-jp/learning/dns/dns-records/dns-dmarc-record/

### OpenDMARC

PostfixがDMARCの認証をするようにします。

まず、OpenDMARCをインストールします。

```bash
$ sudo apt install opendmarc
```

`/etc/opendmarc.conf`を開き、以下の設定を変更します。
こちらも元の設定を残し忘れたので、変更後のみ載せます。

`authserv-id`を指定します。

```diff:/etc/opendmarc.conf
 # AuthservID name
+AuthservID OpenDMARC
```

いったん認証に失敗したときに、rejectしないようにします。

```diff:/etc/opendmarc.conf
-#RejectFailures false
+RejectFailures false
```

OpenDMARCを実行するためのソケットファイルの場所をPostfixが使いやすい場所にします。

```diff:/etc/opendmarc.conf
 # Socket local:/run/opendmarc/opendmarc.sock
+Socket local:/var/spool/postfix/opendmarc/opendmarc.sock
```

信頼できる`authserv-id`として、ホスト名を記載します。

```diff:/etc/opendmarc.conf
 # TrustedAuthservIDs HOSTNAME
+TrustedAuthservIDs subdomain.domain.tld
```

実行ユーザを`opendmarc`にします。

```diff:/etc/opendmarc.conf
-# UserID opendmarc
+UserID opendmarc
```

認証を無視するホストを`/etc/opendmarc/ignore.hosts`に指定し、認証済みのクライアントも無視するようにする。
また、ヘッダーがRFC5322のセクション3.6に適合するかどうかを確認する。

```diff:/etc/opendmarc.conf
+IgnoreHosts /etc/opendmarc/ignore.hosts
+IgnoreAuthenticatedClients true
+RequiredHeaders true
```

`ignore.hosts`の設定をします。
`opendmarc`ディレクトリを作成して、`ignore.hosts`を作成します。

```bash
$ cd /etc/
$ sudo mkdir opendmarc
$ sudo vi ignore.hosts
```

今回はローカルホストやループバックアドレス、プライベートアドレスを指定しておきました。

```:/etc/opendmarc/ignore.hosts
localhost
127.0.0.1
::1
172.16.0.0/12
fe80::/64
```

これらを保存した後、ソケットファイルのためにディレクトリを作成します。

```bash
$ cd /var/spool/postfix/
$ sudo mkdir opendmarc
$ sudo chown opendmarc:opendmarc opendmarc
$ sudo chmod 750 opendmarc
```

### Postfixの設定

OpenDMARCの処理も挟むようにします。

```diff:/etc/postfix/main.cf
-smtpd_milters = local:opendkim/opendkim.sock
+smtpd_milters = local:opendkim/opendkim.sock,local:opendmarc/opendmarc.sock
```

また、ユーザー`postfix`がOpenDMARCのSocketを実行できるように、`opendmarc`グループに追加します。

```bash
$ sudo adduser postfix opendmarc
```

# 参考

https://centossrv.com/postfix.shtml
https://qiita.com/rocinante-ein/items/9c31cb0e36fb2d01d343
https://monmon.jp/301/introspates-spf-dkim-dmarc-to-avoid-spam-maetization/
https://kmuto.hatenablog.com/entry/2022/09/19/105809
https://salt.iajapan.org/wpmu/anti_spam/admin/tech/explanation/dkim/
https://tm.root-n.com/tec:opendkim:setup
https://www.rem-system.com/dkim-postfix04/
https://rin-ka.net/centos-postfix-dkim/
https://www.hs3.org/Ubuntu20_Postfix_03
