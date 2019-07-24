---
title: Goで動的なJSONをパースする
date: 2019-03-08T00:00:00+09:00
tags: ["golang", "json"]
draft: false

---

GoでJSONパースする場合は構造体を定義し、そこにデータを入れ込むとのが一般的だと思います。

```go
type Response struct {
  Status int      `json:"status"`
}
```

パース対象のJSONはこんな感じで、

```json
{
  "status": 200,
}
```

デコードする処理はこう。
```go
var result Response{}
json.NewDecoder(body).Decode(&result)
```

ただ、実際にはプロパティは同一にも関わらず文字列やブールと、型が異なるようなケースもあります。

```json
{
  "status": "200",
}
```

```json
{
  "status": {
    "code": 200,
    "message": "ok",
  },
}
```

こういう場合、そもそも構造体の定義時にプロパティをinterface{}で定義し、パースした後に判定し入れてあげられればいい感じに扱えます。

```go
type Response struct {
  Status interface{}  `json:"status"`
}

type Status struct {
  Code  int `json:"code"`
  Message string  `json:"message"`
}

switch value := result.Status.(type) {
  case string:
    result.SyncStatus = value
  case Status:
    result.SyncStatus = value
}
```
