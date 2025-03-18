Traits (이전의 Units)
=====

이 문서는 traits가 무엇인지, 왜 유용한 개념인지, 그리고 어떻게 사용하고 생성하는지 설명합니다.

* [일반](#일반)
  * [이유](#이유)
  * [작동방식](#작동방식)
* [RxSwift traits](#rxswift-traits)
  * [Single](#single)
    * [Single 생성하기](#single-생성하기)
  * [Completable](#completable)
    * [Completable 생성하기](#completable-생성하기)
  * [Maybe](#maybe)
    * [Maybe 생성하기](#maybe-생성하기)
* [RxCocoa traits](#rxcocoa-traits)
  * [Driver](#driver)
    * [왜 Driver라는 이름인가](#왜-driver라는-이름인가)
    * [실용적인 사용 예제](#실용적인-사용-예제)
  * [Signal](#signal)
  * [ControlProperty / ControlEvent](#controlproperty--controlevent)


## 일반
### 이유

Swift는 강력한 타입 시스템을 가지고 있어 애플리케이션의 정확성과 안정성을 향상시키고, Rx를 더 직관적이고 명확하게 사용할 수 있게 합니다.

Traits는 인터페이스 경계 간에 Observable 시퀀스의 속성을 전달하고 보장하는 데 도움이 되며, 맥락적 의미와 문법적 편의성을 제공하고, 어떤 맥락에서든 사용될 수 있는 원시 Observable보다 더 구체적인 사용 사례를 대상으로 합니다. **이런 이유로, Traits는 완전히 선택적입니다. 모든 핵심 RxSwift/RxCocoa API가 지원하므로 프로그램 어디에서나 원시 Observable 시퀀스를 자유롭게 사용할 수 있습니다.**

***참고:** 이 문서에 설명된 일부 Trait(예: `Driver`)은 [RxCocoa](https://github.com/ReactiveX/RxSwift/tree/main/RxCocoa) 프로젝트에만 특화되어 있고, 일부는 일반 [RxSwift](https://github.com/ReactiveX/RxSwift) 프로젝트의 일부입니다. 그러나 필요한 경우 동일한 원칙을 다른 Rx 구현에서도 쉽게 구현할 수 있습니다. 특별한 비공개 API 마법이 필요하지 않습니다.*

### 작동방식

Traits는 단순히 단일 읽기 전용 Observable 시퀀스 속성을 가진 래퍼 구조체입니다.

```swift
struct Single<Element> {
    let source: Observable<Element>
}

struct Driver<Element> {
    let source: Observable<Element>
}
...
```

Observable 시퀀스를 위한 일종의 빌더 패턴 구현으로 생각할 수 있습니다. Trait이 구축되면, `.asObservable()`을 호출하여 다시 일반 Observable 시퀀스로 변환할 수 있습니다.

---

## RxSwift traits

### Single

Single은 Observable의 변형으로, 일련의 요소를 방출하는 대신 항상 _단일 요소_ 또는 _오류_ 를 방출합니다.

- 정확히 하나의 요소 또는 오류를 방출합니다.
- 부작용을 공유하지 않습니다.

Single을 사용하는 일반적인 사례는 응답 또는 오류만 반환할 수 있는 HTTP 요청을 수행하는 경우이지만, Single은 무한한 요소 스트림이 아닌 단일 요소만 신경 쓰는 모든 경우를 모델링하는 데 사용할 수 있습니다.

#### Single 생성하기
Single을 생성하는 것은 Observable을 생성하는 것과 유사합니다.
간단한 예는 다음과 같습니다:

```swift
func getRepo(_ repo: String) -> Single<[String: Any]> {
    return Single<[String: Any]>.create { single in
        let task = URLSession.shared.dataTask(with: URL(string: "https://api.github.com/repos/\(repo)")!) { data, _, error in
            if let error = error {
                single(.failure(error))
                return
            }

            guard let data = data,
                  let json = try? JSONSerialization.jsonObject(with: data, options: .mutableLeaves),
                  let result = json as? [String: Any] else {
                single(.failure(DataError.cantParseJSON))
                return
            }

            single(.success(result))
        }

        task.resume()

        return Disposables.create { task.cancel() }
    }
}
```

이후 다음과 같은 방식으로 사용할 수 있습니다:

```swift
getRepo("ReactiveX/RxSwift")
    .subscribe { event in
        switch event {
            case .success(let json):
                print("JSON: ", json)
            case .failure(let error):
                print("Error: ", error)
        }
    }
    .disposed(by: disposeBag)
```

또는 다음과 같이 `subscribe(onSuccess:onError:)`를 사용할 수 있습니다:
```swift
getRepo("ReactiveX/RxSwift")
    .subscribe(onSuccess: { json in
                   print("JSON: ", json)
               },
               onError: { error in
                   print("Error: ", error)
               })
    .disposed(by: disposeBag)
```

구독은 Swift `Result` 열거형을 사용하며, 열겨형에는 Single 타입의 요소를 포함하는 `.success` 또는 오류를 포함하는 `.failure`일 수 있습니다. 첫 번째 이벤트 이후에는 더 이상의 이벤트가 방출되지 않습니다.

원시 Observable 시퀀스에서 `.asSingle()`을 사용하여 Single로 변환하는 것도 가능합니다.

### Completable

Completable은 Observable의 변형으로, _완료_ 또는 _오류_ 방출만 할 수 있습니다. 요소를 방출하지 않음을 보장합니다.

* 0개의 요소를 방출합니다.
* 완료 이벤트 또는 오류를 방출합니다.
* 부작용을 공유하지 않습니다.

Completable의 유용한 사용 사례는 작업이 완료되었다는 사실만 신경 쓰고, 그 완료로 인한 요소는 신경 쓰지 않는 모든 경우를 모델링하는 것입니다.
요소를 방출할 수 없는 Observable<Void>를 사용하는 것과 비교할 수 있습니다.

#### Completable 생성하기
Completable을 생성하는 것은 Observable을 생성하는 것과 유사합니다. 간단한 예는 다음과 같습니다:

```swift
func cacheLocally() -> Completable {
    return Completable.create { completable in
       // Store some data locally
       ...
       ...

       guard success else {
           completable(.error(CacheError.failedCaching))
           return Disposables.create {}
       }

       completable(.completed)
       return Disposables.create {}
    }
}
```

이후 다음과 같은 방식으로 사용할 수 있습니다:
```swift
cacheLocally()
    .subscribe { completable in
        switch completable {
            case .completed:
                print("오류 없이 완료됨")
            case .error(let error):
                print("오류와 함께 완료됨: \(error.localizedDescription)")
        }
    }
    .disposed(by: disposeBag)
```

또는 다음과 같이 `subscribe(onCompleted:onError:)`를 사용할 수 있습니다:
```swift
cacheLocally()
    .subscribe(onCompleted: {
                   print("오류 없이 완료됨")
               },
               onError: { error in
                   print("오류와 함께 완료됨: \(error.localizedDescription)")
               })
    .disposed(by: disposeBag)
```

구독은 `.completed` - 작업이 오류 없이 완료되었음을 나타내거나, `.error`일 수 있는 `CompletableEvent` 열거형을 제공합니다. 첫 번째 이벤트 이후에는 더 이상의 이벤트가 방출되지 않습니다.

### Maybe
Maybe는 Observable의 변형으로, Single과 Completable 사이에 위치합니다. 단일 요소를 방출하거나, 요소를 방출하지 않고 완료하거나, 오류를 방출할 수 있습니다.

**참고:** 이 세 가지 이벤트 중 어느 것이든 Maybe를 종료시킵니다. 즉, 완료된 Maybe는 요소를 방출할 수 없으며, 요소를 방출한 Maybe는 완료 이벤트를 보낼 수 없습니다.

- 완료 이벤트, 단일 요소 또는 오류 중 하나를 방출합니다.
- 부작용을 공유하지 않습니다.

Maybe는 요소를 방출 **할 수도** 있지만, 반드시 요소를 방출 **할 필요는 없는** 모든 작업을 모델링하는 데 사용할 수 있습니다.

#### Maybe 생성하기
Maybe를 생성하는 것은 Observable을 생성하는 것과 유사합니다. 간단한 예는 다음과 같습니다:

```swift
func generateString() -> Maybe<String> {
    return Maybe<String>.create { maybe in
        maybe(.success("RxSwift"))

        // OR

        maybe(.completed)

        // OR

        maybe(.error(error))

        return Disposables.create {}
    }
}
```

이후 다음과 같은 방식으로 사용할 수 있습니다:
```swift
generateString()
    .subscribe { maybe in
        switch maybe {
            case .success(let element):
                print("요소 \(element)와 함께 완료됨")
            case .completed:
                print("요소 없이 완료됨")
            case .error(let error):
                print("Completed with an error \(error.localizedDescription)")
        }
    }
    .disposed(by: disposeBag)
```

Or by using `subscribe(onSuccess:onError:onCompleted:)` as follows:

```swift
generateString()
    .subscribe(onSuccess: { element in
                   print("요소 \(element)와 함께 완료됨")
               },
               onError: { error in
                   print("오류 \(error.localizedDescription)와 함께 완료됨")
               },
               onCompleted: {
                   print("요소 없이 완료됨")
               })
    .disposed(by: disposeBag)
```

원시 Observable 시퀀스에서 `.asMaybe()`를 사용하여 `Maybe`로 변환하는 것도 가능합니다.

---

## RxCocoa traits

### Driver

이것은 가장 정교한 trait입니다. UI 레이어에서 리액티브 코드를 작성하는 직관적인 방법을 제공하거나, 애플리케이션을 _운전(Drive)_ 하는 데이터 스트림을 모델링하려는 모든 경우에 사용할 수 있습니다.

- 오류가 발생하지 않습니다.
- 관찰은 메인 스케줄러에서 발생합니다.
- 부작용을 공유합니다 (`share(replay: 1, scope: .whileConnected)`).

#### 왜 Driver라는 이름인가

이 trait의 의도된 사용 사례는 애플리케이션을 운전하는 시퀀스를 모델링하는 것입니다.

예:
* CoreData 모델에서 UI 운전.
* 다른 UI 요소의 값을 사용하여 UI 운전(바인딩).
...

일반 운영 체제 드라이버와 마찬가지로, 시퀀스에 오류가 발생하면 애플리케이션이 사용자 입력에 반응하지 않게 됩니다.

또한 UI 요소와 애플리케이션 로직은 일반적으로 스레드 안전하지 않기 때문에, 이러한 요소들이 메인 스레드에서 관찰되는 것이 매우 중요합니다.

또한 `Driver`는 부작용을 공유하는 Observable 시퀀스를 구축합니다.

예:

#### 실용적인 사용 예제

이것은 전형적인 초보자 예제입니다.

```swift
let results = query.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
    }

results
    .map { "\($0.count)" }
    .bind(to: resultCount.rx.text)
    .disposed(by: disposeBag)

results
    .bind(to: resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```

이 코드의 의도된 동작은 다음과 같습니다:
* 사용자 입력을 조절(Throttle)합니다.
* 서버에 연결하고 사용자 결과 목록을 가져옵니다(쿼리당 한 번).
* 결과를 두 개의 UI 요소에 바인딩합니다: 결과 테이블 뷰와 결과 수를 표시하는 레이블.

그렇다면 이 코드의 문제점은 무엇일까요?:
* `fetchAutoCompleteItems` Observable 시퀀스에 오류가 발생하면(연결 실패 또는 파싱 오류), 이 오류로 인해 모든 바인딩이 해제되고 UI는 더 이상 새로운 쿼리에 반응하지 않습니다.
* `fetchAutoCompleteItems`가 백그라운드 스레드에서 결과를 반환하면, 결과는 백그라운드 스레드에서 UI 요소에 바인딩되어 예측할 수 없는 크래시를 일으킬 수 있습니다.
* 결과는 두 개의 UI 요소에 바인딩되어 있으므로, 각 사용자 쿼리마다 각 UI 요소에 대해 두 개의 HTTP 요청이 발생하게 됩니다. 이는 의도된 동작이 아닙니다.

코드의 더 적절한 버전은 다음과 같을 것입니다:

```swift
let results = query.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .observeOn(MainScheduler.instance)  // 결과는 MainScheduler에서 반환됨
            .catchErrorJustReturn([])           // 최악의 경우, 오류를 처리함
    }
    .share(replay: 1)                           // HTTP 요청은 공유되고 결과는 모든 UI 요소에 재생됨

results
    .map { "\($0.count)" }
    .bind(to: resultCount.rx.text)
    .disposed(by: disposeBag)

results
    .bind(to: resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```

대규모 시스템에서 이러한 모든 요구 사항이 제대로 처리되는지 확인하는 것은 어려울 수 있지만, 컴파일러와 trait를 사용하여 이러한 요구 사항이 충족되었음을 증명하는 더 간단한 방법이 있습니다.

다음 코드는 거의 동일하게 보입니다:

```swift
let results = query.rx.text.asDriver()        // 이는 일반 시퀀스를 `Driver` 시퀀스로 변환합니다.
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .asDriver(onErrorJustReturn: [])  // 빌더는 오류 발생 시 반환할 내용에 대한 정보만 필요합니다.
    }

results
    .map { "\($0.count)" }
    .drive(resultCount.rx.text)               // `bind(to:)` 대신 `drive` 메서드를 사용할 수 있다면,
    .disposed(by: disposeBag)              // 이는 컴파일러가 모든 속성이 충족되었음을 증명했음을 의미합니다.

results
    .drive(resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```

그렇다면 여기서 무슨 일이 일어나고 있을까요?

첫 번째 `asDriver` 메서드는 `ControlProperty` trait을 `Driver` trait으로 변환합니다.

```swift
query.rx.text.asDriver()
```

특별히 해야 할 일은 없었습니다. `Driver`는 `ControlProperty` trait의 모든 속성과 더 많은 것을 가지고 있습니다. 기본 Observable 시퀀스는 단지 `Driver` trait으로 래핑될 뿐입니다.

두 번째 변경 사항은 다음과 같습니다:

```swift
.asDriver(onErrorJustReturn: [])
```

모든 Observable 시퀀스는 다음 3가지 속성을 만족하는 한 `Driver` trait으로 변환될 수 있습니다:

- 오류가 발생하지 않습니다.
- 메인 스케줄러에서 관찰합니다.
- 부작용 공유 (`share(replay: 1, scope: .whileConnected)`).

그렇다면 이러한 속성이 충족되는지 어떻게 확인할 수 있을까요? 일반 Rx 연산자를 사용하면 됩니다. `asDriver(onErrorJustReturn: [])`는 다음 코드와 동일합니다.

```swift
let safeSequence = xs
  .observeOn(MainScheduler.instance)        // 메인 스케줄러에서 이벤트 관찰
  .catchErrorJustReturn(onErrorJustReturn)  // 오류가 발생하지 않음
  .share(replay: 1, scope: .whileConnected) // 부작용 공유

return Driver(raw: safeSequence)            // 래핑
```

마지막 부분은 `bind(to:)` 대신 `drive`를 사용하는 것입니다.

`drive`는 `Driver` trait에서만 정의됩니다. 이는 코드 어딘가에서 `drive`를 보게 된다면, 해당 Observable 시퀀스는 절대 오류가 발생하지 않으며 메인 스레드에서 관찰한다는 것을 의미하므로, UI 요소에 바인딩하는 것이 안전합니다.

그러나 이론적으로는 누군가가 여전히 `ObservableType`이나 다른 인터페이스에서 작동하도록 `drive` 메서드를 정의할 수 있으므로, 더욱 안전하고 완전한 증명을 위해서는 UI 요소에 바인딩하기 전에 `let results: Driver<[Results]> = ...` 와 같은 임시 정의를 생성하는 것이 필요할 것입니다. 그러나 이것이 현실적인 시나리오인지 여부는 독자의 판단에 맡기겠습니다.

### Signal

`Signal`은 한 가지 차이점을 제외하고는 `Driver`와 유사합니다. 구독 시 최신 이벤트를 **재생하지 않지만**, 구독자는 여전히 시퀀스의 계산 리소스를 공유합니다.

이는 명령형 이벤트를 애플리케이션의 일부로서 리액티브 방식으로 모델링하기 위한 빌더 패턴으로 간주될 수 있습니다.

`Signal`의 특징:

* 오류가 발생하지 않습니다.
* 메인 스케줄러에서 이벤트를 전달합니다.
* 계산 리소스를 공유합니다 (`share(scope: .whileConnected)`).
* 구독 시 요소를 재생하지 않습니다.

## ControlProperty / ControlEvent

### ControlProperty

UI 요소의 속성을 나타내는 `Observable`/`ObservableType`용 Trait입니다.

값 시퀀스는 초기 컨트롤 값과 사용자가 시작한 값 변경만 나타냅니다. 프로그래밍 방식 값 변경은 보고되지 않습니다.

속성은 다음과 같습니다:

- 실패하지 않습니다
- `share(replay: 1)` 동작
    - 상태 유지, 구독(subscribe 호출) 시 마지막 요소가 생성되었다면 즉시 재생됩니다
- 컨트롤이 해제될 때 시퀀스를 `완료`합니다
- 오류가 발생하지 않습니다
- `MainScheduler.instance`에서 이벤트를 전달합니다

`ControlProperty`의 구현은 이벤트 시퀀스가 메인 스케줄러에서 구독되도록 보장합니다(`subscribeOn(ConcurrentMainScheduler.instance)` 동작).

#### 실용적인 사용 예제

`UISearchBar+Rx`와 `UISegmentedControl+Rx`에서 매우 좋은 실용적인 예를 찾을 수 있습니다:

```swift
extension Reactive where Base: UISearchBar {
    /// `text` 속성에 대한 리액티브 래퍼입니다.
    public var value: ControlProperty<String?> {
        let source: Observable<String?> = Observable.deferred { [weak searchBar = self.base as UISearchBar] () -> Observable<String?> in
            let text = searchBar?.text
            
            return (searchBar?.rx.delegate.methodInvoked(#selector(UISearchBarDelegate.searchBar(_:textDidChange:))) ?? Observable.empty())
                    .map { a in
                        return a[1] as? String
                    }
                    .startWith(text)
        }

        let bindingObserver = Binder(self.base) { (searchBar, text: String?) in
            searchBar.text = text
        }
        
        return ControlProperty(values: source, valueSink: bindingObserver)
    }
}
```

```swift
extension Reactive where Base: UISegmentedControl {
    /// `selectedSegmentIndex` 속성에 대한 리액티브 래퍼입니다.
    public var selectedSegmentIndex: ControlProperty<Int> {
        value
    }
    
    /// `selectedSegmentIndex` 속성에 대한 리액티브 래퍼입니다.
    public var value: ControlProperty<Int> {
        return UIControl.rx.value(
            self.base,
            getter: { segmentedControl in
                segmentedControl.selectedSegmentIndex
            }, setter: { segmentedControl, value in
                segmentedControl.selectedSegmentIndex = value
            }
        )
    }
}
```

### ControlEvent

UI 요소의 이벤트를 나타내는 `Observable`/`ObservableType`용 Trait입니다.

속성은 다음과 같습니다:

- 실패하지 않습니다
- 구독 시 초기 값을 보내지 않습니다
- 컨트롤이 해제될 때 시퀀스를 `완료`합니다
- 오류가 발생하지 않습니다
- `MainScheduler.instance`에서 이벤트를 전달합니다

`ControlEvent`의 구현은 이벤트 시퀀스가 메인 스케줄러에서 구독되도록 보장합니다(`subscribeOn(ConcurrentMainScheduler.instance)` 동작).

#### 실용적인 사용 예제

다음은 이를 사용할 수 있는 전형적인 사례 예입니다:

```swift
public extension Reactive where Base: UIViewController {
    
    /// `UIViewController:viewDidLoad:` 메시지 `viewDidLoad`에 대한 리액티브 래퍼입니다.
    public var viewDidLoad: ControlEvent<Void> {
        let source = self.methodInvoked(#selector(Base.viewDidLoad)).map { _ in }
        return ControlEvent(events: source)
    }
}
```

그리고 `UICollectionView+Rx`에서 다음과 같은 방식으로 발견할 수 있습니다:

```swift

extension Reactive where Base: UICollectionView {
    
    /// `delegate` 메시지 `collectionView:didSelectItemAtIndexPath:`에 대한 리액티브 래퍼입니다.
    public var itemSelected: ControlEvent<IndexPath> {
        let source = delegate.methodInvoked(#selector(UICollectionViewDelegate.collectionView(_:didSelectItemAt:)))
            .map { a in
                return a[1] as! IndexPath
            }
        
        return ControlEvent(events: source)
    }
}
```
