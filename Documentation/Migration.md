# RxSwift 1.9에서 2.0으로 마이그레이션 하는 방법

마이그레이션은 쉬워야 합니다. 겉모습만 변경되었기 때문에 모든 기능들이 다 그대로 존재합니다.

* 모든 `>- `를 `.`로 변경하세요.
* 모든 `variable`을 `shareReplay(1)`로 변경하세요.
* 모든 `catch`를 `catchErrorJustReturn`으로 변경하세요.
* 모든 `returnElement`를 `Observable.just`로 변경하세요.
* 모든 `failWith`를 `Observable.error`로 변경하세요.
* 모든 `never`를 `Observable.never`로 변경하세요.
* 모든 `empty`를 `Observable.empty`로 변경하세요.
* `>-`가 `.`로 바뀌었기 때문에, free function들이 메소드가 되었고, 이제 `>- switchLatest`, `>- distinctUntilChanged`대신에 `.switchLatest()`, `.distinctUntilChanged()`를 사용하시면 됩니다.
* free function에서 익스텐션으로 옮긴 경우엔 `concat([a, b, c])`, `merge(sequences)`대신에 `[a, b, c].concat()`, `.merge()`를 사용하시면 됩니다.
* 비슷하게, `>- disposeBag.addDisposable` 대신에  `subscribe { n in ... }.disposed(by: disposeBag)`라고 써야합니다.
* `Variable`의 `next`가 `value` 세터가 되었습니다.
* `UITableView`나 `UICollectionView`를 사용하고 싶으시면, 다음과 같이 작성하면 됩니다.

```swift
viewModel.rows
    .bindTo(resultsTableView.rx_itemsWithCellIdentifier("WikipediaSearchCell", cellType: WikipediaSearchCell.self)) { (_, viewModel, cell) in
        cell.viewModel = viewModel
    }
    .disposed(by: disposeBag)
```

RxSwift 2.0의 컨셉에 대해 궁금한 점이 있다면, [예제 앱](../RxExample)이나 플레이그라운드를 확인하세요.
