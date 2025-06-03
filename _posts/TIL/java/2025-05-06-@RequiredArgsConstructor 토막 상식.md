---
layout: post
category: java
---

Spring에서 생성자를 통한 의존성 주입을 할 때, 클래스에 단일 생성자가 있는 경우 해당 생성자를 사용해서 자동으로 의존성 주입을 시켜줍니다.

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
         this.userRepository = userRepository;
     }
```

여기에서 `@RequiredArgsConstructor` 롬복을 붙여주면 컴파일 시점에서 final 필드나 @Nonnull 로 표시된 필드를 파라미터로 받는 생성자를 자동으로 생성합니다.

위의 코드에서 @RequiredArgsConstructor를 붙이면 다음과 같이 간략해집니다.

```java
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;  // final 키워드 중요!
    
    // @RequiredArgsConstructor가 다음과 같은 생성자를 자동 생성
    // public UserService(UserRepository userRepository) {
    //     this.userRepository = userRepository;
    // }
    
    public User findUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

해당 롬복을 사용하면 얻을 수 있는 장점은 여러가지 있습니다.`final` 필드를 사용하기 때문에 객체의 불변성을 보장하고, 무엇보다도 코드의 가동성이 좋아집니다. 반복되는 생성자 코드를 만들지 않아도 되니깐요

## @AllArgsConstructor과의 차이점

주요한 차이점은 @RequiredArgsConstructor은 final 필드와 NonNull 필드만 매개변수로 받는 생성자를 생성합니다. 즉 다음과 같은 경우 생성자에 포함되는 것은 userRepository 입니다.

```java
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository; // 생성자에 포함됨
    private String name; // final이 아니므로 생성자에 포함되지 않음
}
```

반면 @AllArgsConstructor은 클래스의 모든 필드를 매개변수로 받는 생성자를 생성합니다. 즉, final이 아닌 필드도 모두 포함할 뿐더러 모든 필드의 값을 초기화해야 하기 때문에 의존성 주입에는 적합하지 않을 수 있습니다.

```java
@AllArgsConstructor
public class UserService {
    private final UserRepository userRepository; // 생성자에 포함됨
    private String name; // 생성자에 포함됨
}
```