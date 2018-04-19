시작하기
===============

이 프로젝트는 [ReactiveX.io](http://reactivex.io/)와 일관성을 유지할 예정입니다. 크로스 플랫폼 문서 및 튜토리얼은 RxSwift의 경우에도 유효해야 합니다.

1. [Observables aka Sequences](#observables-aka-sequences)
1. [Disposing](#disposing)
1. [`Observable`이 보장하는 사항들 (Implicit `Observable` guarantees)](#observable이-보장하는-사항들)
1. [Creating your first `Observable` (aka observable sequence)](#creating-your-own-observable-aka-observable-sequence)
1. [Creating an `Observable` that performs work](#creating-an-observable-that-performs-work)
1. [Sharing subscription and `shareReplay` operator](#sharing-subscription-and-sharereplay-operator)
1. [Operators](#operators)
1. [Playgrounds](#playgrounds)
1. [Custom operators](#custom-operators)
1. [Error handling](#error-handling)
1. [Debugging Compile Errors](#debugging-compile-errors)
1. [Debugging](#debugging)
1. [Debugging memory leaks](#debugging-memory-leaks)
1. [Enabling Debug Mode](#enabling-debug-mode)
1. [KVO](#kvo)
1. [UI layer tips](#ui-layer-tips)
1. [Making HTTP requests](#making-http-requests)
1. [RxDataSources](#rxdatasources)
1. [Driver](Traits.md#driver)
1. [Traits: Driver, Single, Maybe, Completable](Traits.md)
1. [Examples](Examples.md)

# Observables aka Sequences

## 기초
옵저버 패턴(`Observable<Element>` 시퀀스)과 보통의 시퀀스(`Sequence`) 사이의 [동등성](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/MathBehindRx.md)을 아는 것은 Rx를 이해하기 위한 가장 중요한 요소 중 하나입니다.

**모든 `Observable` 시퀀스는 그냥 시퀀스이기도 합니다. `Observable`과 Swift의 `Sequence`간의 비교 중 가장 핵심적인 이점은 요소들을 비동기로 받을 수 있다는 점입니다. 이것은 RxSwift의 커널이며, 여기 문서에서는 그 아이디어를 확장하는 방법에 대해 다룰 것입니다.**

* `Observable`(`ObservableType`)은 `Sequence`와 동일합니다.

* `ObservableType.subscribe` 메소드는 `Sequence.makeIterator` 메소드와 동일합니다.

* 옵저버(콜백)는 반환된 이터레이터에서 `next()` 메소드를 호출하는 대신에 시퀀스 요소를 받기 위해 `ObservableType.subscribe` 메소드에 전달돼야 합니다.

시퀀스는 **시각화하기 쉬운** 간단하고 친근한 개념입니다.

인간은 거대한 시각 피질을 가진 생물입니다. 개념을 쉽게 시각화할 수 있다면, 그러한 이유에 대해 아는 것은 어렵지 않습니다.

모든 Rx 연산자의 이벤트 상태 기계를 시퀀스보다 높은 수준의 작업으로 시뮬레이션하는 시도는 우리에게 인지적 부하를 많이 줄 수 있습니다.

만약 우리가 Rx를 쓰지않고 모델 비동기 시스템을 쓴다면 그것은 아마 우리의 코드가 상태 기계와 일시적인 상태들로 가득차서 추상화하기보단 시뮬레이션해야 한다는 의미일 것입니다.

리스트와 시퀀스는 수학자와 프로그래머가 가장 먼저 배우는 개념 중 하나일 것입니다.

여기 숫자들의 시퀀스가 있습니다:

```
--1--2--3--4--5--6--| // 정상적으로 종료됩니다
```

문자로 이루어진 또 다른 시퀀스입니다:

```
--a--b--a--a--a---d---X // 정상적으로 종료하지 못하고 에러가 발생합니다
```

버튼을 누르는 일과 같이 어떤 시퀀스는 한정적인 반면에, 어떤 시퀀스는 무한합니다:

```
---tap-tap-------tap--->
```

이것들은 마블 다이어그램이라고 부릅니다. 더 많은 마블 다이어그램들이 [rxmarbles.com](http://rxmarbles.com/)에 있습니다.

만약 시퀀스 문법을 정규식으로 표현한다면 다음과 같을 것입니다:

next (error | completed)?*

이는 다음과 같은 의미를 가집니다:

* **시퀀스는 0개 이상의 요소들을 가질 수 있습니다.**

* **`error`나 `completed` 이벤트를 받으면, 시퀀스는 더 이상 다른 요소를 만들어낼 수 없습니다.**

Rx의 시퀀스는 푸시 인터페이스로 표현되기도 합니다. (콜백이라고도 하죠)

```swift
enum Event<Element>  {
    case next(Element)      // 시퀀스의 다음 요소
    case error(Swift.Error) // 시퀀스에서 에러가 발생함
    case completed          // 시퀀스가 성공적으로 종료됨
}

class Observable<Element> {
    func subscribe(_ observer: Observer<Element>) -> Disposable
}

protocol ObserverType {
    func on(_ event: Event<Element>)
}
```

**시퀀스가 `completed`나 `error` 이벤트를 내부 리소스들에 보냈을 때 그 계산된 시퀀스 요소는 해제될 것입니다.**

**시퀀스 요소가 만들어지는 것을 취소하고 리소스를 즉시 해제하고 싶다면, 반환된 구독에서 `dispose`를 호출하면 됩니다.**

만약 시퀀스가 정해진 시간에 종료된다면, `dispose`를 호출하지 않거나 `disposed(by: disposeBag)`를 사용하지 않아도 자원 누출이 일어나지 않을 것입니다. 하지만, 그 자원들은 시퀀스가 요소를 만들어내는 일을 끝내거나 에러를 발생할때까지 사용될 것입니다.

만약 시퀀스가 버튼 탭과 같이 알아서 종료되지 않는다면, 할당된 자원들은 `dispose`가 호출되거나 `disposeBag`에 담겨있거나 `takeUntil` 연산자를 사용하는 등의 방법을 쓰지 않는 한 영원히 살아있을 것입니다.

**dispose bag이나 `takeUntil` 연산자를 쓰는 것은 리소스가 확실히 정리되는데에 확실한 방법들입니다. 심지어 시퀀스가 한정된 시간안에 끝나는 경우에도 dispose bag이나 `takeUntil` 연산자를 쓰는 것을 추천합니다.**

왜 `Swift.Error`가 제네릭이 아닌지에 대해 궁금하다면 [여기에서](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/DesignRationale.md#why-error-type-isnt-generic) 그에 대한 내용을 확인하실 수 있습니다.

## Disposing

관찰당하고 있는 시퀀스를 종료할 수 있는 추가적인 방법이 하나 있습니다. 우리가 시퀀스로 할 일을 끝냈고 다가올 요소들을 계산하기 위해 할당된 리소스들을 모두 해제하고 싶을 때, 구독(subscription)에서 `dispose`를 호출하면 됩니다.

여기 `interval` 연산자에 대한 예제입니다.

```
let scheduler = SerialDispatchQueueScheduler(qos: .default)
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
    .subscribe { event in
        print(event)
    }

Thread.sleep(forTimeInterval: 2.0)

subscription.dispose()
```

위의 코드는 다음과 같은 결과를 출력할 것입니다:

```
0
1
2
3
4
5
```

여러분은 수동으로 dispose를 호출하는 것을 원하지 않을 것을 압니다. 이것은 그저 교육용 예제일 뿐입니다. dispose를 수동으로 호출하는 것은 나쁜 코드의 냄새가 납니다. DisposeBag, takeUntil 연산자, 또는 다른 매커니즘과 같이 구독을 dispose하는 더 나은 방법들이 많이 있습니다.

그래서 이 무언가를 출력하는 코드는 `dispose`를 호출한 이후에 종료되나요? 대답은 상황에 따라 다르다 입니다.

* 만약 `scheduler`가 **시리얼 스케쥴러**(예. `MainScheduler`)이고, `dispose`가 **같은 시리얼 스케쥴러에서** 호출됐다면 대답은 아니요 입니다.

* 다른 경우엔 모두 **예** 입니다.

스케쥴러에 대한 내용은 [여기서](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Schedulers.md) 더 확인하실 수 있습니다.

간단하게 말하자면 여러분엔겐 평행하게 일어나고 있는 두 개의 프로세스가 있다고 생각하시면 됩니다.

* 하나는 요소를 만드는 것

* 또 다른 하나는 구독을 해제(dispose)하는 것

"이후에 무언가가 출력되나요?"라는 질문은 이 프로세스들이 다른 스케쥴러에 있다면 의미가 없는 질문이 됩니다.

확실하게 이해하기 위한 몇 가지 예제를 보여드리겠습니다. (`observeOn`은 [여기](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Schedulers.md)에 설명돼있습니다)

아래와 같은 코드가 있다고 가정합니다:

```
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .observeOn(MainScheduler.instance)
            .subscribe { event in
                print(event)
            }

// ....

subscription.dispose() // 메인 쓰레드에서 호출됩니다
```

**`dispose` 호출이 반환되면 아무것도 출력되지 않습니다. 이것은 보장되어있는 일입니다.**

또, 아래와 같은 경우가 있다고 가정합니다:

```
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .observeOn(serialScheduler)
            .subscribe { event in
                print(event)
            }

// ...

subscription.dispose() // 같은 `serialScheduler`에서 실행합니다
```

**`dispose` 호출이 반환되면 아무것도 출력되지 않습니다. 이것도 보장되어있는 일입니다.**

### Dispose Bags

Dispose Bag은 Rx에서 ARC와 비슷한 역할을 합니다.

`DisposeBag`이 해제됐을 때, 추가된 disposables들 모두에게 `dispose`를 호출합니다.

그 자체에는 `dispose` 메소드가 없어서 다른 목적으로 예외적인 호출은 불가능합니다. 즉각적인 정리가 필요하다면 새로운 가방을 만들면 됩니다.

```
  self.disposeBag = DisposeBag()
```

위의 코드는 오래된 참조들을 정리해주고 자원들이 해제되도록 합니다.

만약 예외적인 수동의 해제가 여전히 필요하다면, `CompositeDisposable`을 사용해보세요. **CompositeDisposable은 원하는 기능을 가지고 있을것이지만 `dispose`가 호출됐을 때 새롭게 추가된 disposable도 즉시 해제시켜 버립니다.**

### Take until

dealloc시에 구독을 자동으로 해제하는 또 다른 방법으론 `takeUntil` 연산자를 사용하는 방법이 있습니다.

```
sequence
    .takeUntil(self.rx.deallocated)
    .subscribe {
        print($0)
    }
```

## `Observable`이 보장하는 사항들

모든 시퀀스 생성자들(`Observables`)이 지켜야하는 추가적인 규칙들이 있습니다.

어떤 쓰레드에서 요소를 만들던지 상관없지만 그것들이 요소 하나를 만든 후에 옵저버인 `observer.on(.next(nextElement))` 에 보냈다면, `observer.on` 메소드가 실행을 마칠때까지 다음 요소를 보낼 수 없습니다.

생성자들은 또한 `.completed` 나 `.error` 를 `.next` 이벤트가 끝나지 않았을 때 보내서 종료할 수 없습니다.

요약하자면, 아래의 예제와 같을 것입니다:

```
someObservable
  .subscribe { (e: Event<Element>) in
      print("이벤트 처리 시작")
      // 처리중
      print("이벤트 처리 끝")
  }
```

위의 코드는 아래와 같은 결과를 항상 출력할 것입니다:

```
이벤트 처리 시작
이벤트 처리 끝
이벤트 처리 시작
이벤트 처리 끝
이벤트 처리 시작
이벤트 처리 끝
```

절대 아래와 같이 출력될 수는 없습니다:

```
이벤트 처리 시작
이벤트 처리 시작
이벤트 처리 끝
이벤트 처리 끝
```

## 나만의 `옵저버블` 만들기 (aka 옵저버블 시퀀스)

이 부분은 옵저버블을 이해하는데에 아주 필수적인 부분입니다.

**옵저버블이 만들어지면, 그것은 단들어졌을 뿐이기 때문에 아무일도 하지 않습니다.**

`옵저버블`이 다양한 방법으로 요소를 만들 수 있다는 것은 사실입니다. 그 중 어떤 방법은 부작용을 일으킬 수 있고 또 다른 방법은 마우스 이벤트를 탭하는 것과 같이 이미 실행되고 있는 프로세스에 접근하기도 합니다.

**하지만, 만약 여러분이 그저 `옵저버블`을 반환하는 메소드를 호출했다면, 시퀀스 생성은 일어나지 않을 것이고 부작용도 없을 것입니다. `옵저버블`은 그저 시퀀스가 어떻게 만들어지고 요소 생성에 어떤 파라미터가 필요한지에 대해 적혀있을 뿐입니다. 시퀀스 생성은 `subscribe` 메소드가 호출됐을 때 시작됩니다.**

예를 들어, 비슷한 프로토타입을 가진 메소드가 있다고 해봅시다:

```swift
func searchWikipedia(searchTerm: String) -> Observable<Results> {}
```

```swift
let searchForMe = searchWikipedia("me")

// 아무 리퀘스트도 실행되지 않으며, 어떤 작업도 작동하지 않고, URL 리퀘스트도 발생하지 않습니다

let cancel = searchForMe
  // 여기서 시퀀스 생성이 시작되고, URL 리퀘스트들이 발생합니다
  .subscribe(onNext: { results in
      print(results)
  })

```

자신의 `옵저버블` 시퀀스를 만드는 방법은 아주 많습니다. 가장 쉬운 방법은 아마 `create` 함수를 사용하는 것입니다.

구독을 하면 하나의 요소를 반환하는 시퀀스를 만드는 함수를 작성해봅시다. 이 함수를 `just`라고 부릅니다.

*실제 구현 방법은 아래와 같습니다*

```swift
func myJust<E>(_ element: E) -> Observable<E> {
    return Observable.create { observer in
        observer.on(.next(element))
        observer.on(.completed)
        return Disposables.create()
    }
}

myJust(0)
    .subscribe(onNext: { n in
      print(n)
    })
```

결과는 다음과 같을 것입니다:

```
0
```

나쁘지 않네요. 그러면 `create` 함수는 뭘까요?

`create` 함수는 스위프트 클로저를 사용해서 우리가 쉽게 `subscribe` 메소드를 구현하는 것을 가능하게 해주는 메소드입니다. `subscribe` 메소드처럼 딱 하나 `observer` 만 받아서 disposable을 반환합니다.

이런 방법으로 구현하는 시퀀스는 실제로 동기성을 띕니다. 요소를 만들고 `subscribe` 가 호출되고 disposable을 반환하기 전에 종료됩니다. 왜냐하면 어떤 disposable을 반환하는지 상관없고, 요소를 만드는 작업은 방해할 수 없기 때문입니다.

동기 시퀀스를 만들때, 반환될 보통의 disposable `NopDisposable`의 싱글톤 인스턴스입니다.

배열의 요소를 반환하는 옵저버블을 만들어봅시다.

*아래는 실제 구현 코드입니다*

```swift
func myFrom<E>(_ sequence: [E]) -> Observable<E> {
    return Observable.create { observer in
        for element in sequence {
            observer.on(.next(element))
        }

        observer.on(.completed)
        return Disposables.create()
    }
}

let stringCounter = myFrom(["first", "second"])

print("Started ----")

// 첫 시도
stringCounter
    .subscribe(onNext: { n in
        print(n)
    })

print("----")

// 다시
stringCounter
    .subscribe(onNext: { n in
        print(n)
    })

print("Ended ----")
```

아래와 같이 출력될 것입니다:

```
Started ----
first
second
----
first
second
Ended ----
```

# 

## 작동하는 `옵저버블` 만들기

이제 좀 더 흥미로운 걸 해보겠습니다. 이전 예제에서 사용했던 `interval` 연산자를 만들 것입니다.

*이는 디스패치 큐 스케쥴러의 실제 구현 방식과 동일합니다*

```swift
func myInterval(_ interval: TimeInterval) -> Observable<Int> {
    return Observable.create { observer in
        print("Subscribed")
        let timer = DispatchSource.makeTimerSource(queue: DispatchQueue.global())
        timer.scheduleRepeating(deadline: DispatchTime.now() + interval, interval: interval)

        let cancel = Disposables.create {
            print("Disposed")
            timer.cancel()
        }

        var next = 0
        timer.setEventHandler {
            if cancel.isDisposed {
                return
            }
            observer.on(.next(next))
            next += 1
        }
        timer.resume()

        return cancel
    }
}
```

```swift
let counter = myInterval(0.1)

print("Started ----")

let subscription = counter
    .subscribe(onNext: { n in
        print(n)
    })


Thread.sleep(forTimeInterval: 0.5)

subscription.dispose()

print("Ended ----")
```

위의 코드는 다음과 같은 결과를 출력할 것입니다.
```
Started ----
Subscribed
0
1
2
3
4
Disposed
Ended ----
```

아래와 같이 입력하신다면

```swift
let counter = myInterval(0.1)

print("Started ----")

let subscription1 = counter
    .subscribe(onNext: { n in
        print("First \(n)")
    })
let subscription2 = counter
    .subscribe(onNext: { n in
        print("Second \(n)")
    })

Thread.sleep(forTimeInterval: 0.5)

subscription1.dispose()

Thread.sleep(forTimeInterval: 0.5)

subscription2.dispose()

print("Ended ----")
```

다음과 같이 출력될 것입니다.

```
Started ----
Subscribed
Subscribed
First 0
Second 0
First 1
Second 1
First 2
Second 2
First 3
Second 3
First 4
Second 4
Disposed
Second 5
Second 6
Second 7
Second 8
Second 9
Disposed
Ended ----
```

**구독하고 있는 모든 것은 자신만의 개별 요소들의 시퀀스를 만듭니다. 연산자들은 기본적으로 상태가 없습니다. 상태가 있는 연산자보다 상태가 없는 연산자가 광대하게 많습니다.**

## 구독 공유와 `shareReplay` 연산자

그런데 여러분이 여러개의 관찰자가 하나의 구독에서 다같이 이벤트를 공유하려 한다면 어떨까요?

그러기 위해 정의해야 할 게 두 가지가 있습니다.

* 새로운 구독자가 관찰하고 싶어하기 전에 받은 과거의 요소들은 어떻게 관리할 것인가 (replay latest only, replay all, replay last n)
* 그 공유된 구독은 언제 발생시킬 것인가 (레퍼런스 카운트, 수동 혹은 다른 알고리즘)

보통은 `replay(1).refCount()` 즉, `shareReplay()` 조합을 선택합니다.

```swift
let counter = myInterval(0.1)
    .shareReplay(1)

print("Started ----")

let subscription1 = counter
    .subscribe(onNext: { n in
        print("First \(n)")
    })
let subscription2 = counter
    .subscribe(onNext: { n in
        print("Second \(n)")
    })

Thread.sleep(forTimeInterval: 0.5)

subscription1.dispose()

Thread.sleep(forTimeInterval: 0.5)

subscription2.dispose()

print("Ended ----")
```

이는 다음과 같은 결과를 출력할 것입니다.

```
Started ----
Subscribed
First 0
Second 0
First 1
Second 1
First 2
Second 2
First 3
Second 3
First 4
Second 4
First 5
Second 5
Second 6
Second 7
Second 8
Second 9
Disposed
Ended ----
```

`Subscribed` 와 `Disposed` 이벤트가 딱 하나씩 있을 때 어떻게 되는지 기억하세요.

URL 옵저버블에 대한 행동은 동일합니다.

다음은 HTTP 리퀘스트가 어떻게 Rx로 감싸지는 지에 대한 내용입니다. `interval` 연산자의 패턴과 꽤나 비슷합니다.

```swift
extension Reactive where Base: URLSession {
    public func response(_ request: URLRequest) -> Observable<(Data, HTTPURLResponse)> {
        return Observable.create { observer in
            let task = self.dataTaskWithRequest(request) { (data, response, error) in
                guard let response = response, let data = data else {
                    observer.on(.error(error ?? RxCocoaURLError.Unknown))
                    return
                }

                guard let httpResponse = response as? HTTPURLResponse else {
                    observer.on(.error(RxCocoaURLError.nonHTTPResponse(response: response)))
                    return
                }

                observer.on(.next(data, httpResponse))
                observer.on(.completed)
            }

            task.resume()

            return Disposables.create {
                task.cancel()
            }
        }
    }
}
```

## 연산자

RxSwift에는 수많은 연산자가 존재합니다.

모든 연산자의 마블 다이어그램은 [ReactiveX.io](http://reactivex.io/)에서 확인하실 수 있습니다.

거의 모든 연산자가 [Playgrounds](../Rx.playground)에서 증명이 됩니다.

플레이그라운드를 사용하시려면 `Rx.xcworkspace` 를 열고, `RxSwift-macOS` 스킴을 빌드하신 다음에 `Rx.xcworkspace` 의 트리 뷰에 있는 플레이그라운드를 열면됩니다.

연산자가 필요한데 어떤 것을 써야할 지 모르겠을 때는 [연산자 결정에 도움을 주는 트리](http://reactivex.io/documentation/operators.html#tree)를 사용해보세요.

### 커스텀 연산자

커스텀 연산자를 만드는 방법은 두 가지가 있습니다.

#### 쉬운 방법

모든 내부의 코드들은 극도로 최적화된 버전의 연산자를 사용합니다. 그래서 그것들은 좋은 튜토리얼의 예가 아닙니다. 그것이 표준 연산자를 주로 예를 드는 이유입니다.

운좋게도 연산자를 만드는 쉬운 방법이 존재합니다. 새로운 연산자를 만드는 것은 사실 새로운 옵저버블을 만드는 것이라고 할 수 있습니다. 그리고 전 챕터에서 그 방법을 이미 설명했습니다.

최적화되지 못한 map 연산자는 어떻게 구현할 수 있는지 알아봅시다.

```swift
extension ObservableType {
    func myMap<R>(transform: @escaping (E) -> R) -> Observable<R> {
        return Observable.create { observer in
            let subscription = self.subscribe { e in
                    switch e {
                    case .next(let value):
                        let result = transform(value)
                        observer.on(.next(result))
                    case .error(let error):
                        observer.on(.error(error))
                    case .completed:
                        observer.on(.completed)
                    }
                }

            return subscription
        }
    }
}
```

이제 여러분만의 map을 사용하실 수 있게 되었습니다.

```swift
let subscription = myInterval(0.1)
    .myMap { e in
        return "This is simply \(e)"
    }
    .subscribe(onNext: { n in
        print(n)
    })
```

이 코드는 다음과 같은 결과를 출력할 것입니다:

```
Subscribed
This is simply 0
This is simply 1
This is simply 2
This is simply 3
This is simply 4
This is simply 5
This is simply 6
This is simply 7
This is simply 8
...
```

### 엄청난 변화

커스텀 연산자로도 해결하기 어려운 문제가 나타난다면 어떡할까요? 그럴땐 Rx 세계를 떠나서 `Subject`를 사용하여 실제 세계에서 액션을 취한 후에 결과를 Rx로 돌려보내면 됩니다.

이것은 아주 나쁜 코드의 냄새가 나기 때문에 아래와 같이 고칠 수 있습니다.

```swift
  let magicBeings: Observable<MagicBeing> = summonFromMiddleEarth()

  magicBeings
    .subscribe(onNext: { being in     // Rx 세계를 떠납니다
        self.doSomeStateMagic(being)
    })
    .disposed(by: disposeBag)

  //
  //  다른 세계
  //
  let kitten = globalParty(   // 복잡한 세계에서 무언가를 계산합니다
    being,
    UIApplication.delegate.dataSomething.attendees
  )
  kittens.on(.next(kitten))   // 결과를 Rx로 다시 보냅니다
  //
  // 또 다른 세계
  //

  let kittens = Variable(firstKitten) // Rx 세계로 다시 돌아옵니다

  kittens.asObservable()
    .map { kitten in
      return kitten.purr()
    }
    // ....
```

여러분이 이런 코드를 어딘가에 쓸 때, 누군가는 아래와 같이 사용할 것입니다:

```swift
  kittens
    .subscribe(onNext: { kitten in
      // do something with kitten
    })
    .disposed(by: disposeBag)
```

그러니 따라하지 마십시오.

## 플레이그라운드

만약 여러분이 몇몇 연산자가 어떻게 작동하는지 잘 모르시겠다면, [playgrounds](../Rx.playground)에 모든 연산자와 그것들의 사용 예제까지 준비되어 있습니다.

**플레이그라운드를 사용하시려면 Rx.xcworkspace를 열고 RxSwift-macOS 스킴을 빌드한 후에 Rx.xcworkspace 트리 뷰의 플레이그라운드를 여세요.**

**플레이그라운드 예제들의 결과를 보려면 `Assistant Editor` 를 열면 됩니다. `Assistant Editor` 는 `View > Assistant Editor > Show Assistant Editor` 를 클릭하시면 열 수 있습니다.**

## 에러 핸들링

에러 메커니즘은 두 가지가 있습니다.

### 옵저버블에서의 비동기 에러 핸들링 메커니즘

에러 핸들링은 꽤 간단합니다. 만약 한 시퀀스에서 에러가 발생했다면 모든 독립 시퀀스는 에러를 발생시킬 것입니다. 이것은 보통의 짧은 서킷 로직입니다.

`catch` 연산자를 사용하면 옵저버블의 실패를 복구시킬 수 있습니다. 엄청난 디테일의 특정한 부분을 복구할 수 있는 다양한 오버로드도 있습니다.

또한 `retry` 연산자를 사용하면 에러가 난 시퀀스를 다시 시도하게도 할 수 있습니다.

## 컴파일 에러 디버깅하기

엘레강스한 RxSwift/RxCocoa 코드를 작성할 때, 여러분은 `Observable`의 타입을 추론하기 위해 아마 컴파일러에 엄청나게 의존할 것입니다. 이건 Swift가 대단한 이유 중 하나이지만, 가끔씩 좌절감을 주기도 합니다.

```swift
images = word
    .filter { $0.containsString("important") }
    .flatMap { word in
        return self.api.loadFlickrFeed("karate")
            .catchError { error in
                return just(JSON(1))
            }
      }
```

만약 컴파일러가 위의 코드에 에러가 있다고 보고한다면, 첫번째 반환 타입을 지정해주는 것을 추천드립니다.

```swift
images = word
    .filter { s -> Bool in s.containsString("important") }
    .flatMap { word -> Observable<JSON> in
        return self.api.loadFlickrFeed("karate")
            .catchError { error -> Observable<JSON> in
                return just(JSON(1))
            }
      }
```

만약 작동하지 않는다면, 에러를 찾아낼 때까지 더 많은 타입을 지정하면 됩니다.

```swift
images = word
    .filter { (s: String) -> Bool in s.containsString("important") }
    .flatMap { (word: String) -> Observable<JSON> in
        return self.api.loadFlickrFeed("karate")
            .catchError { (error: Error) -> Observable<JSON> in
                return just(JSON(1))
            }
      }
```

**저는 클로저의 첫번째 반환 타입과 변수를 지정하는 것을 추천드립니다.**

보통 에러를 수정하고 난 후엔 지정해놓은 타입을 삭제하고 코드 정리를 하시면 됩니다.

## 디버깅

디버거만 사용하는 것은 유용하지만 `debug` 연산자를 사용하면 더욱 효율적일 것입니다. `debug` 연산자는 표준 출력의 모든 이벤트를 출력하고 그 이벤트들에 대한 설명도 출력합니다.

아래의 예제를 보면 `debug`는 조사하는 것처럼 작동됩니다.

```swift
let subscription = myInterval(0.1)
    .debug("my probe")
    .map { e in
        return "This is simply \(e)"
    }
    .subscribe(onNext: { n in
        print(n)
    })

Thread.sleepForTimeInterval(0.5)

subscription.dispose()
```

위의 코드는 아래와 같이 출력할 것입니다.

```
[my probe] subscribed
Subscribed
[my probe] -> Event next(Box(0))
This is simply 0
[my probe] -> Event next(Box(1))
This is simply 1
[my probe] -> Event next(Box(2))
This is simply 2
[my probe] -> Event next(Box(3))
This is simply 3
[my probe] -> Event next(Box(4))
This is simply 4
[my probe] dispose
Disposed
```

또한 여러분만의 버전인 `debug` 연산자를 쉽게 만들 수 있습니다.

```swift
extension ObservableType {
    public func myDebug(identifier: String) -> Observable<Self.E> {
        return Observable.create { observer in
            print("subscribed \(identifier)")
            let subscription = self.subscribe { e in
                print("event \(identifier)  \(e)")
                switch e {
                case .next(let value):
                    observer.on(.next(value))

                case .error(let error):
                    observer.on(.error(error))

                case .completed:
                    observer.on(.completed)
                }
            }
            return Disposables.create {
                   print("disposing \(identifier)")
                   subscription.dispose()
            }
        }
    }
 }
 ```

### 디버그 모드 켜기

`RxSwift.Resources`를 사용해서 [메모리 릭 디버깅하기](#debugging-memory-leaks) 또는 [모든 HTTP 리퀘스트들을 자동으로 로깅](#logging-http-traffic)하기를 위해서, 우선 디버그 모드를 켜야합니다.

디버그 모드를 작동시키려면 `TRACE_RESOURCES` 플래그가 RxSwift 타겟 빌드 세팅의 _Other Swift Flags_ 에 추가돼 있어야 합니다.

Cocoapods과 Carthage를 위한 `TRACE_RESOURCES` 플래스 설정법에 대해 더 알고 싶으시다면 [#378](https://github.com/ReactiveX/RxSwift/issues/378)을 보세요.

## 메모리 누수 디버깅하기

Rx의 디버그 모드에선 글로벌 변수인 `Resources.total` 에 할당된 모든 자원을 추적합니다.

만약 여러분이 자원 누수 탐지 로직을 원한다면, 가장 간단한 방법은 `RxSwift.Resources.total` 을 주기적으로 출력해보는 것입니다.

```swift
    /* 아래 함수의 어딘가에 추가하세요
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey : Any]? = nil)
    */
    _ = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
        .subscribe(onNext: { _ in
            print("Resource count \(RxSwift.Resources.total)")
        })
```

메모리 누수를 테스트하기 위한 가장 효과적인 방법은 다음과 같습니다.
* 원하는 화면으로 가서 작동한다
* 뒤로 돌아간다
* 초기 자원 갯수를 관찰한다
* 다시 그 화면으로 돌아가서 같은 작업을 한다
* 뒤로 돌아간다
* 마지막 자원 갯수를 관찰한다

만약 처음과 끝의 자원 갯수가 다르다면, 어디선가 메모리 누수가 나고 있다는 뜻입니다.

네비게이션을 하나가 아닌 둘로 확인하는 것을 추천하는 이유는 첫번째 네비게이션은 자원을 게으르게 불러오도록 강요하기 때문입니다.

## Variables

`Variable` 는 관찰가능한 상태를 표시합니다. `Variable` 은 초기값을 필수로 하기 때문에 값이 없다면 존재할 수 없습니다.

Variable은 [`Subject`](http://reactivex.io/documentation/subject.html)를 감쌉니다. 더 자세하게 말하자면 `BehaviorSubject` 를 감쌉니다. `BehaviorSubject` 와는 다르게, 오직 `value` 인터페이스를 표시하며, 그래서 variable은 절대 에러로 인한 종료가 일어나지 않습니다.

또한 variable을 구독하면 그것의 현재 값을 즉시 알려줍니다.

variable이 해제되면, `.asObservable()` 에서 반환된 옵저버블 시퀀스를 완료할 것입니다.

```swift
let variable = Variable(0)

print("Before first subscription ---")

_ = variable.asObservable()
    .subscribe(onNext: { n in
        print("First \(n)")
    }, onCompleted: {
        print("Completed 1")
    })

print("Before send 1")

variable.value = 1

print("Before second subscription ---")

_ = variable.asObservable()
    .subscribe(onNext: { n in
        print("Second \(n)")
    }, onCompleted: {
        print("Completed 2")
    })

print("Before send 2")

variable.value = 2

print("End ---")
```

위의 코드는 다음과 같은 결과를 출력할 것입니다.

```
Before first subscription ---
First 0
Before send 1
First 1
Before second subscription ---
Second 1
Before send 2
First 2
Second 2
End ---
Completed 1
Completed 2
```

## KVO

KVO는 Objective-C의 메커니즘입니다. 즉, 타입 안정성을 생각하지 않고 만들어졌다는 것을 뜻합니다. 이 프로젝트에선 그러한 문제점을 해결하기 위한 시도를 했습니다.

이 라이브러리가 KVO를 지원하는 것에는 두 가지 방법이 있습니다.

```swift
// KVO
extension Reactive where Base: NSObject {
    public func observe<E>(type: E.Type, _ keyPath: String, options: KeyValueObservingOptions, retainSelf: Bool = true) -> Observable<E?> {}
}

#if !DISABLE_SWIZZLING
// KVO
extension Reactive where Base: NSObject {
    public func observeWeakly<E>(type: E.Type, _ keyPath: String, options: KeyValueObservingOptions) -> Observable<E?> {}
}
#endif
```

예를 들어 `UIView`의 frame을 관찰하는 법을 알아보겠습니다.

**경고: UIKit은 KVO를 따르지 않지만, 아래 코드는 잘 작동합니다.**

```swift
view
  .rx.observe(CGRect.self, "frame")
  .subscribe(onNext: { frame in
    ...
  })
```

또는

```swift
view
  .rx.observeWeakly(CGRect.self, "frame")
  .subscribe(onNext: { frame in
    ...
  })
```

### `rx.observe`

`rx.observe` is more performant because it's just a simple wrapper around KVO mechanism, but it has more limited usage scenarios

* it can be used to observe paths starting from `self` or from ancestors in ownership graph (`retainSelf = false`)
* it can be used to observe paths starting from descendants in ownership graph (`retainSelf = true`)
* the paths have to consist only of `strong` properties, otherwise you are risking crashing the system by not unregistering KVO observer before dealloc.

E.g.

```swift
self.rx.observe(CGRect.self, "view.frame", retainSelf: false)
```

### `rx.observeWeakly`

`rx.observeWeakly` has somewhat slower than `rx.observe` because it has to handle object deallocation in case of weak references.

It can be used in all cases where `rx.observe` can be used and additionally

* because it won't retain observed target, it can be used to observe arbitrary object graph whose ownership relation is unknown
* it can be used to observe `weak` properties

E.g.

```swift
someSuspiciousViewController.rx.observeWeakly(Bool.self, "behavingOk")
```

### Observing structs

KVO is an Objective-C mechanism so it relies heavily on `NSValue`.

**RxCocoa has built in support for KVO observing of `CGRect`, `CGSize` and `CGPoint` structs.**

When observing some other structures it is necessary to extract those structures from `NSValue` manually.

[Here](../RxCocoa/Foundation/KVORepresentable+CoreGraphics.swift) are examples how to extend KVO observing mechanism and `rx.observe*` methods for other structs by implementing `KVORepresentable` protocol.

## UI layer tips

There are certain things that your `Observable`s need to satisfy in the UI layer when binding to UIKit controls.

### Threading

`Observable`s need to send values on `MainScheduler`(UIThread). That's just a normal UIKit/Cocoa requirement.

It is usually a good idea for your APIs to return results on `MainScheduler`. In case you try to bind something to UI from background thread, in **Debug** build RxCocoa will usually throw an exception to inform you of that.

To fix this you need to add `observeOn(MainScheduler.instance)`.

**URLSession extensions don't return result on `MainScheduler` by default.**

### Errors

You can't bind failure to UIKit controls because that is undefined behavior.

If you don't know if `Observable` can fail, you can ensure it can't fail using `catchErrorJustReturn(valueThatIsReturnedWhenErrorHappens)`, **but after an error happens the underlying sequence will still complete**.

If the wanted behavior is for underlying sequence to continue producing elements, some version of `retry` operator is needed.

### Sharing subscription

You usually want to share subscription in the UI layer. You don't want to make separate HTTP calls to bind the same data to multiple UI elements.

Let's say you have something like this:

```swift
let searchResults = searchText
    .throttle(0.3, $.mainScheduler)
    .distinctUntilChanged
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .shareReplay(1)              // <- notice the `shareReplay` operator
```

What you usually want is to share search results once calculated. That is what `shareReplay` means.

**It is usually a good rule of thumb in the UI layer to add `shareReplay` at the end of transformation chain because you really want to share calculated results. You don't want to fire separate HTTP connections when binding `searchResults` to multiple UI elements.**

**Also take a look at `Driver` unit. It is designed to transparently wrap those `shareReply` calls, make sure elements are observed on main UI thread and that no error can be bound to UI.**

## Making HTTP requests

Making http requests is one of the first things people try.

You first need to build `URLRequest` object that represents the work that needs to be done.

Request determines is it a GET request, or a POST request, what is the request body, query parameters ...

This is how you can create a simple GET request

```swift
let req = URLRequest(url: URL(string: "http://en.wikipedia.org/w/api.php?action=parse&page=Pizza&format=json"))
```

If you want to just execute that request outside of composition with other observables, this is what needs to be done.

```swift
let responseJSON = URLSession.shared.rx.json(request: req)

// no requests will be performed up to this point
// `responseJSON` is just a description how to fetch the response


let cancelRequest = responseJSON
    // this will fire the request
    .subscribe(onNext: { json in
        print(json)
    })

Thread.sleep(forTimeInterval: 3.0)

// if you want to cancel request after 3 seconds have passed just call
cancelRequest.dispose()

```

**URLSession extensions don't return result on `MainScheduler` by default.**

In case you want a more low level access to response, you can use:

```swift
URLSession.shared.rx.response(myURLRequest)
    .debug("my request") // this will print out information to console
    .flatMap { (data: NSData, response: URLResponse) -> Observable<String> in
        if let response = response as? HTTPURLResponse {
            if 200 ..< 300 ~= response.statusCode {
                return just(transform(data))
            }
            else {
                return Observable.error(yourNSError)
            }
        }
        else {
            rxFatalError("response = nil")
            return Observable.error(yourNSError)
        }
    }
    .subscribe { event in
        print(event) // if error happened, this will also print out error to console
    }
```
### Logging HTTP traffic

In debug mode RxCocoa will log all HTTP request to console by default. In case you want to change that behavior, please set `Logging.URLRequests` filter.

```swift
// read your own configuration
public struct Logging {
    public typealias LogURLRequest = (URLRequest) -> Bool

    public static var URLRequests: LogURLRequest =  { _ in
    #if DEBUG
        return true
    #else
        return false
    #endif
    }
}
```

## RxDataSources

... is a set of classes that implement fully functional reactive data sources for `UITableView`s and `UICollectionView`s.

RxDataSources are bundled [here](https://github.com/RxSwiftCommunity/RxDataSources).

Fully functional demonstration how to use them is included in the [RxExample](../RxExample) project.
