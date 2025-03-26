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
### 다른 포맷 예시

| 포맷 문자열                  | 결과 예시                 |
| ----------------------- | --------------------- |
| `"2006-01-02"`          | `2025-03-26`          |
| `"20060102"`            | `20250326`            |
| `"2006-01-02_15-04-05"` | `2025-03-26_13-45-02` |
| `"15:04:05"`            | `13:45:02`            |
| `"02 Jan 2006"`         | `26 Mar 2025`         |
