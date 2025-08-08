---
layout: post
category: go
---

log로도 panic을 발생시킬 수 있는데, error와 panic의 차이점은 무엇일까요?

- `error`는 예상 가능한 문제(예: 파일 없음, 네트워크 연결 끊김)를 나타내며, 호출자가 복구하거나 처리할 것으로 기대됩니다.
- `panic`은 예상치 못했거나 프로그램이 더 이상 안전하게 실행될 수 없는 치명적인 상태를 나타냅니다.

go는 다른 언어처럼 예외를 raise하는 것이 아닌 err를 반환해서 오류를 처리합니다. 그래서 panic을 exception처럼 사용한다면 .. 그냥 애플리케이션이 종료됩니다. 아주 치명적이죠?

그래서 panic은 주로 복구로직을 포함하는데, `defer`를 사용해서 이를 구현합니다.

```go
package main

import "fmt"

func main() {
	fmt.Println("프로그램 시작")
	defer func() {
		if r := recover(); r != nil { // recover()가 nil이 아니면 panic이 발생했음을 의미
			fmt.Println("Panic 감지 및 복구:", r)
			// 여기에서 오류를 로깅하거나, 사용자에게 알리거나,
			// 프로그램의 다른 부분을 초기화하는 등의 복구 작업을 수행
		}
	}()

	// panic이 발생할 수 있는 함수 호출
	doSomethingRisky(10) // 이 호출에서 panic 발생

	fmt.Println("프로그램 종료") // recover 덕분에 이 줄이 실행됩니다.
}

func doSomethingRisky(val int) {
	fmt.Println("doSomethingRisky 함수 시작")
	if val > 5 {
		panic("값이 너무 큽니다! panic 발생!") // 여기서 panic이 발생합니다.
	}
	fmt.Println("doSomethingRisky 함수 종료") // 이 줄은 실행되지 않습니다.
}
```

이렇게 main 함수에서 defer로 익명 함수를 등록합니다. `doSomethingRisky(10)` 에서 일부러 panic을 발생시키고 main 함수의 defer 함수가 실행됩니다.

defer 함수 내부에서는 `recover()` 가 호출이되고, panic의 메시지를 호출합니다.

그리고 그 밑으로 어떠한 복구 로직을 실행시킬 수 있습니다. recover 함수는 defer 내에서만 유효하다고 해요.

따라서 panic은 defer 함수를 실행해야되기 때문에 error보다 성능이 좋지 않습니다. 그래서 일반적으로는 error를 반환해서 이를 처리하는 것이 go에서 사용되고, panic은 복구할 수 없는 예외 상황에서 사용해야 됩니다.

다행히도 fiber와 같은 프레임워크에서는 기본적으로 recover가 미들웨어 형태로 등록이 되어있기 때문에 자동으로 panic을 recover 해준다고 하네요. recover시 `Internal Server Error` 를 반환합니다.