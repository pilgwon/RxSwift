Unit Tests
==========

## 커스텀 연산자 테스트하기

RxSwift는 모든 연산자 테스트에 `Rx.xcworkspace` 프로젝트 안에 있는 AllTests-* 타겟의 `RxTest`를 사용합니다.

다음은 보통의 `RxSwift` 연산자 유닛 테스트의 예제입니다.

```swift
func testMap_Range() {
  // 테스트 스케쥴러를 초기화합니다.
  // 테스트 스케쥴러는 로컬 시간과 떨어져있는 가상의 시간을 구현합니다.
  // 이것은 시뮬레이션 실행을 최대한 빠르게 가능하도록 만들고
  // 모든 이벤트가 처리됐다고 증명하게 해줍니다.
  let scheduler = TestScheduler(initialClock: 0)

  // 가상의 핫 옵저버블 시퀀스를 만듭니다.
  // 그 시퀀스는 구독한 옵저버가 없을지라도 정해진 타이밍에 이벤트를 발생할 것입니다.
  // (그게 핫 옵저버블의 의미죠)
  // 이 옵저버블 시퀀스는 모든 구독들을 없어지기 전까지 기록합니다. (`subscriptions` 속성)
  let xs = scheduler.createHotObservable([
    next(150, 1),  // 첫번째 인자는 가상의 시간이고, 두번째 인자는 요소의 값입니다
    next(210, 0),
    next(220, 1),
    next(230, 2),
    next(240, 4),
    completed(300) // 끝나는 가상의 시간
  ])

  // `start` 메소드는 기본적으로,
  // * 시뮬레이션을 실행하고 옵저버가 `res`에 의해 참조되는 모든 이벤트를 기록합니다.
  // * 가상의 시간 200에 구독합니다.
  // * 구독을 가상의 시간 1000에 Dispose 합니다.
  let res = scheduler.start { xs.map { $0 * 2 } }

  let correctMessages = [
    next(210, 0 * 2),
    next(220, 1 * 2),
    next(230, 2 * 2),
    next(240, 4 * 2),
    completed(300)
  ]

  let correctSubscriptions = [
    Subscription(200, 300)
  ]

  XCTAssertEqual(res.events, correctMessages)
  XCTAssertEqual(xs.subscriptions, correctSubscriptions)
}
```

## 연산자 구성 테스트하기 (뷰 모델, 컴포넌트)

테스트 연산자 구성이 어떻게 되는지에 대한 예제는 `Rx.xcworkspace` > `RxExample-iOSTests` 타겟안에 존재합니다.

`RxTest` 익스텐션을 정의하는 것은 쉽기 때문에 여러분도 자신만의 테스트를 가독성 좋게 작성할 수 있습니다. `RxExample-iOSTests`에서 제공되는 예제는 그 익스텐션들을 어떻게 작성하는지에 대한 제안이지만, 테스트를 어떻게 작성하는지에 대한 많은 가능성을 포함하고 있기도 합니다.

```swift
// 예상된 이벤트와 테스트 데이터
let (
  usernameEvents,
  passwordEvents,
  repeatedPasswordEvents,
  loginTapEvents,

  expectedValidatedUsernameEvents,
  expectedSignupEnabledEvents
) = (
  scheduler.parseEventsAndTimes("e---u1----u2-----u3-----------------", values: stringValues).first!,
  scheduler.parseEventsAndTimes("e----------------------p1-----------", values: stringValues).first!,
  scheduler.parseEventsAndTimes("e---------------------------p2---p1-", values: stringValues).first!,
  scheduler.parseEventsAndTimes("------------------------------------", values: events).first!,

  scheduler.parseEventsAndTimes("e---v--f--v--f---v--o----------------", values: validations).first!,
  scheduler.parseEventsAndTimes("f--------------------------------t---", values: booleans).first!
)
```

## 통합 테스트

또한 `RxBlocking` 연산자를 사용해서 통합 테스트를 작성할 수 있는 가능성도 있습니다.

`RxBlocking`의 `toBlocking()` 메소드를 사용하면 현재 스레드를 블락할 수 있고 시퀀스가 완료되기를 기다릴 수 있으며, 동시에 결과에 접근할 수 있도록 해줍니다.

여러분의 시퀀스의 결과를 테스트하는 쉬운 방법은 `toArray` 메소드를 사용하는 것입니다. 이는 시퀀스가 성공적으로 완료되거나 에러가 발생했을 경우 시퀀스가 종료되는 `throw`의 경우에 발생된 모든 요소를 배열로 반환합니다.

```swift
let result = try fetchResource(location)
        .toBlocking()
        .toArray()

XCTAssertEqual(result, expectedResult)
```

또 다른 선택지로는 시퀀스를 세밀하게 검사해주는 `materialize` 연산자를 사용하는 것입니다. 이는 `MaterializedSequenceResult`를 반환할 것이고 이것은 시퀀스가 성공적으로 완료됐으면 `.completed`, 에러가 발생해서 시퀀스가 종료되면 `.failed`를 반환합니다.

```swift
let result = try fetchResource(location)
        .toBlocking()
        .materialize()

// 에러로 인해 종료됐을 경우 결과를 테스트하기 위해
switch result {
        case .completed:
            XCTFail("Expected result to complete with error, but result was successful.")
        case .failed(let elements, let error):
            XCTAssertEqual(elements, expectedResult)
            XCTAssertErrorEqual(error, expectedError)
        }

// 완료로 인해 종료됐을 경우 결과를 테스트하기 위해
switch result {
        case .completed(let elements):
            XCTAssertEqual(elements, expectedResult)
        case .failed(_, let error):
            XCTFail("Expected result to complete without error, but received \(error).")
        }
```
