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


### **🔹 `r.POST()`의 역할과 사용법**

`r.POST()`는 Gin 프레임워크에서 **POST 요청을 처리할 라우트를 등록하는 함수**야.  
기본적인 형식은 아래와 같아.

```go
r.POST("/path", handlerFunction)
```


- 첫 번째 인자: `"경로"`
- 두 번째 인자: 요청을 처리할 핸들러 함수

---

### **📌 1. `POST` 요청에서 파라미터 전달 방식**

#### **(1) 요청 본문 (Body)에서 JSON 데이터 받기**

> API 요청의 데이터는 보통 JSON 형태로 `Body`에 담겨서 전달돼.  
> `c.ShouldBindJSON()`을 사용하면 자동으로 `struct` 또는 `map`으로 변환할 수 있어.

```go
type RequestBody struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

func createUser(c *gin.Context) {
    var body RequestBody
    if err := c.ShouldBindJSON(&body); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid JSON format"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"message": "User created", "data": body})
}

```

#### **(2) 쿼리 파라미터 (`?key=value`)**

> `c.Query("key")`를 사용해서 쿼리 파라미터를 읽을 수 있어.  
> 예를 들어 `/search?query=golang` 같은 요청이 오면 `query` 값을 가져올 수 있어.

```go
func search(c *gin.Context) {
    query := c.Query("query") // 없으면 "" 반환
    if query == "" {
        query = "default"
    }

    c.JSON(http.StatusOK, gin.H{"message": "Searching for " + query})
}
```


#### **(3) URL 파라미터 (`/path/:value`)**

> `c.Param("key")`를 사용해서 URL 경로에 포함된 값을 가져올 수 있어.

```go
func getUser(c *gin.Context) {
    userID := c.Param("id") // /user/123 → userID = "123"
    c.JSON(http.StatusOK, gin.H{"user_id": userID})
}
```

---

### **📌 2. `POST` 요청의 주의점 (라우팅 가이드)**

✅ **JSON 데이터를 받을 때는 `c.ShouldBindJSON()`을 사용하자.**  
✅ **쿼리 파라미터와 바디(JSON) 데이터는 다르다.**  
✅ **URL에 `/:param`을 사용할 때는 `c.Param()`으로 가져온다.**  
✅ **중복되는 엔드포인트(`/users/:id` vs `/users/search` 등) 주의해야 한다.**  
✅ **RESTful API 규칙을 따르는 것이 좋다.**

---

### **💡 예제 코드 (모든 방식 포함)**

```go
func main() {
    r := gin.Default()

    // Body에서 JSON 데이터 받기
    r.POST("/users", createUser) 

    // 쿼리 파라미터 받기
    r.GET("/search", search) 

    // URL 파라미터 받기
    r.GET("/users/:id", getUser) 

    r.Run(":8080")
}

```


---

이제 **`r.POST()`를 사용해서 API 요청을 처리할 때 주의해야 할 점과 활용법을 알았어!**  
이해됐으면 다음 단계로 가자 🚀

> 쿼리파라미터는 언제 써?

###### ChatGPT의 말:

### **📌 쿼리 파라미터 (`?key=value`)를 언제 쓰는가?**

쿼리 파라미터는 **필터링, 정렬, 검색, 옵션 설정** 같은 용도로 주로 사용돼.  
즉, **리소스를 특정 조건으로 조회하거나 부가적인 정보를 추가할 때** 적합해.

---

### **✅ 1. 필터링 & 검색**

🔹 **예제:** `/products?category=electronics&brand=apple`  
🔹 **설명:** 특정 카테고리나 브랜드로 상품을 필터링할 때 사용

```go
func searchProducts(c *gin.Context) {
    category := c.Query("category") // 기본값 없음 ("" 반환)
    brand := c.Query("brand")

    c.JSON(http.StatusOK, gin.H{"message": "Filtering", "category": category, "brand": brand})
}

```


✔️ **언제 사용?**  
✅ 사용자가 검색 옵션을 선택할 때 (`/search?query=golang`)  
✅ 특정 카테고리/태그/조건을 선택할 때 (`/posts?tag=tech`)

---

### **✅ 2. 정렬 & 페이지네이션**

🔹 **예제:** `/articles?sort=latest&page=2&limit=10`  
🔹 **설명:** 최신순으로 정렬하고, 2페이지에서 10개만 가져옴

```go
func getArticles(c *gin.Context) {
    sort := c.DefaultQuery("sort", "latest") // 기본값 설정 가능
    page := c.DefaultQuery("page", "1")
    limit := c.DefaultQuery("limit", "10")

    c.JSON(http.StatusOK, gin.H{"message": "Fetching articles", "sort": sort, "page": page, "limit": limit})
}

```


✔️ **언제 사용?**  
✅ 정렬 옵션 (`/posts?sort=desc`)  
✅ 페이지네이션 (`/users?page=3&limit=20`)

---

### **✅ 3. 부가적인 요청 옵션**

