---
title: Nginxでの静的ファイルの配信
date: 2018-12-17T00:00:00+09:00
tags: ["nginx"]
draft: false
---

自分用にメモ。

Nginxで静的コンテンツを配信する場合に

```nginx
location /images/ {
  root /path/to/app/public/images;
}
```

とするのではなく、

```nginx
location /images/ {
  root /path/to/app/public;
}
```

こうすると。

[Nginxの静的ファイル配信でハマった - shoya.io](https://shoya.io/ja/posts/nginx-root/)  

> rootはstaticディレクトリのrootを指すのではなく、アプリケーションのroot。/static/はURLとして生きているので/static/を含むパスでファイルへ届くように書く必要がある。

[NGINX Docs | Serving Static Content](https://docs.nginx.com/nginx/admin-guide/web-server/serving-static-content/#root)  

> The root directive specifies the root directory that will be used to search for a file. To obtain the path of a requested file, NGINX appends the request URI to the path specified by the root directive. The directive can be placed on any level within the http {}, server {}, or location {} contexts. In the example below, the root directive is defined for a virtual server. It applies to all location {} blocks where the root directive is not included to explicitly redefine the root:

