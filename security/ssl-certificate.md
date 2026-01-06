# SSL Certificate
## クラウドにおけるオレオレ証明書を発行するための認証局の作り方
基本的には IaaS でシンプルに構成するのが一番楽。Windows Server を立てて AD CS 機能を追加し、証明書を発行する

### IIS の場合の証明書インストール
- 発行した証明書の `.pfx` を GUI ベースでサーバーにインストール
- IIS から Binding を作成して証明書を認識させる
### Apache/Nginx の場合の証明書インストール
- 発行した証明書の `.crt` と `.key` を特定のパスに保存
- Apache/Nginx の conf からその置き場所を指定して、認識させる

```
<VirtualHost *:443>
    ServerName internal.example.local

    SSLEngine on

    SSLCertificateFile      /etc/ssl/certs/server.crt
    SSLCertificateKeyFile   /etc/ssl/private/server.key
    SSLCertificateChainFile /etc/ssl/ca/chain.crt

    # Apache 2.4.8+ の場合（chain不要）
    # SSLCertificateFile に fullchain を指定することも可
</VirtualHost>
```

## そもそも .pfx とか .pem とかの違いって何なん？
とても大事なところ来ました 👍
**拡張子の違い＝中身の役割の違い**です。名前より「**何が入っているか**」で理解すると一気に整理できます。

---

# まず結論（超要約）

| 拡張子             | 正体              | 秘密鍵 | 主な用途           |
| --------------- | --------------- | --- | -------------- |
| `.key`          | 秘密鍵             | ✅   | Apache / Nginx |
| `.crt`          | 証明書             | ❌   | サーバ証明書         |
| `.pem`          | **入れ物（テキスト形式）** | △   | Linux 系全般      |
| `.pfx` (`.p12`) | **入れ物（バイナリ）**   | ✅   | Windows / IIS  |

👉 **拡張子はラベル**。中身が重要。

---

# それぞれをちゃんと説明

## 🔐 `.key`（秘密鍵）

* **秘密鍵そのもの**
* サーバが TLS で「自分だ」と証明するために使う
* 絶対に漏らしたらダメ

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

✔ Apache / Nginx はこれを直接参照
❌ IIS は使わない（ストアに格納）

---

## 📜 `.crt`（証明書）

* 公開鍵＋署名＋有効期限など
* **秘密鍵は含まれない**
* クライアントに見せる側

```text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

* サーバ証明書
* 中間 CA
* Root CA
  👉 全部 `.crt` になりうる

---

## 📄 `.pem`（PEM 形式）

ここが一番混乱ポイント。

### PEM = **形式（エンコーディング）**

* Base64 + ヘッダ
* 中身は何でもあり

```text
-----BEGIN CERTIFICATE-----
-----BEGIN PRIVATE KEY-----
-----BEGIN RSA PRIVATE KEY-----
```

つまり：

* `.pem` = 証明書のこともある
* `.pem` = 秘密鍵のこともある
* `.pem` = 両方入ってることもある 😅

👉 **拡張子だけ見ても意味がない**

---

## 📦 `.pfx` / `.p12`（PKCS#12）

* **証明書＋秘密鍵＋中間CA**をまとめたもの
* パスワードで保護
* Windows / IIS の大好物

```text
server.pfx
 ├─ server cert
 ├─ private key
 └─ intermediate CA
```

* certlm.msc にインポート可能
* IIS / App Gateway / Azure で多用

---

# よくある組み合わせ（実務）

## Apache / Nginx

```text
server.crt   ← 証明書
server.key   ← 秘密鍵
chain.crt    ← 中間CA
```

## IIS / Windows

```text
server.pfx   ← ぜんぶ入り
```

---

# 変換の対応関係（超重要）

### PEM → PFX（Apache → IIS）

```bash
openssl pkcs12 -export \
  -in server.crt \
  -inkey server.key \
  -certfile chain.crt \
  -out server.pfx
```

### PFX → PEM（IIS → Apache）

```bash
openssl pkcs12 -in server.pfx -out all.pem
```

---

# 一言で整理すると

* `.key` = **秘密**
* `.crt` = **身分証**
* `.pem` = **文字起こし形式**
* `.pfx` = **全部入りカバン**

---

## 実務での黄金ルール

* 🔴 **秘密鍵は最小権限・最小配置**
* 🔴 `.pem` は中身を見て判断
* 🔴 IIS は `.pfx`、Apache は `.crt + .key`
