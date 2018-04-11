시작하기
===============

이 프로젝트는 [ReactiveX.io](http://reactivex.io/)와 일관성을 유지할 예정입니다. 크로스 플랫폼 문서 및 튜토리얼은 RxSwift의 경우에도 유효해야 합니다.

1. [Observables aka Sequences](#observables-aka-sequences)
1. [Disposing](#disposing)
1. [Implicit `Observable` guarantees](#implicit-observable-guarantees)
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

## Implicit `Observable` guarantees
### 내장된 `Observable`에 대해 보증된 사항들

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

## Creating your own `Observable` (aka observable sequence)

There is one crucial thing to understand about observables.

**When an observable is created, it doesn't perform any work simply because it has been created.**

It is true that `Observable` can generate elements in many ways. Some of them cause side effects and some of them tap into existing running processes like tapping into mouse events, etc.

**However, if you just call a method that returns an `Observable`, no sequence generation is performed and there are no side effects. `Observable` just defines how the sequence is generated and what parameters are used for element generation. Sequence generation starts when `subscribe` method is called.**

E.g. Let's say you have a method with similar prototype:

```swift
func searchWikipedia(searchTerm: String) -> Observable<Results> {}
```

```swift
let searchForMe = searchWikipedia("me")

// no requests are performed, no work is being done, no URL requests were fired

let cancel = searchForMe
  // sequence generation starts now, URL requests are fired
  .subscribe(onNext: { results in
      print(results)
  })

```

There are a lot of ways to create your own `Observable` sequence. The easiest way is probably to use the `create` function.

Let's write a function that creates a sequence which returns one element upon subscription. That function is called 'just'.

*This is the actual implementation*

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

This will print:

```
0
```

Not bad. So what is the `create` function?

It's just a convenience method that enables you to easily implement `subscribe` method using Swift closures. Like `subscribe` method it takes one argument, `observer`, and returns disposable.

Sequence implemented this way is actually synchronous. It will generate elements and terminate before `subscribe` call returns disposable representing subscription. Because of that it doesn't really matter what disposable it returns, process of generating elements can't be interrupted.

When generating synchronous sequences, the usual disposable to return is singleton instance of `NopDisposable`.

Lets now create an observable that returns elements from an array.

*This is the actual implementation*

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

// first time
stringCounter
    .subscribe(onNext: { n in
        print(n)
    })

print("----")

// again
stringCounter
    .subscribe(onNext: { n in
        print(n)
    })

print("Ended ----")
```

This will print:

```
Started ----
first
second
----
first
second
Ended ----
```

## Creating an `Observable` that performs work

Ok, now something more interesting. Let's create that `interval` operator that was used in previous examples.

*This is equivalent of actual implementation for dispatch queue schedulers*

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

This will print
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

What if you would write

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

This would print:

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

**Every subscriber upon subscription usually generates it's own separate sequence of elements. Operators are stateless by default. There are vastly more stateless operators than stateful ones.**

## Sharing subscription and `shareReplay` operator

But what if you want that multiple observers share events (elements) from only one subscription?

There are two things that need to be defined.

* How to handle past elements that have been received before the new subscriber was interested in observing them (replay latest only, replay all, replay last n)
* How to decide when to fire that shared subscription (refCount, manual or some other algorithm)

The usual choice is a combination of `replay(1).refCount()` aka `shareReplay()`.

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

This will print

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

Notice how now there is only one `Subscribed` and `Disposed` event.

Behavior for URL observables is equivalent.

This is how HTTP requests are wrapped in Rx. It's pretty much the same pattern like the `interval` operator.

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

## Operators

There are numerous operators implemented in RxSwift.

Marble diagrams for all operators can be found on [ReactiveX.io](http://reactivex.io/)

Almost all operators are demonstrated in [Playgrounds](../Rx.playground).

To use playgrounds please open `Rx.xcworkspace`, build `RxSwift-macOS` scheme and then open playgrounds in `Rx.xcworkspace` tree view.

In case you need an operator, and don't know how to find it there a [decision tree of operators](http://reactivex.io/documentation/operators.html#tree).

### Custom operators

There are two ways how you can create custom operators.

#### Easy way

All of the internal code uses highly optimized versions of operators, so they aren't the best tutorial material. That's why it's highly encouraged to use standard operators.

Fortunately there is an easier way to create operators. Creating new operators is actually all about creating observables, and previous chapter already describes how to do that.

Lets see how an unoptimized map operator can be implemented.

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

So now you can use your own map:

```swift
let subscription = myInterval(0.1)
    .myMap { e in
        return "This is simply \(e)"
    }
    .subscribe(onNext: { n in
        print(n)
    })
```

This will print:

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

### Life happens

So what if it's just too hard to solve some cases with custom operators? You can exit the Rx monad, perform actions in imperative world, and then tunnel results to Rx again using `Subject`s.

This isn't something that should be practiced often, and is a bad code smell, but you can do it.

```swift
  let magicBeings: Observable<MagicBeing> = summonFromMiddleEarth()

  magicBeings
    .subscribe(onNext: { being in     // exit the Rx monad
        self.doSomeStateMagic(being)
    })
    .disposed(by: disposeBag)

  //
  //  Mess
  //
  let kitten = globalParty(   // calculate something in messy world
    being,
    UIApplication.delegate.dataSomething.attendees
  )
  kittens.on(.next(kitten))   // send result back to rx
  //
  // Another mess
  //

  let kittens = Variable(firstKitten) // again back in Rx monad

  kittens.asObservable()
    .map { kitten in
      return kitten.purr()
    }
    // ....
```

Every time you do this, somebody will probably write this code somewhere:

```swift
  kittens
    .subscribe(onNext: { kitten in
      // do something with kitten
    })
    .disposed(by: disposeBag)
```

So please try not to do this.

## Playgrounds

If you are unsure how exactly some of the operators work, [playgrounds](../Rx.playground) contain almost all of the operators already prepared with small examples that illustrate their behavior.

**To use playgrounds please open Rx.xcworkspace, build RxSwift-macOS scheme and then open playgrounds in Rx.xcworkspace tree view.**

**To view the results of the examples in the playgrounds, please open the `Assistant Editor`. You can open `Assistant Editor` by clicking on `View > Assistant Editor > Show Assistant Editor`**

## Error handling

There are two error mechanisms.

### Asynchronous error handling mechanism in observables

Error handling is pretty straightforward. If one sequence terminates with error, then all of the dependent sequences will terminate with error. It's usual short circuit logic.

You can recover from failure of observable by using `catch` operator. There are various overloads that enable you to specify recovery in great detail.

There is also `retry` operator that enables retries in case of errored sequence.

## Debugging Compile Errors

When writing elegant RxSwift/RxCocoa code, you are probably relying heavily on compiler to deduce types of `Observable`s. This is one of the reasons why Swift is awesome, but it can also be frustrating sometimes.

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

If compiler reports that there is an error somewhere in this expression, I would suggest first annotating return types.

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

If that doesn't work, you can continue adding more type annotations until you've localized the error.

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

**I would suggest first annotating return types and arguments of closures.**

Usually after you have fixed the error, you can remove the type annotations to clean up your code again.

## Debugging

Using debugger alone is useful, but usually using `debug` operator will be more efficient. `debug` operator will print out all events to standard output and you can add also label those events.

`debug` acts like a probe. Here is an example of using it:

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

will print

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

You can also easily create your version of the `debug` operator.

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

### Enabling Debug Mode
In order to [Debug memory leaks using `RxSwift.Resources`](#debugging-memory-leaks) or [Log all HTTP requests automatically](#logging-http-traffic), you have to enable Debug Mode.

In order to enable debug mode, a `TRACE_RESOURCES` flag must be added to the RxSwift target build settings, under _Other Swift Flags_.

For further discussion and instructions on how to set the `TRACE_RESOURCES` flag for Cocoapods & Carthage, see [#378](https://github.com/ReactiveX/RxSwift/issues/378)

## Debugging memory leaks

In debug mode Rx tracks all allocated resources in a global variable `Resources.total`.

In case you want to have some resource leak detection logic, the simplest method is just printing out `RxSwift.Resources.total` periodically to output.

```swift
    /* add somewhere in
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey : Any]? = nil)
    */
    _ = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
        .subscribe(onNext: { _ in
            print("Resource count \(RxSwift.Resources.total)")
        })
```

Most efficient way to test for memory leaks is:
* navigate to your screen and use it
* navigate back
* observe initial resource count
* navigate second time to your screen and use it
* navigate back
* observe final resource count

In case there is a difference in resource count between initial and final resource counts, there might be a memory
leak somewhere.

The reason why 2 navigations are suggested is because first navigation forces loading of lazy resources.

## Variables

`Variable`s represent some observable state. `Variable` without containing value can't exist because initializer requires initial value.

Variable wraps a [`Subject`](http://reactivex.io/documentation/subject.html). More specifically it is a `BehaviorSubject`.  Unlike `BehaviorSubject`, it only exposes `value` interface, so variable can never terminate with error.

It will also broadcast its current value immediately on subscription.

After variable is deallocated, it will complete the observable sequence returned from `.asObservable()`.

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

will print

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

KVO is an Objective-C mechanism. That means that it wasn't built with type safety in mind. This project tries to solve some of the problems.

There are two built in ways this library supports KVO.

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

Example how to observe frame of `UIView`.

**WARNING: UIKit isn't KVO compliant, but this will work.**

```swift
view
  .rx.observe(CGRect.self, "frame")
  .subscribe(onNext: { frame in
    ...
  })
```

or

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
