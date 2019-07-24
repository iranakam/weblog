---
title: Goで構造体からマップへ変換する
date: 2019-05-31T00:00:00+09:00
tags: ["golang"]
draft: false

---

```go
import "encoding/json"

func structToMap(structArgs Args) map[string]interface{} {
	mapArgs := map[string]interface{}{}
	encoded, _ := json.Marshal(structArgs)
	json.Unmarshal(encoded, &mapArgs)
	return mapArgs
}
```
