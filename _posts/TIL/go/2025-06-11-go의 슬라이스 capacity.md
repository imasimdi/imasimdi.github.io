---
layout: post
category: go
---

## 슬라이스 초기화 차이점

- **`var slice []int`**는 **nil slice** (아직 메모리 할당 X)
- **`slice2 := []int{}`**는 **빈 슬라이스** (0길이 배열 가리킴, 이미 메모리 할당 O)
- nil 여부와 직렬화 결과 등에서 차이가 있으니 상황에 맞게 사용해야함

## capacitiy에 관하여

- 슬라이스는 힙에 저장되기 때문에, 슬라이스를 잘게 쪼게도 같은 주소 값을 갖음
- 따라서 `append` 를 할 때, slice의 용량이 남아있다면 slice 전체에 영향이 갈 수 있음

```go
func main() {
	original := []int{1, 2, 3, 4, 5}

	// full slice expression으로 용량(cap)을 제한
	limited := original[1:3:4] // len=2, cap=3 (4-1)

	fmt.Println("Before append:")
	fmt.Println("original:", original) // [1 2 3 4 5]
	fmt.Println("limited:", limited)   // [2 3]

	limited = append(limited, 100) // [2 3 100]
	fmt.Println("After append beyond capacity:")
	fmt.Println("original:", original) // [1 2 3 100 5] (원본 슬라이스의 원소도 변경)
	fmt.Println("limited:", limited)   // [2 3 100]
	fmt.Printf("original 주소: %p, limited 주소: %p\n", &original[1], &limited[0])
}
// original 주소: 0xc00001e188, limited 주소: 0xc00001e188 -> 여전히 같은 주소
```

하지만 용량을 len과 같게한 뒤, append를 하면 cap을 초과하기 때문에 새로운 배열이 할당된다.

```sql
func main() {
	original := []int{1, 2, 3, 4, 5}

	// full slice expression으로 용량(cap)을 제한
	limited := original[1:3:3] // len=2, cap=2 (3-1)

	fmt.Println("Before append:")
	fmt.Println("original:", original) // [1 2 3 4 5]
	fmt.Println("limited:", limited)   // [2 3]

	limited = append(limited, 100) // [2 3 100]
	fmt.Println("After append beyond capacity:")
	fmt.Println("original:", original) // [1 2 3 4 5] (변화 없음)
	fmt.Println("limited:", limited)   // [2 3 100] (새로운 배열 할당)
	fmt.Printf("original 주소: %p, limited 주소: %p\n", &original[1], &limited[0])
}
// original 주소: 0xc00001e188, limited 주소: 0xc000020100 -> 새로운 배열이 할당되어서 다른 주소 
```