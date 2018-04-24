Examples
========

1. [계산된 변수 (Calculated variable)](#계산된-변수)
1. [간단한 UI 바인딩 (Simple UI bindings)](#간단한-UI-바인딩)
1. [자동 입력 유효성 확인 (Automatic input validation)](#자동-입력-유효성-확인)
1. [더 많은 예제](../RxExample)
1. [플레이그라운드](Playgrounds.md)

## 계산된 변수

먼저, 몇가지 명령형 코드를 봅시다.
이 예제의 목적은 `a` 와 `b` 사이에서 나온 계산된 값이 조건을 만족하면 `c` 식별자에 바인딩하는 것입니다.

여기 `c`에게 값을 계산해주는 명령형 코드가 있습니다.

```swift
// 이것은 기본적인 명령형 코드입니다
var c: String
var a = 1       // 여기서는 `1`이라는 값을 `a`에 한 번 할당할 것입니다
var b = 2       // 여기서는 `2`라는 값을 `b`에 한 번 할당할 것입니다

if a + b >= 0 {
    c = "\(a + b) is positive" // 여기서는 `c`에 결과값을 한 번 할당할 것입니다
}
```

이제 `c`의 값은 `3 is positive`일 것입니다. 하지만 우리가 `a`의 값ㅇ르 `4`로 바꿔도, `c`의 값은 여전히 예전의 값일 것입니다.

```swift
a = 4   // `c`는 여전히 제대로 된 게 아닌 "3 is positive"라는 값을 가지고 있을 것입니다
        // 4 + 2 = 6이기 때문에, `c`가 "6 is positive"라는 값을 가지기를 원합니다
```

이것은 원했던 결과가 아닙니다.

다음은 RxSwift를 통해 더 발전된 로직입니다.

```swift
let a /*: Observable<Int>*/ = Variable(1)   // a = 1
let b /*: Observable<Int>*/ = Variable(2)   // b = 2

// `+`를 사용해서 변수 `a`와 `b`의 최근 값을 조합합니다
let c = Observable.combineLatest(a.asObservable(), b.asObservable()) { $0 + $1 }
	.filter { $0 >= 0 }               // `a + b >= 0`가 참이면, `a + b`가 map 연산자로 넘어갑니다
	.map { "\($0) is positive" }      // `a + b`를 매핑해서 "\(a + b) is positive"로 만듭니다

// 초기값은 a = 1, b = 2 이었고
// 1 + 2 = 3 은 0보다 큽니다. 그래서 `c`의 초기값은 "3 is positive"가 됩니다.

// Rx `옵저버블`인 `c`의 값을 가져오려면, `c`의 값을 구독해야 합니다.
// `subscribe(onNext:)`는 `c`의 다음 (신선한)값에 대해서 구독한다는 것을 의미합니다.
// 그것은 초기값인 "3 is positive"도 포함한다는 의미입니다.
c.subscribe(onNext: { print($0) })          // "3 is positive"를 출력합니다

// 이제 `a`의 값을 증가시켜 보겠습니다
a.value = 4                                   // "6 is positive"를 출력합니다
// 최근 값인 `4`와 `2`의 합은 `6`이 됩니다.
// 이 값은 `>= 0` 조건을 만족하고, `map` 연산자는 "6 is positive"를 생산합니다.
// 그리고 그 결과는 `c`에 "할당"됩니다.
// `c`의 값이 바뀌었기 때문에, `{ print($0) }` 이 호출됩니다.
// 그리고 "6 is positive"가 출력됩니다.

// 이제 `b`의 값을 바꿔보겠습니다
b.value = -8                                 // 아무것도 출력하지 않습니다
// 최근 값들을 합해보면 `4 + (-8)`의 결과는 `-4`입니다.
// 이 값은 `>= 0`의 조건을 만족하지 않으므로, `map` 연산자는 실행되지 않습니다.
// 이것은 `c`가 여전히 "6 is positive" 라는 값을 가지고 있을 것을 의미합니다.
// `c`가 업데이트되지 않았기 떄문에, 새로운 "다음" 값은 생성되지 않습니다.
// 그리고 `{ print($0) }` 도 호출되지 않습니다.
```

## 간단한 UI 바인딩

* Instead of binding to variables, let's bind to `UITextField` values using the `rx.text` property
* Next, `map` the `String` into an `Int` and determine if the number is prime using an async API
* If the text is changed before the async call completes, a new async call will replace it via `concat`
* Bind the results to a `UILabel`

```swift
let subscription/*: Disposable */ = primeTextField.rx.text      // type is Observable<String>
            .map { WolframAlphaIsPrime(Int($0) ?? 0) }          // type is Observable<Observable<Prime>>
            .concat()                                           // type is Observable<Prime>
            .map { "number \($0.n) is prime? \($0.isPrime)" }   // type is Observable<String>
            .bind(to: resultLabel.rx.text)                        // return Disposable that can be used to unbind everything

// This will set `resultLabel.text` to "number 43 is prime? true" after
// server call completes.
primeTextField.text = "43"

// ...

// to unbind everything, just call
subscription.dispose()
```

All of the operators used in this example are the same operators used in the first example with variables. There's nothing special about it.

## 자동 입력 유효성 확인

If you are new to Rx, the next example will probably be a little overwhelming at first. However, it's here to demonstrate how RxSwift code looks in the real-world.

This example contains complex async UI validation logic with progress notifications.
All operations are canceled the moment `disposeBag` is deallocated.

Let's give it a shot.

```swift
enum Availability {
    case available(message: String)
    case taken(message: String)
    case invalid(message: String)
    case pending(message: String)
    
    var message: String {
        switch self {
        case .available(message: let message),
             .taken(message: let message),
             .invalid(message: let message),
             .pending(message: let message): 

             return message
        }
    }
}

// bind UI control values directly
// use username from `usernameOutlet` as username values source
self.usernameOutlet.rx.text
    .map { username in

        // synchronous validation, nothing special here
        if username.isEmpty {
            // Convenience for constructing synchronous result.
            // In case there is mixed synchronous and asynchronous code inside the same
            // method, this will construct an async result that is resolved immediately.
            return Observable.just(Availability.invalid(message: "Username can't be empty."))
        }

        // ...

        // User interfaces should probably show some state while async operations
        // are executing.
        let loadingValue = Availability.pending(message: "Checking availability ...")

        // This will fire a server call to check if the username already exists.
        // Its type is `Observable<ValidationResult>`
        return API.usernameAvailable(username)
          .map { available in
              if available {
                  return Availability.available(message: "Username available")
              }
              else {
                  return Availability.unavailable(message: "Username already taken")
              }
          }
          // use `loadingValue` until server responds
          .startWith(loadingValue)
    }
// Since we now have `Observable<Observable<ValidationResult>>`
// we need to somehow return to a simple `Observable<ValidationResult>`.
// We could use the `concat` operator from the second example, but we really
// want to cancel pending asynchronous operations if a new username is provided.
// That's what `switchLatest` does.
    .switchLatest()
// Now we need to bind that to the user interface somehow.
// Good old `subscribe(onNext:)` can do that.
// That's the end of `Observable` chain.
    .subscribe(onNext: { validity in
        errorLabel.textColor = validationColor(validity)
        errorLabel.text = validity.message
    })
// This will produce a `Disposable` object that can unbind everything and cancel
// pending async operations.
// Instead of doing it manually, which is tedious,
// let's dispose everything automagically upon view controller dealloc.
    .disposed(by: disposeBag)
```

It doesn't get any simpler than that. There are [more examples](../RxExample) in the repository, so feel free to check them out.

They include examples on how to use Rx in the context of MVVM pattern or without it.