🔹 **예제:** `/download?format=pdf&quality=high`  
🔹 **설명:** 특정 파일 형식과 품질 옵션을 추가

```go
func downloadFile(c *gin.Context) {
    format := c.DefaultQuery("format", "pdf")
    quality := c.DefaultQuery("quality", "medium")

    c.JSON(http.StatusOK, gin.H{"message": "Downloading file", "format": format, "quality": quality})
}

```


✔️ **언제 사용?**  
✅ 응답 형식 선택 (`/export?format=json`)  
✅ 추가 옵션 (`/video?quality=hd&subtitles=on`)

---

### **🚀 정리: 언제 `쿼리 파라미터`를 써야 할까?**

✔️ **리소스를 조회할 때** (검색, 필터링)  
✔️ **리소스를 정렬할 때** (정렬, 페이지네이션)  
✔️ **옵션을 추가할 때** (응답 형식, 품질 설정)  
✔️ **서버 상태를 변경하지 않을 때** (쿼리 파라미터는 GET 요청에서 많이 사용)

---

쿼리 파라미터는 데이터 조회(READ)에 적합하고, **데이터 변경(CREATE, UPDATE, DELETE)** 작업에는 보통 `POST/PUT/DELETE`의 **Body**를 사용해! 🚀

> c.ShouldBindJSON()랑 c.BindJSON()차이가 뭐고 언제 뭘 써?

###### ChatGPT의 말:

### **🔍 `c.ShouldBindJSON()` vs `c.BindJSON()` 차이 및 사용법**

둘 다 JSON 요청 바디를 구조체로 매핑하는 역할을 하지만, **에러 핸들링 방식이 다름**.

---

### ✅ **1. `c.BindJSON()` → 내부적으로 `c.ShouldBindJSON()`을 호출하지만, 에러 발생 시 자동으로 400 응답 반환**

```go
func handler(c *gin.Context) {
    var reqBody RequestStruct
    if err := c.BindJSON(&reqBody); err != nil {
        return // 자동으로 400 Bad Request 응답 반환
    }
    c.JSON(http.StatusOK, gin.H{"data": reqBody})
}
```


✔ **특징:**

- 에러가 발생하면 **자동으로 `400 Bad Request` 응답 반환**
- **명시적인 에러 처리가 필요 없음**
- **간단한 API에서 빠르게 처리할 때 적합**

📌 **언제 사용?**  
✔ **에러를 따로 처리할 필요 없을 때**  
✔ 요청 형식이 단순하고, 잘못된 요청은 그냥 `400` 응답을 주면 되는 경우

---

### ✅ **2. `c.ShouldBindJSON()` → 직접 에러 핸들링 가능**

```go
func handler(c *gin.Context) {
    var reqBody RequestStruct
    if err := c.ShouldBindJSON(&reqBody); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"data": reqBody})
}

```


✔ **특징:**

- 에러 발생 시 자동 응답 X → 직접 처리해야 함
- **커스텀 에러 메시지를 반환 가능**
- **더 정교한 에러 핸들링이 필요할 때 적합**

📌 **언제 사용?**  
✔ **에러 메시지를 커스텀하고 싶을 때**  
✔ **다양한 바인딩 로직이 필요할 때** (예: `Query`, `Form`, `XML` 등)  
✔ **400 이외의 다른 상태 코드를 반환해야 할 때**

---

### **🚀 정리: 언제 뭘 써야 할까?**

✅ **`c.BindJSON()`** → 단순한 API에서 사용 (자동 400 응답)  
✅ **`c.ShouldBindJSON()`** → 에러 핸들링이 필요할 때 사용 (직접 처리 가능)


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


> 1차 코드 작성을 했어

```go
func callExternalAPI() {
  url := "https://example.com/api"

  requestBody := strings.NewReader({"key":"value"})
  resp, err := http.POST(url, "application/json", requestBody)
  if err != nil {
    fmt.Println("Error:", err)
    return
  }
  defer resp.Body.close()

  body, err := io.ReadAll(resp.Body)
  if err != nil {
    fmt.Println("Error reading response:", err)
    return
  }

  fmt.Println("Response:",string(body))
}
```


코드에 몇 가지 문제가 있어. 수정해야 할 부분을 정리해 줄게.

### 🔴 **수정해야 할 부분**

1. **`http.POST` → `http.Post` (대문자 오류)**
    - Go의 `http` 패키지에서는 `Post` 함수가 소문자로 시작해야 해.
2. **`defer resp.Body.close()` → `defer resp.Body.Close()` (대소문자 오류)**
    - `Close()`는 대문자로 시작해야 해.
3. **`requestBody := strings.NewReader(`{"key":"value"}`)`**
    
    - 문자열을 `strings.NewReader`로 감싸는 건 괜찮지만, 만약 동적으로 JSON을 생성하려면 `json.Marshal`을 사용하는 게 더 좋아.

---

### ✅ **수정된 코드**

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
)

