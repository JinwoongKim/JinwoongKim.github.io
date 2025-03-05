---
title: ChatGPT를 이용해 Go언어 배우기
categories: study
tags:
  - ChatGPT
  - Go
published: true
---

최근 Go언어로 API 서버를 구축해보게 되었다.

하지만, 오랜만인 Go언어와 익숙하지 않은 API 서버 개발로 난항을 겪고 있었다.

ChatGPT를 이용해 코드를 구현 할 수는 있지만, 사이트를 찾아가며 공부를 해도
내것으로 체득이 된다는 느낌은 없었다.

고민을 하다가, ChatGPT한테 이것도 떠넘겨볼까?란 생각이 들었다.

그래서 아래와 같이 부탁을 해보았다.

> 너가 준 코드가 아주 훌륭한데, 나는 고랭도 익숙하지 않고 패키지도 익숙하지 않거든. 그래서 코드를 100% 내껄로 체득하기가 너무 어려워. 너가 10단계로 나누어서 하나씩 하나씩 나에게 알려줄 수 있어? 빈칸 채우기도 좋고 아니면 너가 설명이나 힌트를 주면 내가 코드를 짤게 그렇게 해서 위의 코드를 내가 직접 짜고 이해할 수 있게 해줘

참고로 모델은 GPT-4o를 사용함

바로 1단계 시작

## **🔥 단계 1: Gin 서버 기본 설정 및 실행**

**목표:** Gin을 사용해 **기본 웹 서버를 띄우기**

**힌트:**

- `gin.Default()`를 사용해 라우터를 생성
- `r.Run(":8080")`으로 서버 실행

**🛠️ 네가 작성할 코드 (완성해 봐!)**

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	// 1️⃣ Gin 라우터 생성
	r := ____________

	// 2️⃣ 서버 실행 (포트 8080)
	___________
}

```

내 답변

```go
package main

import {
    "github.com/gin-gonic/gin"
}

func main() {
    r := gin.default()

    r.run("8080")
}
```


당연히 틀린 부분이 많았고, 아래처럼 힌트를 줬다.

### **❌ 틀린 부분 & 힌트**

1. **`import { ... }` → `import (...)`**
        - Go에서는 `{}` 대신 `()`를 사용해 여러 패키지를 import해.
2. **`gin.default()` → `gin.Default()`**
    - Go에서는 함수나 구조체의 첫 글자가 **대문자**여야 호출 가능해.
3. **`r.run("8080")` → `r.Run(":8080")`**
    
    - 포트 앞에 `:`가 필요해.
    - `Run()` 메서드는 첫 글자를 **대문자**로 써야 해.

오, 생각보다 머리에 쏙쏙 들어온다.
바로 고침

```go
package main

import (
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()

    r.Run(":8080")
}
```


### ✅ **2단계: "Hello, World!" API 추가하기**

이제 `/hello` 엔드포인트를 추가해 보자.  
이 엔드포인트를 호출하면 `"Hello, World!"`를 JSON으로 반환해야 해.

#### **📌 힌트**

1. `r.GET()`을 사용해서 `"/hello"` 경로를 추가해야 해.
2. `gin.Context`를 사용해서 `c.JSON()`으로 응답을 보내야 해.
3. `c.JSON(http.StatusOK, gin.H{"message": "Hello, World!"})` 형식으로 응답을 보내면 돼.


처음 내 답변. 역시 보는 것과 쓰는 것은 다르다.
여러 번 봤는데도 기억이 안 나서 막 적었다..
화를 내지 않는 기계 선생님이라 다행..

```go
package main

import (
    "http"
    "github.com/gin-gonic/gin"
)

func helloWorld(p gin.Context()){
    c.JSON(http.StatusOK, gin.H({"message" : "Hello, world!"}))
}

func main() {
    r := gin.Default()
    r := r.GET("/hello")

    r.Run(":8080")
}
```


### ❌ **틀린 부분 & 수정해야 할 것**

1. `"http"` → `"net/http"`
    - `http.StatusOK`를 사용하려면 `"net/http"` 패키지를 불러와야 해.
2. `helloWorld(p gin.Context())` → `helloWorld(c *gin.Context)`
    - `gin.Context`는 **포인터로 받아야 해 (`*gin.Context`)**
    - `p` 대신 `c`를 쓰는 게 일반적이야.
3. `gin.H({"message": "Hello, world!"})` → `gin.H{"message": "Hello, world!"}`
    - `gin.H`는 맵이라서 `()`가 아니라 `{}`를 써야 해.
4. `r := r.GET("/hello")` → `r.GET("/hello", helloWorld)`
    - `r.GET()`은 핸들러 함수를 받아야 해.

위에 힌트들 보고 수정함

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func helloWorld(c *gin.Context()){
    c.JSON(http.StatusOK, gin.H{"message" : "Hello, world!"})
}

func main() {
    r := gin.Default()
    r.GET("/hello", helloWorld)
    r.Run(":8080")
}
```

수정은 했지만, `gin.Context`랑 `gin.H` 등이 이해가 잘 가지 않았다.
그래서 아래와 같이 물어보았다.

> 그런데, gin.Context랑 c.JSON, gin.H를 잘 모르겠거든. 설명해줄수있어?

바로 알려줌.. 대박!
### 🔍 **궁금한 개념들 정리해줄게!**

1. **`gin.Context`란?**
    
    - `gin.Context`는 **요청(Request)과 응답(Response)** 을 관리하는 핵심 객체야.
    - 클라이언트가 보낸 요청 데이터(쿼리, 바디, 헤더)를 읽거나, 응답을 보낼 때 사용해.
    
    ```go
func handler(c *gin.Context) {
    body := c.PostForm("name") // POST 데이터에서 "name" 값 읽기
    c.String(http.StatusOK, "Hello, " + body)
}
```

---

1. **`c.JSON()`이란?**
    
    - 클라이언트에게 JSON 형식의 응답을 보내는 함수야.
    - 첫 번째 인자: HTTP 상태 코드
    - 두 번째 인자: JSON으로 변환할 데이터 (`map`, `struct`, `gin.H` 등)
    
```go
c.JSON(http.StatusOK, gin.H{"message": "Hello!"})
```


---

1. **`gin.H`란?**
    
    - `map[string]interface{}` 타입의 **JSON 데이터 생성 도우미**
    - `{}` 안에 키-값을 넣어서 JSON 응답을 만들 때 편해.

```go
gin.H{
    "name": "Alice",
    "age": 25,
}

```

    **동일한 코드**
    
```go
map[string]interface{}{
    "name": "Alice",
    "age": 25,
}

```


---

📌 **정리하면?**

- `gin.Context` → 요청과 응답을 관리하는 핵심 객체
- `c.JSON()` → JSON 형식의 응답을 반환
- `gin.H` → JSON 데이터를 쉽게 만들도록 도와주는 헬퍼

---

💡 **이제 다음 단계로 가볼까?**

1. 쿼리 파라미터(`c.Query()`) 받는 법
2. JSON 바디(`c.BindJSON()`) 처리하는 법

어떤 걸 먼저 하고 싶어? 🚀