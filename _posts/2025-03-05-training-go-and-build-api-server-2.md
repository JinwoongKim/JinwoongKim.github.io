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

### **🔹 6단계: 헤더(Header) 및 쿼리 스트링 활용하기**

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

bash

복사편집

`curl -X GET "http://localhost:8080/auth"`

json

복사편집

`{"error": "API Key missing"}`

### ✅ API Key 포함하여 호출 (성공)

bash

복사편집

`curl -X GET "http://localhost:8080/auth" -H "Authorization: Bearer my-secret-key"`

json

복사편집

`{"message": "Authorized", "api_key": "Bearer my-secret-key"}`

---

## **🔍 정리**

1️⃣ `c.GetHeader("Authorization")` → 헤더에서 값 가져오기  
2️⃣ `c.Query("query")` → URL에서 쿼리 스트링 가져오기  
3️⃣ API Key가 없으면 `401 Unauthorized` 반환  
4️⃣ API Key가 있으면 정상 응답

---

### **🚀 직접 코드 작성해보고 실행해봐!**

완성된 코드를 올리면 검토해줄게! 😃
