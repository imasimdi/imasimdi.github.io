---
layout: post
category: go
---

## 포인터

### 포인터 기본 개념

- `*`: 포인터가 가르키는 값을 가져옴, 즉 역참조
- `&`: 변수의 메모리 주소를 가져옴
- 함수로 보면 이해가 더 빠릅니다.

```go
// 함수에서 값 변경하기
func modify(x *int) {
    *x = 100  // x가 가리키는 값을 100으로 변경
}

num := 42
modify(&num)  // num의 주소를 전달
fmt.Println(num)  // 출력: 100
```

- 여기서 왜 `*x`를 하냐? → 메모리의 값을 변경하기 위해

### 구조체에서는?

```go
type Person struct {
    Name string
    Age  int
}

func (p *Person) Birthday() {
    p.Age++  // 실제 구조체의 값이 변경됨
}

person := Person{"Kim", 25}
person.Birthday()  // Go는 자동으로 &person.Birthday()로 변환
```

- 메서드 리시버에 포인터를 넘겼기 때문에 person.Age는 26이 된다.
- 만약 `p Person`을 하게되면 값은 바뀌지 않는다

### 구조체를 선언할 때 포인터를 붙이냐 안 붙이냐의 차이

```go
type Person struct {
    Name string
    Age  int
}

// 1. & 없이 선언 (값 타입)
p1 := Person{Name: "Kim", Age: 25}
// p1은 Person 구조체의 값

// 2. & 붙여서 선언 (포인터 타입)
p2 := &Person{Name: "Lee", Age: 30}
// p2는 Person 구조체의 포인터
```

- 이 두 선언의 차이점은 함수에 전달할 때 방법의 차이가 있다.

```go
// 값을 변경하는 함수
func Birthday(p *Person) {
    p.Age++  // 실제 나이가 증가
}

p1 := Person{Name: "Kim", Age: 25}
p2 := &Person{Name: "Lee", Age: 30}

Birthday(&p1)
Birthday(p2)
```

- 값 타입은 &를 붙여서 전달해야 한다. 반면에 포인터 타입은 바로 전달 가능하다.
- 또한 값 타입은 구조체 전체가 복사되어서 p는 원본의 복사본이 된다.
- 반면에 포인터 타입은 주소만 복사되기 때문에 더 효율적이다.
- 또한 값 타입은 nil 타입이 가능하지 않지만 포인터 타입은 nil이 될 수 없다.

### 똑똑한 GO 컴파일러

```go
func (p *Person) Birthday() {
    p.Age++
}

p1 := Person{Name: "Kim", Age: 25}
p2 := &Person{Name: "Lee", Age: 30}

// 
p1.Birthday()
p2.Birthday() 
```

- 메서드에 포인트 리시버를 넣으면 값 타입의 구조체를 넣어도 알아서 Go는 자동으로 필요한 변환을 수행
- 컴파일러가 자동으로 `(&p1).Birthday()` 처럼 처리한다.

## Optional

### 구조체에서 포인터 사용하기
- Go에서는 다른 언어들처럼 내장된 `Optional`이 존재하지 않는다. 대신 포인터를 사용해서 Optional 값을 처리할 수 있다.

```go
type User struct {
    Name     string
    Age      *int
}

user := User{
	Name: "kim"
}

age := 25

user2 := User{
	Name: "Lee",
	Age: &age,
}
```

- 이렇게 포인터를 사용하여 Optional을 다룰 수 있다.

### 왜 `&` 가 들어가요?

- Age가 포인터 타입으로 선언되었기 때문
- 복습하자면 `&`는 변수의 메모리 주소를 가져온다. 여기서 `Age: &age` 는 age 변수의 메모리 주소를 가져와서 Age에 할당시켜준다.
- 그리고 `*` 를 통해서 포인터가 가리키는 메모리 주소에 있는 실제 값을 가져온다.

### 다른 자료구조는요?

### Map

- 맵에서는 기본적으로 Optional 처리를 지원한다.

```go
users := map[string]string{
    "user1": "Kim",
}

if name, exists := users["user1"]; exists {
    fmt.Println("사용자 찾음:", name)
} else {
    fmt.Println("사용자 없음")
}
```

- 기본적으로 값을 찾을 때, python의 `dict.get()` 처럼 없다면 두번째 반환값에 `false`을 반환한다.
- 또한 첫번째 반환값에는 해당 타입의 zero value를 반환한다. 여기서는 값의 타입이 string이었기 때문에 `""`를 반환한다.

### Slice

- 포인터 슬라이스를 사용하면 nil을 통해서 Optional을 표현할 수 있다.

```go
type User struct {
    ID   string
    Name string
    Age  int
}

// 1. 포인터 슬라이스 선언
users := []*User{
    &User{ID: "1", Name: "Kim", Age: 25},
    nil,
    &User{ID: "3", Name: "Park", Age: 35},
}
```

- 마찬가지로 users 슬라이스는 포인터 슬라이스 이기 때문에 User 구조체를 넣을 때 메모리 주소를 넣어주기 위해 `&` 를 붙여준다.
- 포인터의 크기만큼만 메모리를 사용하기 때문에 어느정도 메모리 효율성이 있다.