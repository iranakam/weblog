---
title: Nginxでのロードバランシング
date: 2019-02-05T00:00:00+09:00
tags: ["nginx"]
draft: false
---

Nginxでロードバランシングを試したのでメモを残します。

# 基本的な設定

Nginxは80番ポートで待ち受け、サーバーグループで指定するサーバーにリクエストを振り分けます。  

```nginx
upstream app {
    server unix:/var/www/app1/tmp/sockets/puma.sock;
    server unix:/var/www/app2/tmp/sockets/puma.sock;
}

server {
    listen 80;
    location / {
        proxy_pass http://app;
    }
}
```

# ラウンドロビン

上記の設定ではロードバランシングのメソッドを指定していません。  
その場合、ラウンドロビンによる切り替えとなり、交互にリクエストが振り分けされます。  

下記は、app1とapp2のログです。  
交互に振り分けされていることがわかります。

```text
127.0.0.1 - - [04/Feb/2019:02:34:24 +0900] "GET /hello HTTP/1.0" 200 189 0.0036
127.0.0.1 - - [04/Feb/2019:02:34:26 +0900] "GET /hello HTTP/1.0" 200 189 0.0020
127.0.0.1 - - [04/Feb/2019:02:34:27 +0900] "GET /hello HTTP/1.0" 200 189 0.0040
127.0.0.1 - - [04/Feb/2019:02:34:28 +0900] "GET /hello HTTP/1.0" 200 189 0.0024
```

```text
127.0.0.1 - - [04/Feb/2019:02:34:22 +0900] "GET /hello HTTP/1.0" 200 189 0.0020
127.0.0.1 - - [04/Feb/2019:02:34:25 +0900] "GET /hello HTTP/1.0" 200 189 0.0025
127.0.0.1 - - [04/Feb/2019:02:34:26 +0900] "GET /hello HTTP/1.0" 200 189 0.0026
127.0.0.1 - - [04/Feb/2019:02:34:27 +0900] "GET /hello HTTP/1.0" 200 189 0.0021
```

# リーストコネクション

サーバーグループへ`least_conn`ディレクティブを追加します。  
それにより各アプリケーションサーバーのコネクション数によってリクエストが振り分けされます。

```nginx
upstream app {
    least_conn;
    server unix:/var/www/app1/tmp/sockets/puma.sock;
    server unix:/var/www/app2/tmp/sockets/puma.sock;
}
```

スリープ時間をパラメータで受取、その分だけスリープしてセッション保持して検証をします。

下記は、上から順にNginx、app1、app2のログです。  
その時点でのセッション数に応じて振り分けされます。

```text
127.0.0.1 - - [04/Feb/2019:02:45:43 +0900] "GET /hello/1 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:44 +0900] "GET /hello/2 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:46 +0900] "GET /hello/3 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:48 +0900] "GET /hello/5 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:48 +0900] "GET /hello/4 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:50 +0900] "GET /hello/6 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:51 +0900] "GET /hello/8 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:51 +0900] "GET /hello/7 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:51 +0900] "GET /hello/9 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:02:45:52 +0900] "GET /hello/10 HTTP/1.1" 200 189 "-" "curl/7.58.0"
```

```text
127.0.0.1 - - [04/Feb/2019:02:45:48 +0900] "GET /hello/5 HTTP/1.0" 200 189 5.0069
127.0.0.1 - - [04/Feb/2019:02:45:48 +0900] "GET /hello/4 HTTP/1.0" 200 189 4.0010
127.0.0.1 - - [04/Feb/2019:02:45:51 +0900] "GET /hello/9 HTTP/1.0" 200 189 9.0014
127.0.0.1 - - [04/Feb/2019:02:45:52 +0900] "GET /hello/10 HTTP/1.0" 200 189 10.0008
```

```text
127.0.0.1 - - [04/Feb/2019:02:45:43 +0900] "GET /hello/1 HTTP/1.0" 200 189 1.0130
127.0.0.1 - - [04/Feb/2019:02:45:44 +0900] "GET /hello/2 HTTP/1.0" 200 189 2.0070
127.0.0.1 - - [04/Feb/2019:02:45:46 +0900] "GET /hello/3 HTTP/1.0" 200 189 3.0083
127.0.0.1 - - [04/Feb/2019:02:45:50 +0900] "GET /hello/6 HTTP/1.0" 200 189 6.0016
127.0.0.1 - - [04/Feb/2019:02:45:51 +0900] "GET /hello/8 HTTP/1.0" 200 189 8.0014
127.0.0.1 - - [04/Feb/2019:02:45:51 +0900] "GET /hello/7 HTTP/1.0" 200 189 7.0014
```

# IPハッシュ

サーバーグループへ`ip_hash`を追加することで、クライアントのIPにより振り分け先を固定し、セッションを持続することができます。  
検証が面倒なので省略します。