func callExternalAPI() {
	url := "https://example.com/api"

	// JSON 데이터 생성
	requestData := map[string]string{"key": "value"}
	requestBody, err := json.Marshal(requestData)
	if err != nil {
		fmt.Println("Error encoding JSON:", err)
		return
	}

	// HTTP POST 요청 보내기
	resp, err := http.Post(url, "application/json", bytes.NewBuffer(requestBody))
	if err != nil {
		fmt.Println("Error:", err)
		return
	}
	defer resp.Body.Close() // 대문자 수정

	// 응답 읽기
	body, err := io.ReadAll(resp.Body)
	if err != nil {
		fmt.Println("Error reading response:", err)
		return
	}

	fmt.Println("Response:", string(body))
}

```


---

### 🔥 **추가 팁**

- `json.Marshal()`을 사용하면 `map[string]string`을 JSON으로 변환해 안전하게 처리할 수 있어.
- `strings.NewReader`를 그대로 사용해도 되지만, 나중에 동적으로 데이터를 생성해야 한다면 `bytes.NewBuffer()`가 더 적절해.
- `fmt.Println()`으로 에러 메시지를 출력하는 것도 괜찮지만, `log.Fatal()`을 쓰면 더 직관적일 수도 있어.

🚀 **이제 실행해보고 정상 동작하는지 확인해봐!**

1,2번이야 오탈자인데, 3번은 지가 알려준대로 했는데 어이가 없네;;

> bytes.NewBuffer(requestBody)랑 resp.Body.Close() 가 이해가 잘 안돼

###### ChatGPT의 말:

### 1️⃣ `bytes.NewBuffer(requestBody)`

#### ❓ **왜 `bytes.NewBuffer()`를 쓰는가?**

`http.Post()`의 세 번째 인자는 `io.Reader` 타입을 요구해.  
하지만 `json.Marshal()` 등으로 만든 `requestBody`는 `[]byte` 타입이야.  
이걸 `io.Reader`로 변환하려면 `bytes.NewBuffer()`를 사용해야 해.

#### 📌 **비교**

```go
requestBody, _ := json.Marshal(map[string]string{"key": "value"})

// ❌ 이렇게 하면 오류 (io.Reader 타입이 아님)
// http.Post(url, "application/json", requestBody)

// ✅ 올바른 사용법
http.Post(url, "application/json", bytes.NewBuffer(requestBody))

```


`bytes.NewBuffer()`는 `[]byte` 데이터를 `io.Reader` 인터페이스를 구현하는 객체로 변환해 줘서  
`http.Post()`에서 사용할 수 있게 해.

---

### 2️⃣ `resp.Body.Close()`

#### ❓ **왜 `resp.Body.Close()`를 해야 하나?**

`http.Post()` 요청이 성공하면 응답 객체 `*http.Response`가 반환되는데,  
이 안의 `Body`는 스트림(`io.ReadCloser`) 형태로 되어 있어.  
따라서 사용이 끝나면 `Close()`를 호출해서 리소스를 해제해야 해.

#### 📌 **비교**

```go
resp, err := http.Post(url, "application/json", bytes.NewBuffer(requestBody))
if err != nil {
    fmt.Println("Error:", err)
    return
}

// ❌ Close() 안 하면 메모리 누수 발생 가능
body, _ := io.ReadAll(resp.Body)
fmt.Println(string(body))

// ✅ defer로 Close() 예약 (안전한 방법)
defer resp.Body.Close()
```


응답 본문을 다 읽고 나면 반드시 `Close()`를 호출해야 리소스가 정리돼.  
특히 `defer resp.Body.Close()`를 쓰면 함수가 끝날 때 자동으로 닫혀서 안전해.

> 왜 http.Post()의 세 번째 인자는 io.Reader 타입을 요구해? 그리고 io.Reader가 뭐야?

###### ChatGPT의 말:

### 1️⃣ **왜 `http.Post()`의 세 번째 인자는 `io.Reader` 타입을 요구할까?**

`http.Post(url string, contentType string, body io.Reader)` 함수에서  
세 번째 인자인 `body`는 **요청 본문(request body)**을 의미해.  
Go의 표준 라이브러리에서는 HTTP 요청의 본문을 **스트림(stream)** 방식으로 처리해.

📌 **스트림이 필요한 이유**

- HTTP 요청은 보통 **큰 데이터를 포함**할 수 있어 (예: 파일 업로드, JSON 데이터).
- 한 번에 모든 데이터를 메모리에 올리는 방식이 아니라,  
    **필요할 때마다 데이터를 읽어오는 방식**(스트리밍)으로 처리하는 게 더 효율적이야.
- `io.Reader`는 이 스트림 방식으로 데이터를 읽을 수 있도록 해.

---

### 2️⃣ **`io.Reader`가 뭐야?**

`io.Reader`는 Go의 **인터페이스(interface)** 중 하나야.  
이 인터페이스를 구현한 타입은 **읽기 기능(Read method)**을 제공해야 해.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

📌 **즉, `Read()` 메서드를 가진 타입은 모두 `io.Reader`로 쓸 수 있어.**