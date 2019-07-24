---
title: sshフォワーディングについて
date: 2019-07-14T00:00:00+09:00
tags: ["ssh"]
draft: false

---

同僚と話していて、自身の理解が非常にぼにゃりしていると感じたので、まとめます。

# ローカルフォワード

ローカルの特定ポートを、特定のホスト及びポートにバインドします。例えば、www側のclientよりserver-aに対して接続したい、但し直接はアクセスできない。そういった場合、clientからsshで接続できるようなserver-bがあれば、server-bとsshのセッションを確立し、そこを介してclientとserver-aを繋げるイメージです。

コマンドは下記の通り。

```bash
# clientの2222への通信をserver-b経由でserver-aの22へフォワーディング
ssh user@server-b -L client:2222:server-a:22
```

上記を実行するとプロンプトが占められてしまうので、下記のオプションを加えるのが一般的。

<dl>
  <dt>-N</dt>
  <dd>リモート先で何もコマンドを実行しない</dd>
  <dt>-f</dt>
  <dd>コマンドを実行した後にバックグラウンドにまわす</dd>
  <dt>-g</dt>
  <dd>ローカルの転送ポートにリモートから接続することを許可</dd>
</dl>

なお、`-N`と`-f`は併用します。

```bash
ssh user@server-b -L 8080:server-a:80 -N -f
```

# リモートフォワード

リモートの特定ポートを、特定のホスト及びポートにバインドする。例えば、www側のserver-aよりserver-bに対して接続したいが、直接アクセスはできない。そういった場合、server-bがclientからsshで接続できるとすれば、server-bとsshのセッションを確立し、そこを介してserver-aとserver-bを繋げるイメージです。

コマンドは下記の通り。

```bash
# server-bの2222への通信をclient経由でserver-aの22へフォワーディング
ssh -R 2222:server-a:22 user@server-b
```

こちらもローカルフォワードと同様、実行するとプロンプトが占められてしまうので`-N`と`-f`を加えるのが一般的です。

```bash
# server-bの2222への通信をclient経由でserver-aの22へフォワーディング
ssh -R 2222:server-a:22 user@server-b -N -f
```

なお、リモートフォワードの場合、リモートの特定ポートに別のリモートから接続するには、sshdの設定ファイルのGatewayPortsを'yes'に設定する必要があります。

```text
GatewayPorts yes
```

