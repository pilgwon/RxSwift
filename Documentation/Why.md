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

### 구성 요소 배치

블러 이미지를 테이블 뷰에 보여주려고 한다는 시나리오를 가정해봅시다. 먼저, 이미지는 URL로 부터 가져올 것이고, 디코딩되고, 블러 작업을 거칠 것입니다.

블러 작업에드는 작업 비용은 꽤나 크기때문에 이미지가 그려질 셀이 화면에서 나가면 모든 처리 과정이 중단될 수 있으면 더욱 좋을 것입니다.

그리고 셀이 화면에 보여졌다고 바로 이미지를 불러오는 작업을 하지 않아야 할 것입니다. 사용자가 매우 빠르게 스와이프하고 있을 수도 있는데 그럴 경우엔 아주 많은 리퀘스트들이 발생했다가 취소되는 현상이 일어날 것이기 때문입니다.

게다가 동시에 불러올 수 있는 이미지의 숫자도 제한할 수 있다면 좋을 것입니다. 다시 한 번 말하자면, 이미지 블러 작업은 매우 큰 비용의 작업이기 때문입니다.

위의 내용을 Rx로 만든다면 다음과 같습니다:

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

위의 코드는 모든 기능을 포함하고 있는데다가 `imageSubscription`이 dispose되면 다른 모든 비동기 연산들이 한꺼번에 취소되기 때문에 이상한 이미지가 UI에 바운딩되는 경우도 없을 것입니다.

### 네트워크 리퀘스트 합치기

만약 여러분이 두 개의 리퀘스트를 발생했고 두 개가 모두 끝났을 때 결과를 합쳐서 받기를 원하면 어떻게 하시나요?

Rx에는 `zip`이라는 연산자가 당연히 존재합니다.

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.subscribe(onNext: { user, friends in
    // UI에 바인딩합니다
})
.disposed(by: disposeBag)
```

API 결과는 백그라운드 쓰레드에서 받고 바인딩하는 것은 메인 UI 쓰레드에서 하려면 어떻게 해야 할까요? 그래서 `observeOn` 연산자가 존재합니다.

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.observeOn(MainScheduler.instance)
.subscribe(onNext: { user, friends in
    // UI에 바인딩 합니다
})
.disposed(by: disposeBag)
```

Rx에는 더 많은 빛나는 실전 예제들이 있습니다.

### 상태

변화(mutation)를 허락하는 언어는 전역 상태에 접근하기 쉽고 변경할 수 있게 해줍니다. 공유된 전역 상태의 컨트롤되지 않는 변화는 쉽게 [combinatorial explosion](https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing)을 일으킬 수 있습니다.

하지만 그 반대로, 똑똑하게 사용한다면 언어로 더욱 효율적인 코드를 작성할 수 있고 하드웨어에 더 가까이 다가갈 수 있습니다.

combinatorial explosion에 대항하는 보통의 방법은 상태를 최대한 간단하게 유지하고, [단방향 데이터 흐름](https://developer.apple.com/videos/play/wwdc2014-229)을 사용해서 모델에서 유래된 데이터를 다루는 것입니다.

여기가 바로 Rx가 정말로 빛이 나는 부분입니다.

Rx는 함수형과 명령형 세계 사이의 그 달콤한 부분 어딘가일 것입니다. Rx는 여러분이 변하지 않는 정의를 사용하고 변화하는 상태의 스냅샷을 처리할 수 있도록 해줍니다.

실제 예제는 어떤것이 있을까요?

### 쉬운 통합

만약 여러분만의 옵저버블을 만들어야하면 어떨까요? 꽤나 쉽습니다. 다음 코드는 RxCocoa에서 가져왔고 여러분이 해야하는 일은 그저 `URLSession`으로 HTTP 리퀘스트를 감싸는 일입니다.

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

### 이득

짧게 말하면, Rx를 쓰는 것은 여러분의 코드를,

* 쉽게 구성할 수 있습니다. Rx는 구성(composition)의 별명이기 때문입니다.
* 재사용이 가능합니다. 쉽게 구성할 수 있기 때문입니다.
* 선언형입니다. 정의 자체는 바꿀 수 없고 데이터만 바뀌기 때문입니다.
* 이해가 쉽고 간결합니다. 추상화의 단계를 올리고 임시적인 상태를 없애줍니다.
* 안정적입니다. Rx의 코드는 완전히 유닛 테스트되었기 때문입니다.
* 상태에 적게 영향을 받습니다. 어플리케이션은 단방향 데이터 흐름으로 모델링되기 때문입니다.
* 누수가 일어나지 않습니다. 자원 관리가 매우 쉽기 때문입니다.

### 모두 알지 않는다면 아무것도 아닙니다

여러분의 어플리케이션의 모델을 Rx를 사용할 수 있게 만드는 것은 보통 좋은 아이디어입니다.

하지만 여러분이 모든 연산자에 대해 모르고 특별한 경우에 써야하는 연산자가 있다는 것을 모른다면 어떨까요?

모든 Rx 연산자는 수학에 기반해 있어서 직관적입니다.

좋은 소식은 10 ~ 15개의 연산자가 대부분의 경우를 커버한다는 것입니다. 그리고 그 목록은 우리에게 친근한 `map`, `filter`, `zip`, `observeOn`을 포함하고 있습니다.

여기 [모든 Rx 연산자](http://reactivex.io/documentation/operators.html)에 대한 목록이 있습니다.

각각의 연산자는 그것만의 [마블 다이어그램](http://reactivex.io/documentation/operators/retry.html)이 있으며, 그 다이어그램은 연산자가 어떻게 작동하는지 이해하기 쉽게 만들어줍니다.

하지만 만약 이 목록에 없는 연산자가 필요하면 어떻게 할까요? 그러면 여러분만의 연산자를 만드시면 됩니다.

아니면 몇 가지 이유로 그러한 커스텀 연산자를 만들기 어려운 경우거나, 상태를 건드리는 레거시 코드 조각들이 여전히 많이 남아 있는 경우신가요? 그런 경우라면 복잡하다고 생각하실수도 있겠지만, 쉽게 [Rx 모나드를 잠시 나갔다가](GettingStarted.md#엄청난-변화), 데이터를 처리하고, 그리고 돌아오면 됩니다.
