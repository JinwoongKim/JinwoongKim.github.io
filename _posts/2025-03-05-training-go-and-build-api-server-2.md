---
title: ChatGPT를 이용해 Go언어 배워서 API 서버 개발 하기 (2/2)
categories: study
tags:
  - ChatGPT
  - Go
  - API
published: true
---
이전 포스트에서 이어서..

# 6단계: 헤더(Header) 및 쿼리 스트링 활용하기

이번 단계에서는 **HTTP 헤더(Header)와 쿼리 스트링(Query String)** 을 다룬다.  
이를 통해 **API Key 인증** 같은 작업을 처리할 수 있다.

---

## **📌 목표**

✅ `c.GetHeader("Authorization")`을 사용하여 헤더에서 API Key 읽기  
✅ `c.Query("query")`와 비교하며 쿼리 스트링 이해하기  
✅ API Key가 없으면 `401 Unauthorized` 반환

---

## **1️⃣ 헤더 값 읽기**

아래 코드를 작성해보자.

```go
func authExample(c *gin.Context) {
    apiKey := c.GetHeader("Authorization")
    if apiKey == "" {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "API Key missing"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "Authorized", "api_key": apiKey})
}
```

💡 **힌트**

- `c.GetHeader("Authorization")` → `Authorization` 헤더 값 가져오기
- 값이 없으면 `401 Unauthorized` 반환
- 정상일 경우 API Key를 JSON 응답에 포함

---

## **2️⃣ 쿼리 스트링과 비교**

아래 코드를 추가해보자.

```go
func searchWithAuth(c *gin.Context) {
    apiKey := c.GetHeader("Authorization")
    query := c.Query("query")

    if apiKey == "" {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "API Key missing"})
        return
    }
    if query == "" {
        query = "default"
    }

    c.JSON(http.StatusOK, gin.H{"message": "Searching for " + query, "api_key": apiKey})
}
```


---

## **3️⃣ 라우터 등록하기**

위에서 만든 핸들러를 `main()`에 추가한다.

```go
func main() {
    r := gin.Default()

    r.GET("/auth", authExample) // 헤더 값 확인
    r.GET("/searchAuth", searchWithAuth) // 헤더 + 쿼리 스트링 활용

    r.Run(":8080")
}
```

---

## **4️⃣ 테스트하기**

### ✅ API Key 없이 호출 (실패)

```go
curl -X GET "http://localhost:8080/auth"
```

```json
{"error": "API Key missing"}
```


### ✅ API Key 포함하여 호출 (성공)

```go
curl -X GET "http://localhost:8080/auth" -H "Authorization: Bearer my-secret-key"
```

```json
{"message": "Authorized", "api_key": "Bearer my-secret-key"}
```


---

## **🔍 정리**

1️⃣ `c.GetHeader("Authorization")` → 헤더에서 값 가져오기  
2️⃣ `c.Query("query")` → URL에서 쿼리 스트링 가져오기  
3️⃣ API Key가 없으면 `401 Unauthorized` 반환  
4️⃣ API Key가 있으면 정상 응답

---

### **🚀 직접 코드 작성해보고 실행해봐!**

완성된 코드를 올리면 검토해줄게! 😃

아래 코드 전달함!!

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func helloWorld(c *gin.Context){
    c.JSON(http.StatusOK, gin.H{"message" : "Hello, world!"})
}

func ping(c *gin.Context){
    c.JSON(http.StatusOK, gin.H{"message" : "pong"})
}

func echo(c *gin.Context){
	var jsonData map[string]interface{}
	// var jsonData gin.H, same with above?
	if err := c.BindJSON(&jsonData); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error" : "Invalid JSON"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"received": jsonData})
}

func greet(c *gin.Context) {
    name := c.Param("name")
    c.JSON(http.StatusOK, gin.H{"message": "Hello," + name})
}

func search(c *gin.Context) {
    query := c.Query("query")
    if query == "" {
        query = "default"
    }
    c.JSON(http.StatusOK, gin.H{"message": "Searching for: " + query})
}

