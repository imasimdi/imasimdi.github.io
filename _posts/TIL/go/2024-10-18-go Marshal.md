---
layout: post
category: go
---

# `encoding/json`

## Marshal

- Marshal 함수는 GO 데이터 구조를 JSON 형식의 바이트 슬라이스로 변환하는 핵심 기능
- 파이썬의 `json.dumps` 와 유사

```go
json.Marshal(v interface{}) ([]byte, error)
```

- 구조체 필드 태그로 JSON 키 이름을 커스터마이즈 할 수 있다.
    
    ```go
    type Person struct {
        Name string `json:"name"`
        Age  int    `json:"age"`
    }
    ```
    
- 예제
    
    ```go
    package main
    
    import (
        "encoding/json"
        "fmt"
    )
    
    type Person struct {
        Name string `json:"name"`
        Age  int    `json:"age,omitempty"`
        City string `json:"city,omitempty"`
    }
    
    func main() {
        p1 := Person{Name: "Alice", Age: 30}
        p2 := Person{Name: "Bob"}
    
        jsonData1, _ := json.Marshal(p1)
        jsonData2, _ := json.Marshal(p2)
    
        fmt.Printf("p1: %s\n", jsonData1)
        fmt.Printf("p2: %s\n", jsonData2)
    
        jsonDataIndent, _ := json.MarshalIndent(p1, "", "  ")
        fmt.Printf("p1 (들여쓰기): \n%s\n", jsonDataIndent)
    }
    ```
    
    - `Person` 구조체는 JSON 태그를 사용하여 필드 이름을 지정
    - `age`와 `city` 필드에는 `omitempty` 옵션이 있어, 값이 없으면 JSON에서 생략
    - `json.Marshal()`을 사용하여 구조체를 JSON으로 변환
    - `json.MarshalIndent()`를 사용하여 들여쓰기된 JSON을 생성

## Unmarshal

- JSON 데이터를 Go 데이터 구조로 변환, 마치 파이썬의
- 파이썬의 `json.loads` 와 유사
- JSON 키 이름과 구조체 필드 이름이 일치해야 한다.