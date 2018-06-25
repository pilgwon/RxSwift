Tips
====

* 시스템 또는 그 시스템의 부분을 순수한 함수들로 이루어지도록 항상 노력하세요. 그 순수한 함수들은 쉽게 테스트할 수 있고 연산자의 행동을 수정하는데에 사용할 수 있습니다.
* Rx를 사용한다면, 먼저 내장된 연산자를 구성하는 것을 시도해보세요.
* 종종 연산자 몇 개를 섞어서 사용한다면, 자신만의 연산자를 만들어보세요.

예를 들어 보겠습니다.

```swift
extension ObservableType where E: MaybeCool {

    @warn_unused_result(message="http://git.io/rxs.uo")
    public func coolElements()
        -> Observable<E> {
          return filter { e -> Bool in
              return e.isCool
          }
    }
}
```

* Rx 연산자들은 가능한한 일반적입니다. 하지만 모델링하기 힘든 케이스가 있기도 합니다. 그러한 경우에는 자신만의 연산자를 만들고 가능하다면 내장된 연산자 하나를 참조하는 것도 좋습니다.

* 항상 연산자를 subscription을 구성하기 위해 사용하세요.

**절대로 구독을 무슨 수를 써서라도 중첩하지 마세요. 그 코드에선 분명 냄새가 날 것입니다.**

```swift
textField.rx.text.subscribe(onNext: { text in
    performURLRequest(text).subscribe(onNext: { result in
        ...
    })
    .addDisposableTo(disposeBag)
})
.addDisposableTo(disposeBag)
```

**다음은 연산자를 사용해서 disposable을 연쇄로 사용하도록 추천되는 방법입니다.**

```swift
textField.rx.text
  .flatMapLatest { text in
    // 이것이 실패하지 않고 메인 스케쥴러에 결과를 반환한다고 가정하세요.
    // 그렇지 않은 경우엔 `catchError`와 `observeOn(MainScheduler.instance)`가 필요할 것입니다.
    return performURLRequest(text)
  }
  ...
  .addDisposableTo(disposeBag) // 가장 많이 쓰이는 disposable 입니다
```
