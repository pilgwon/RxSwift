Schedulers
==========

1. [Serial vs Concurrent Schedulers](#serial-vs-concurrent-schedulers)
1. [Custom schedulers](#custom-schedulers)
1. [Builtin schedulers](#builtin-schedulers)

스케줄러는 작업을 위한 메커니즘을 추상화 합니다.

각각의 작업을 위한 메커니즘에는 현재 스레드, 디스패치 큐, 오퍼레이션 큐, 새로운 스레드, 스레드 풀이 포함되어 있습니다.

`observeOn` 과 `subscribeOn`은 스케줄러로 작동되는 대표적인 두 연산자입니다.

만약 다른 스케줄러에서 작업을 실행하고 싶다면 `observeOn(scheduler)` 연산자를 사용하면 됩니다.

아마도 여러분은 `subscribeOn` 보다는 `observeOn`를 더 자주 사용하시게 될 것입니다.

`observeOn` 연산자가 명쾌하게 지정하지 않아도, 작업은 해당 요소들이 생성된 스레드/스케줄러에서 실행될 것입니다.

다음은 `observeOn` 연산자를 사용하는 방법에 대한 예제입니다.

```
sequence1
  .observeOn(backgroundScheduler)
  .map { n in
      print("이건 백그라운드 스케줄러에서 실행될 것입니다.")
  }
  .observeOn(MainScheduler.instance)
  .map { n in
      print("이건 메인 스케줄러에서 실행될 것입니다.")
  }
```

만약 시퀀스 생성(`subscribe` 메소드)을 시작하고 특정 스케줄러에서 dispose하고 싶다면, `subscribeOn(scheduler)` 연산자를 사용하면 됩니다.

`subscribeOn`이 명쾌하게 알려주지 않더라도, `subscribe` 클로저(`Observable.create`에 넘겨진 클로저)는 `subscribe(onNext:)` 이나 `subscribe`가 호출된 곳과 동일한 스레드나 스케줄러에서 호출될 것입니다.

`subscribeOn`이 명쾌하게 알려주지 않을경우에도, `dispose`메소드는 초기화했던 스레드/스케줄러에서 호출될 것입니다.

요약하자면, 직접적으로 스케줄러가 선택되지 않아도, 위의 메소드들은 현재 스레드나 스케줄러에서 호출됩니다.

# Serial vs Concurrent Schedulers
스케줄러는 정말로 어떤 것이든 될 수 있고, 시퀀스를 변형하는 모든 연산자들은 추가적인 [암시적인 보증](GettingStarted.md#implicit-observable-guarantees)을 보관해야 하기 때문에, 어떤 종류의 스케줄러를 만들지는 아주 중요한 문제입니다.

스케줄러가 컨커런트할때는, Rx의 `observeOn` 과 `subscribeOn` 연산자는 모든것이 완벽하게 작동되도록 만들 것입니다.

만약 Rx가 순차적이라고 증명할 수 있는 스케줄러를 사용한다면, 그것은 추가적인 최적화가 가능하다는 뜻입니다.

그러한 최적화는 아직까지는 디스패치 큐 스케줄러를 위해서만 실행됩니다.

시리얼 디스패치 큐 스케줄러의 경우, `observeOn`은 그냥 간단한 `dispatch_async`호출로 최적화됩니다.

# Custom schedulers
지금까지 설명한 스케줄러에 뿐만아니라, 여러분 자신만의 스케줄러를 작성할 수 있습니다.

만약 작업 실행이 즉시되는 스케줄러를 원한다면, `ImmediateScheduler`프로토콜을 이용해서 커스텀 스케줄러를 만들면 됩니다.

```swift
public protocol ImmediateScheduler {
    func schedule<StateType>(state: StateType, action: (/*ImmediateScheduler,*/ StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

만약 시간 기반 연산자를 지원하는 새로운 스케줄러를 만들고 싶다면, `Scheduler`프로토콜을 구현하면 됩니다.

```swift
public protocol Scheduler: ImmediateScheduler {
    associatedtype TimeInterval
    associatedtype Time

    var now : Time {
        get
    }

    func scheduleRelative<StateType>(state: StateType, dueTime: TimeInterval, action: (StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

주기적인 스케쥴링을 해주는 스케줄러의 경우엔, `PeriodicScheduler`프로토콜로 Rx를 구현하면 됩니다.

```swift
public protocol PeriodicScheduler : Scheduler {
    func schedulePeriodic<StateType>(state: StateType, startAfter: TimeInterval, period: TimeInterval, action: (StateType) -> StateType) -> RxResult<Disposable>
}
```

만약 스케줄러가 `PeriodicScheduling`을 지원하지 않는다면, Rx가 주기적인 스케쥴링을 투명하게 실행할 것입니다.

# Builtin schedulers
Rx는 모든 종류의 스케줄러를 쓸 수 있습니다. 만약 스케줄러가 시리얼한 것이 증명되면 추가적인 최적화도 적용이 가능합니다.

다음은 현재 지원하는 스케줄러들입니다.

## CurrentThreadScheduler (Serial scheduler)

현재 스레드에 있는 작업의 단위들을 스케쥴해줍니다.
CurrentThreadScheduler는 요소를 생성하는 연산자의 기본으로 적용되는 스케줄러입니다.

이 스케줄러는 가끔씩 "트램펄린 스케줄러(trampoline scheduler)" 라고도 불립니다.

만약 `CurrentThreadScheduler.instance.schedule(state) { }`를 어떤 스레드에서 처음으로 호출했다면, 그 예정된 행동은 즉시 실행될 것이고 모든 재귀적 예정된 액션들이 임시로 저장되는 숨겨진 큐가 생성될 것입니다.

만약 콜 스택의 몇몇 부모 프레임이 이미 `CurrentThreadScheduler.instance.schedule(state) { }`를 실행중이라면, 예정된 액션은 저장되고 현재 실행중인 액션과 모든 전에 저장되었던 액션이 실행 종료되고 나서 실행될 것입니다.

## MainScheduler (Serial scheduler)

`MainThread`에서 실행되어야 하는 추상적인 작업에서 사용합니다. `schedule`메소드가 메인 스레드에서 호출된 경우에, MainScheduler는 스케줄링 없이 액션을 실행할 것입니다.

이 스케줄러는 보통 UI 작업에서 쓰입니다.

## SerialDispatchQueueScheduler (Serial scheduler)

특정 `dispatch_queue_t`에서 실행되어야 하는 추상적인 작업에서 사용합니다. 컨커런트 디스패치 큐에 전달된 경우에도 시리얼 디스패치 큐로 변환됩니다.

시리얼 스케줄러는 `observeOn`를 위한 특정 최적화를 가능하게 해줍니다.

메인 스케줄러는 `SerialDispatchQueueScheduler`의 인스턴스 중 하나입니다.

## ConcurrentDispatchQueueScheduler (Concurrent scheduler)

특정 `dispatch_queue_t`에서 실행되어야 하는 추상적인 작업에서 사용합니다. 시리얼 디스패치 큐에도 보낼 수 있으며 아무 문제도 일으키지 않을 것입니다.

이 스케줄러는 어떤 작업이 백그라운드에서 실행되어야 할 때에 적합합니다.

## OperationQueueScheduler (Concurrent scheduler)

특정 `NSOperationQueue`에서 실행되어야 하는 추상적인 작업에서 사용합니다.

이 스케줄러는 어떤 큰 덩어리의 작업이 있고 이 작업이 백그라운드에서 실행되어야 하며 여러분이 `maxConcurrentOperationCount`를 이용해서 컨커런트 처리과정을 미세 조정하고 싶을 때에 적합합니다.
