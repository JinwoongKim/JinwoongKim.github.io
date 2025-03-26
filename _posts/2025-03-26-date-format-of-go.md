---
title: "\bGo언어에서 데이터 포멧 정의"
categories: study
tags:
  - Go
published: false
---
https://stackoverflow.com/questions/20530327/origin-of-mon-jan-2-150405-mst-2006-in-golang


```go
...
	"gopkg.in/natefinch/lumberjack.v2"
...

// 로그 파일 설정 (init 함수에서 한 번만 실행)
func init() {
	today := time.Now().Format("2006-01-02")
	log.SetOutput(&lumberjack.Logger{
		Filename:   "./logs/" + today + ".log",
		MaxSize:    10, // MB
		MaxBackups: 7,
		MaxAge:     7, // days
		Compress:   true,
	})
	log.SetFlags(log.LstdFlags | log.Lshortfile)
}
```

MaxSize 단일 파일
format