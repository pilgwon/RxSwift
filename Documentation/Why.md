## 왜 Rx를 사용할까요?

**Rx는 앱을 선언형으로 개발할 수 있게 해줍니다.**

### 바인딩

```swift
Observable.combineLatest(firstName.rx.text, lastName.rx.text) { $0 + " " + $1 }
    .map { "Greetings, \($0)" }
    .bind(to: greetingLabel.rx.text)
```

Rx는 `UITableView`와 `UICollectionView`에서도 작동합니다.

```swift
viewModel
    .rows
    .bind(to: resultsTableView.rx.items(cellIdentifier: "WikipediaSearchCell", cellType: WikipediaSearchCell.self)) { (_, viewModel, cell) in
        cell.title = viewModel.title
        cell.url = viewModel.url
    }
    .disposed(by: disposeBag)
```

**dispose 메소드가 필요없는 아주 간단한 바인딩이라도 `.disposed(by: disposeBag)`를 항상 사용하는 것을 공식적으로는 추천하는 편입니다.**

### 재시도

실패하지 않는 API가 있다면 매우 훌륭하겠지만, 불행하게도 그렇지 못하는 경우가 대부분입니다. 다음과 같은 API 메소드가 있다고 해봅시다:

```swift
func doSomethingIncredible(forWho: String) throws -> IncredibleThing
```

