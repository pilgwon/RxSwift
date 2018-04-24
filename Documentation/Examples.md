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

* 변수에 바인딩하는 것 대신에, `rx.text`를 사용해서 `UITextField`에 값을 바인딩합니다.
* 다음으로, `String`을 `Int`에 `map`을 사용해서 매핑하고 비동기 API를 사용해서 그 숫자가 소수인지 확인합니다.
* 비동기 호출이 완료되기 전에 텍스트가 바뀌면, 기존의 비동기 호출을 새로운 비동기 호출로 `concat`을 사용해서 바꿔치기 합니다.
* 결과를 `UILabel`에 바인딩합니다.

```swift
let subscription/*: Disposable */ = primeTextField.rx.text      // 타입은 Observable<String> 입니다
            .map { WolframAlphaIsPrime(Int($0) ?? 0) }          // 타입은 Observable<Observable<Prime>> 입니다
            .concat()                                           // 타입은 Observable<Prime> 입니다
            .map { "number \($0.n) is prime? \($0.isPrime)" }   // 타입은 Observable<String> 입니다
            .bind(to: resultLabel.rx.text)                      // 모두 해제하고 싶을 때 사용할 Disposable을 반환합니다

// 서버 호출 완료 후에 `resultLabel.text`에는 "number 43 is prime? true"가 입력됩니다.
primeTextField.text = "43"

// ...

// 모두 해제하려면 아래의 코드를 입력하면 됩니다.
subscription.dispose()
```

위 예제에서 쓰이는 모든 연산자는 첫 번째 예제에서 변수와 같이 쓰인 연산자가 동일합니다. 특별한 점이 없다는 뜻입니다.

## 자동 입력 유효성 확인

만약 여러분이 Rx에 처음이라면, 다음 예제는 처음 봤을 때 약간 이해가 안 될 수도 있습니다. 하지만, 이 부분이 RxSwift 코드가 실제 세계에서 어떻게 쓰이는지에 대해 제일 잘 보여줄 것입니다.

이 예제는 복잡한 비동기 UI 유효성 체크 로직을 프로그래스 노티피케이션과 함께 포함하고 있습니다.
모든 연산은 `disposeBag`이 해제되는 순간 취소됩니다.

살펴봅시다.

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

// UI 컨트롤 값을 직접적으로 바인딩합니다
// `usernameOutlet`을 통해 username을 사용합니다
self.usernameOutlet.rx.text
    .map { username in

        // 동기 유효성 체크하는 부분입니다. 여긴 특별할 것이 없습니다.
        if username.isEmpty {
            // 동기 결과를 만들어낼 때 좀 더 간편하기 위해.
            // 동기 코드와 비동기 코드가 한 메소드에 섞여 있을 경우에,
            // 다음과 같은 코드는 즉시 해결되도록 비동기 결과를 만들어냅니다.
            return Observable.just(Availability.invalid(message: "Username can't be empty."))
        }

        // ...

        // UI는 비동기 작업이 돌아가는 동안에도 무언가를 보여주어야 할 것입니다.
        let loadingValue = Availability.pending(message: "Checking availability ...")

        // 이 코드는 username이 사용할 수 있는 지 서버 호출을 하는 역할을 합니다.
        // 타입은 `Observable<ValidationResult>` 입니다.
        return API.usernameAvailable(username)
          .map { available in
              if available {
                  return Availability.available(message: "Username available")
              }
              else {
                  return Availability.unavailable(message: "Username already taken")
              }
          }
          // `loadingValue` 를 사용해서 서버가 리스폰스를 보내줄 때까지 기다립니다
          .startWith(loadingValue)
    }

// 이제 `Observable<Observable<ValidationResult>>`를 가지고 있기 때문에
// 간단한 `Observable<ValidationResult>` 를 반환하도록 만들어야 합니다.
// 두번째 예제의 `concat` 연산자를 사용할 수 있지만, 우리가 정말로 원하는 것은
// 새로운 username이 주어졌다면 대기 상태의 비동기 작업을 취소하는 것입니다.
// 이게 `switchLatest`가 하는 일입니다.
    .switchLatest()
// 이제 그 결과를 UI에 바인딩해야 합니다.
// 오랜 좋은 친구인 `subscribe(onNext:)`가 해줄 것입니다.
// 여기가 `옵저버블` 체인의 끝입니다.
    .subscribe(onNext: { validity in
        errorLabel.textColor = validationColor(validity)
        errorLabel.text = validity.message
    })
// 여기선 `Disposable` 객체를 만들어낼 것이고
// 이 객체는 모두를 해제하고 대기상태의 비동기 작업들을 취소할 수 있습니다.
// 해제를 수동으로 하는 것은 너무 지겨우니,
// 뷰 컨트롤러가 dealloc될 때 모두를 dispose 시킵시다.
    .disposed(by: disposeBag)
```

이보다 더 간단해질 순 없습니다. 레포지토리에는 [더 많은 예제](../RxExample)들이 준비되어 있으니 마음 편하게 보세요.

예제 중에는 MVVM 패턴에서 Rx를 어떻게 사용해야 하는지에 대해서 보여주는 예제도 포함하고 있으니 관심있으시면 [여기서](../RxExample) 확인가능합니다.