# 加重ロードバランシング

サーバーグループ内のサーバーへ`weight`を設定することで、振り分けの重み付けが行われます。  

```nginx
upstream app {
    server unix:/var/www/app1/tmp/sockets/puma.sock weight=1;
    server unix:/var/www/app2/tmp/sockets/puma.sock;
}
```

以下のログは、上から順にNginx、app1、app2のログです。  

## app1にweight=1を設定

weight=1では特に違いは確認できません。

```text
127.0.0.1 - - [04/Feb/2019:03:11:21 +0900] "GET /hello/1 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:22 +0900] "GET /hello/2 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:24 +0900] "GET /hello/3 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:26 +0900] "GET /hello/5 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:26 +0900] "GET /hello/4 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:28 +0900] "GET /hello/6 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:29 +0900] "GET /hello/8 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:29 +0900] "GET /hello/7 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:29 +0900] "GET /hello/9 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:11:29 +0900] "GET /hello/10 HTTP/1.1" 200 189 "-" "curl/7.58.0"
```

```text
127.0.0.1 - - [04/Feb/2019:03:11:22 +0900] "GET /hello/2 HTTP/1.0" 200 189 2.0127
127.0.0.1 - - [04/Feb/2019:03:11:24 +0900] "GET /hello/3 HTTP/1.0" 200 189 3.0017
127.0.0.1 - - [04/Feb/2019:03:11:26 +0900] "GET /hello/5 HTTP/1.0" 200 189 5.0021
127.0.0.1 - - [04/Feb/2019:03:11:26 +0900] "GET /hello/4 HTTP/1.0" 200 189 4.0033
127.0.0.1 - - [04/Feb/2019:03:11:29 +0900] "GET /hello/10 HTTP/1.0" 200 189 10.0028
```

```text
127.0.0.1 - - [04/Feb/2019:03:11:21 +0900] "GET /hello/1 HTTP/1.0" 200 189 1.0122
127.0.0.1 - - [04/Feb/2019:03:11:28 +0900] "GET /hello/6 HTTP/1.0" 200 189 6.0089
127.0.0.1 - - [04/Feb/2019:03:11:29 +0900] "GET /hello/8 HTTP/1.0" 200 189 8.0006
127.0.0.1 - - [04/Feb/2019:03:11:29 +0900] "GET /hello/7 HTTP/1.0" 200 189 7.0008
127.0.0.1 - - [04/Feb/2019:03:11:29 +0900] "GET /hello/9 HTTP/1.0" 200 189 9.0013
```

## app1にweight=3を設定

weight=3ではその傾向がより顕著に出ていることが確認できます。  

```text
127.0.0.1 - - [04/Feb/2019:03:14:30 +0900] "GET /hello/1 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:31 +0900] "GET /hello/2 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:33 +0900] "GET /hello/3 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:35 +0900] "GET /hello/5 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:35 +0900] "GET /hello/4 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:37 +0900] "GET /hello/6 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:38 +0900] "GET /hello/8 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:38 +0900] "GET /hello/7 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:38 +0900] "GET /hello/9 HTTP/1.1" 200 189 "-" "curl/7.58.0"
127.0.0.1 - - [04/Feb/2019:03:14:39 +0900] "GET /hello/10 HTTP/1.1" 200 189 "-" "curl/7.58.0"
```

```text
127.0.0.1 - - [04/Feb/2019:03:14:30 +0900] "GET /hello/1 HTTP/1.0" 200 189 1.0096
127.0.0.1 - - [04/Feb/2019:03:14:35 +0900] "GET /hello/5 HTTP/1.0" 200 189 5.0082
127.0.0.1 - - [04/Feb/2019:03:14:35 +0900] "GET /hello/4 HTTP/1.0" 200 189 4.0013
127.0.0.1 - - [04/Feb/2019:03:14:37 +0900] "GET /hello/6 HTTP/1.0" 200 189 6.0008
127.0.0.1 - - [04/Feb/2019:03:14:38 +0900] "GET /hello/8 HTTP/1.0" 200 189 8.0014
127.0.0.1 - - [04/Feb/2019:03:14:38 +0900] "GET /hello/7 HTTP/1.0" 200 189 7.0008
127.0.0.1 - - [04/Feb/2019:03:14:38 +0900] "GET /hello/9 HTTP/1.0" 200 189 9.0016
127.0.0.1 - - [04/Feb/2019:03:14:39 +0900] "GET /hello/10 HTTP/1.0" 200 189 10.0022
```

```text
127.0.0.1 - - [04/Feb/2019:03:14:31 +0900] "GET /hello/2 HTTP/1.0" 200 189 2.0053
127.0.0.1 - - [04/Feb/2019:03:14:33 +0900] "GET /hello/3 HTTP/1.0" 200 189 3.0017
```