만약 위의 메소드를 그대로 사용한다면 실패했을 경우에 다시 시도하는 작업이 매우 어려울 것입니다. [지수 백오프](https://en.wikipedia.org/wiki/Exponential_backoff) 모델링의 복잡성에 대해선 말하지 않아도 아실 것입니다. 물론 가능하지만, 그 코드는 여러분이 신경쓰지 않을 무수히 많은 일시적인 상태를 가질 것이고 다시 사용하지도 못 할 것입니다.

이상적으로, 여러분은 재시도하고자 하는 그 부분만 골라서 재시도고, 어떤 연산자에도 적용할 수 있기를 원할 것입니다.

다음은 Rx로 간단히 재시도하는 방법을 나타낸 코드입니다.

```swift
doSomethingIncredible("me")
    .retry(3)
```

또한 여러분은 커스텀 재시도 연산자를 쉽게 만들 수 있습니다.

### 델리게이트

다음과 같은 지루하고 표현적이지 않은 코드 대신에...

```swift
public func scrollViewDidScroll(scrollView: UIScrollView) { [weak self] // 지금 스크롤되고 있는 스크롤 뷰는 어떤 것인지 알 수 있나요?
    self?.leftPositionConstraint.constant = scrollView.contentOffset.x
}
```

이렇게 작성해보세요

```swift
self.resultsTableView
    .rx.contentOffset
    .map { $0.x }
    .bind(to: self.leftPositionConstraint.rx.constant)
```

### KVO

```
`TickTock`은 KVO가 등록되어 있는 동안 계속 할당되어 있습니다. 관찰에서 누수가 일어났을 때, 그때는 다른 객체로 옮겨질수도 있습니다.
```

```objc
-(void)observeValueForKeyPath:(NSString * )keyPath
                     ofObject:(id)object
                       change:(NSDictionary * )change
                      context:(void * )context
```

위의 두 내용 대신에,

[`rx.observe` 와 `rx.observeWeakly`](GettingStarted.md#kvo)를 사용해보세요.

다음과 같이 사용하시면 됩니다.

```swift
view.rx.observe(CGRect.self, "frame")
    .subscribe(onNext: { frame in
        print("Got new frame \(frame)")
    })
    .disposed(by: disposeBag)
```

또는 이렇게 사용하시면 됩니다.

```swift
someSuspiciousViewController
    .rx.observeWeakly(Bool.self, "behavingOk")
    .subscribe(onNext: { behavingOk in
        print("Cats can purr? \(behavingOk)")
    })
    .disposed(by: disposeBag)
```

### 노티피케이션

다음과 같이 작성하는 것 대신에...

```swift
@available(iOS 4.0, *)
public func addObserverForName(name: String?, object obj: AnyObject?, queue: NSOperationQueue?, usingBlock block: (NSNotification) -> Void) -> NSObjectProtocol
```

이렇게 사용해 보세요.

```swift
NotificationCenter.default
    .rx.notification(NSNotification.Name.UITextViewTextDidBeginEditing, object: myTextView)
    .map {  /*데이터로 작업하는 공간*/ }
    ....
```

### 임시 상태

비동기 프로그램을 작성할 떄 임시 상태를 사용하는 것엔 문제가 많습니다. 가장 많이 사용되는 예제로는 검색창의 자동완성이 있습니다.

만약 여러분이 Rx없이 자동 완성 기능을 추가하려면, 가장 첫번째로 마주치는 문제는 `abc`의 `c`가 언제 입력됐는지 알아내는 것입니다. 그리고 기존에 `ab`에 대한 리퀘스트가 대기상태로 있을 것이고, 그 대기상태의 리퀘스트는 취소될 것입니다. 이러한 문제는 해결하기 아주 어려운 문제이고, 대기상태에 있는 리퀘스트를 잡아두기 위해 추가적인 변수를 만들어야 할 것입니다.

다음 문제는 리퀘스트가 실패했을 때, 여러분은 아주 지저분한 재시도 로직을 작성해야 한다는 것입니다. 하지만 괜찮습니다. 몇 개의 필드만 더 추가하면 재시도를 몇 번이나 했는지에 대해서도 잘 알게 될테니까요.

프로그램이 리퀘스트를 서버로 보내기전에 아주 잠시만 기다려준다면 정말 좋을 것 같지 않나요? 게다가 누군가가 아주아주 긴 무언가를 입력하는 과정이 아니라면 서버를 괴롭히고 싶지도 않습니다. 그러면 추가적인 타이머가 필요하겠네요?

또한 그 검색 결과를 화면에 보여줄지 말지 결정해야 하고, 실패해서 재시도 할 때는 어떤 것을 보여줄지도 정해야 합니다.

앞서나온 모든 기능을 추가하고 알맞게 테스트 하는 것은 매우 지루할 것입니다. 아래는 그 모든 로직을 Rx로 작성한 코드입니다.

```swift
searchTextField.rx.text
    .throttle(0.3, scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .subscribe(onNext: { results in
      // bind to ui
    })
    .disposed(by: disposeBag)
```

추가적인 플래그나 필드는 더 이상 필요없습니다. Rx는 임시적이고 지저분한 모든 것을 관리해줍니다.

### Compositional disposal

Let's assume that there is a scenario where you want to display blurred images in a table view. First, the images should be fetched from a URL, then decoded and then blurred.

It would also be nice if that entire process could be canceled if a cell exits the visible table view area since bandwidth and processor time for blurring are expensive.

It would also be nice if we didn't just immediately start to fetch an image once the cell enters the visible area since, if user swipes really fast, there could be a lot of requests fired and canceled.

It would be also nice if we could limit the number of concurrent image operations because, again, blurring images is an expensive operation.

This is how we can do it using Rx:

```swift
// this is a conceptual solution
let imageSubscription = imageURLs
    .throttle(0.2, scheduler: MainScheduler.instance)
    .flatMapLatest { imageURL in
        API.fetchImage(imageURL)
    }
    .observeOn(operationScheduler)
    .map { imageData in
        return decodeAndBlurImage(imageData)
    }
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { blurredImage in
        imageView.image = blurredImage
    })
    .disposed(by: reuseDisposeBag)
```

This code will do all that and, when `imageSubscription` is disposed, it will cancel all dependent async operations and make sure no rogue image is bound to the UI.

### Aggregating network requests

What if you need to fire two requests and aggregate results when they have both finished?

Well, there is of course the `zip` operator

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.subscribe(onNext: { user, friends in
    // bind them to the user interface
})
.disposed(by: disposeBag)
```

So what if those APIs return results on a background thread, and binding has to happen on the main UI thread? There is `observeOn`.

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.observeOn(MainScheduler.instance)
.subscribe(onNext: { user, friends in
    // bind them to the user interface
})
.disposed(by: disposeBag)
```

There are many more practical use cases where Rx really shines.

### State

Languages that allow mutation make it easy to access global state and mutate it. Uncontrolled mutations of shared global state can easily cause [combinatorial explosion](https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing).

But on the other hand, when used in a smart way, imperative languages can enable writing more efficient code closer to hardware.

The usual way to battle combinatorial explosion is to keep state as simple as possible, and use [unidirectional data flows](https://developer.apple.com/videos/play/wwdc2014-229) to model derived data.

This is where Rx really shines.

Rx is that sweet spot between functional and imperative worlds. It enables you to use immutable definitions and pure functions to process snapshots of mutable state in a reliable composable way.

So what are some practical examples?

### Easy integration

What if you need to create your own observable? It's pretty easy. This code is taken from RxCocoa and that's all you need to wrap HTTP requests with `URLSession`

```swift
extension Reactive where Base: URLSession {
    public func response(request: URLRequest) -> Observable<(Data, HTTPURLResponse)> {
        return Observable.create { observer in
            let task = self.base.dataTask(with: request) { (data, response, error) in
            
                guard let response = response, let data = data else {
                    observer.on(.error(error ?? RxCocoaURLError.unknown))
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

            return Disposables.create(with: task.cancel)
        }
    }
}
```

### Benefits

In short, using Rx will make your code:

* Composable <- Because Rx is composition's nickname
* Reusable <- Because it's composable
* Declarative <- Because definitions are immutable and only data changes
* Understandable and concise <- Raising the level of abstraction and removing transient states
* Stable <- Because Rx code is thoroughly unit tested
* Less stateful <- Because you are modeling applications as unidirectional data flows
* Without leaks <- Because resource management is easy

### It's not all or nothing

It is usually a good idea to model as much of your application as possible using Rx.

But what if you don't know all of the operators and whether or not there even exists some operator that models your particular case?

Well, all of the Rx operators are based on math and should be intuitive.

The good news is that about 10-15 operators cover most typical use cases. And that list already includes some of the familiar ones like `map`, `filter`, `zip`, `observeOn`, ...

There is a huge list of [all Rx operators](http://reactivex.io/documentation/operators.html).

For each operator, there is a [marble diagram](http://reactivex.io/documentation/operators/retry.html) that helps to explain how it works.

But what if you need some operator that isn't on that list? Well, you can make your own operator.

What if creating that kind of operator is really hard for some reason, or you have some legacy stateful piece of code that you need to work with? Well, you've got yourself in a mess, but you can [jump out of Rx monads](GettingStarted.md#life-happens) easily, process the data, and return back into it.