func authExample(c *gin.Context) {
    apiKey := c.GetHeader("Authorization")
    if apiKey == "" {
        c.JSON(http.StatusUnauthorized, gin.H{"error message" : "API Key missing"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"message" : "Authorized", "api_key" : apiKey})
}

func searchWithAuth(c *gin.Context) {
    apiKey := c.GetHeader("Authorization")
    query := c.Query("query")

    if apiKey == "" {
        c.JSON(http.StatusUnauthorized, gin.H{"error message" : "API Key missing"})
        return
    }
    if query == "" {
        query = "default"
        return
    }

    c.JSON(http.StatusOK, gin.H{"message" : "Searching for "+query, "api_key" : apiKey})
}

func main() {
    r := gin.Default()
    r.GET("/hello", helloWorld)
    r.GET("/ping", ping)
    r.GET("/greet/:name", greet)
    r.GET("/search", search)
    r.POST("/echo", echo)

    r.GET("/auth", authExample)
    r.GET("/searchAuth", searchWithAuth)
    r.Run(":8080")
}
```

아주 좋아! 🎉  
거의 다 맞았는데, **`searchWithAuth` 함수에서 하나의 실수**가 있어!

### **🔴 수정해야 할 부분**

```go
if query == "" {
    query = "default"
    return // ❌ 여기에 return 있으면 이후 JSON 응답이 실행되지 않음!
}
```


쳇 하나 틀림

7단계 진행하기 전에 라우터 관련 질문

> r.POST에 대해서 자세히 말해줘 파라미터랑 쿼리에 대해서 라우팅할때 주의할점이나 가이드 알려줘

# 7단계 : 외부 API 호출 및 응답 처리하기

> Nova LLM 또는 OpenAI API를 호출하는 코드를 이해하고 직접 구현해보기

#### **📌 목표**

1. `http.Post()`와 `http.NewRequest()` 차이 이해하기
2. 외부 API 요청을 만들고 응답을 파싱하는 과정 학습

---

### **📝 1차 코드 작성 (빈칸 포함)**

```go
func callExternalAPI() {
    url := "https://example.com/api"

    requestBody := strings.NewReader(`{"key": "value"}`)
    resp, err := http._____(url, "application/json", requestBody)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body._____

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        fmt.Println("Error reading response:", err)
        return
    }

    fmt.Println("Response:", string(body))
}
```

✔ **힌트**

1. `http._____(url, "application/json", requestBody)` → `http.Post()` 또는 `http.NewRequest()` 중 어느 걸 써야 할까?
2. `defer resp.Body._____` → 빈칸에 들어갈 함수는?
3. `io.ReadAll(resp.Body)`는 무엇을 하는 코드일까?

---

### **📝 2차 코드 작성 (Nova LLM 호출)**

Nova LLM API를 호출하는 `runV1` 함수가 어떻게 동작하는지 직접 분석하고,  
아래 질문에 답을 해봐.

```go
func runV1(c *gin.Context) {
    apiKey, err := getAPIKey(c)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": err.Error()})
        return
    }

    var openAIRequest map[string]interface{}
    if err := c.ShouldBindJSON(&openAIRequest); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid request format"})
        return
    }

    novaReq := map[string]interface{}{
        "id":           "generated-uuid",
        "service_name": "test-service",
        "request":      openAIRequest,
    }

    requestBody, err := json.Marshal(novaReq)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to encode request"})
        return
    }

    resp, err := http.Post("http://nova-llm-gateway.alpha.tossinvest.bz/api/v1/openai/chat/completions",
        "application/json", bytes.NewBuffer(requestBody))

    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to call Nova LLM"})
        return
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to read response"})
        return
    }

    var novaResponse map[string]interface{}
    if err := json.Unmarshal(body, &novaResponse); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to parse response"})
        return
    }

    c.JSON(resp.StatusCode, novaResponse)
}
```

---

### **💡 질문**

1. `http.Post()`와 `http.NewRequest()`의 차이는?
2. `json.Marshal(novaReq)`가 하는 역할은?
3. `defer resp.Body.Close()`가 없으면 어떤 문제가 발생할까?

🚀 **빈칸을 채우고, 질문에 대한 답을 고민한 후 알려줘!**