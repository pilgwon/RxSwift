Warnings
========

### <a name="unused-disposable"></a>Unused disposable (unused-disposable)

아래 내용은 `Disposable`을 반환하는 `subscribe*`, `bind*` 그리고 `drive*`와 같은 함수에서 유효합니다.

다음과 같은 경우에 여러분은 경고를 받게 될 것입니다.

```Swift
let xs: Observable<E> ....

xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
```

`subscribe` 함수는 계산을 취소하고 리소스를 해제하는데에 사용되는 구독 `Disposable`을 반환합니다. 하지만, 그것을 사용하지 않는다면(그리고 해제하지 않는다면), 에러를 일으킬 것입니다.

이렇게 자주 있을 법한 호출을 없애기 위해 가장 좋은 방법은 `DisposeBag`을 사용하는 것인데, 그 방법으로는 `.disposed(by: disposeBag)`을 함수 체인 끝에 붙이거나 그 bag에 직접적으로 disposable을 추가하는 것입니다.

```Swift
let xs: Observable<E> ....
let disposeBag = DisposeBag()

xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
  .disposed(by: disposeBag) // <--- 기억하세요 `.disposed(by:)`
```

`disposeBag`이 해제되면, 그 안에 있던 disposable들도 자동으로 해제됩니다.

예측할 수 있는 방법으로 `xs`가 `Completed` 또는 `Error` 메세지를 통해 종료되면, 구독 `Disposable`을 처리하지 않으면 자원을 누수하지 않습니다. 하지만, 이 경우에도 구독 disposable에 disposeBag을 사용하는 것은 여전히 좋은 방법입니다. disposeBag은 요소 계산이 항상 예상되는 시점에서 종료된다는 것을 보증하고 여러분의 코드를 간결하고 미래가 보증되게 만들어줍니다. 왜냐하면 자원들은 `xs`의 구현방식이 변경되더라도 적절히 해제됩니다.

구독과 자원이 어떤 객체의 수명에 묶여있는지 확인하는 또 다른 방법은 `takeUntil` 연산자를 사용하는 것입니다.

```Swift
let xs: Observable<E> ....
let someObject: NSObject  ...

_ = xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .takeUntil(someObject.deallocated) // <-- 기억하세요 `takeUntil` 연산자
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
```

만약 구독 `Disposable`이 행동하는 것을 무시하기를 원한다면, 다음과 같이 컴파일러의 경고를 침묵시키면 됩니다.

```Swift
let xs: Observable<E> ....

_ = xs // <-- _를 기억하세요
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
```

### <a name="unused-observable"></a>Unused observable sequence (unused-observable)

다음과 같이 코드를 작성한다면 경고를 받게 될 것입니다.

```Swift
let xs: Observable<E> ....

xs
  .filter { ... }
  .map { ... }
```

이 코드는 `xs` 시퀀스를 filter하고 map시킨 옵저버블 시퀀스를 정의합니다. 하지만 결과값은 무시합니다.

이 코드는 옵저버블 시퀀스를 정의하고 무시하기 때문에, 실제론 아무일도 안합니다.

여러분의 의도는 아마도 옵저버블 시퀀스 정의를 저장해두고 나중에 사용하려고 했을 것입니다...

```Swift
let xs: Observable<E> ....

let ys = xs // <--- 이름을 `ys`로 정의합니다.
  .filter { ... }
  .map { ... }
```

... 또는 그 정의를 기반해서 계산을 시작하기 위해서일 것입니다.

```Swift
let xs: Observable<E> ....
let disposeBag = DisposeBag()

xs
  .filter { ... }
  .map { ... }
  .subscribe(onNext: { nextElement in  // <-- `subscribe*` 메소드를 기억하세요
    // 요소를 사용하세요
    print(nextElement)
  })
  .disposed(by: disposeBag)
```
