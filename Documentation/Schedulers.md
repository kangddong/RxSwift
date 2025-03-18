# 스케줄러

1. [직렬 vs 동시 스케줄러](#직렬-vs-동시-스케줄러)
1. [사용자 정의 스케줄러](#사용자-정의-스케줄러)
1. [내장 스케줄러](#내장-스케줄러)

스케줄러는 작업을 수행하기 위한 메커니즘을 추상화합니다.

작업을 수행하기 위한 다양한 메커니즘에는 현재 스레드, 디스패치 대기열, 작업 대기열, 새 스레드, 스레드 풀 및 실행 루프가 포함됩니다.

스케줄러와 함께 작업하는 두 개의 주요 연산자가 있습니다. `observeOn`과 `subscribeOn`입니다.

다른 스케줄러에서 작업을 수행하려면 `observeOn(scheduler)` 연산자를 사용하세요.

당신은 보통 `subscribeOn`보다 `observeOn`을 훨씬 더 자주 사용할 것입니다.

`observeOn`이 명시적으로 지정되지 않은 경우, 생성된 스레드/스케줄러 요소에 대해 작업이 수행됩니다.

`observeOn` 연산자를 사용하는 예:

```swift
sequence1
  .observeOn(backgroundScheduler)
  .map { n in
      print("백그라운드 스케줄러에서 수행됩니다.")
  }
  .observeOn(MainScheduler.instance)
  .map { n in
      print("메인 스케줄러에서 수행됩니다.")
  }
```

시퀀스 생성(`subcribe` 메서드)을 시작하고 특정 스케줄러에서 dispose을 호출하려면 `subscribeOn(scheduler)`을 사용하십시오.

`subscribeOn`이 명시적으로 지정되지 않은 경우,  `subscribe`클로저(`Observable.create`에 전달된 클로저)는 `subscribe(onNext:)` 또는 `subscribe`가 호출되는 동일한 스레드/스케줄러에서 호출됩니다.

`subscribeOn`이 명시적으로 지정되지 않은 경우, `dipose` 메서드는 폐기를 시작한 동일한 스레드/스케줄러에서 호출됩니다.

간단히 말해서, 명시적인 스케줄러가 선택되지 않은 경우, 해당 메서드는 현재 스레드/스케줄러에서 호출됩니다.

## 직렬 vs 동시 스케줄러

스케줄러는 정말 무엇이든 될 수 있고, 시퀀스를 변환하는 모든 연산자는 추가적인 [암시적 보증](GettingStarted.md#implicit-observable-guarantees)을 보존해야 하기 때문에, 어떤 종류의 스케줄러를 만드는 것이 중요합니다.

스케줄러가 동시인 경우, Rx의 `observeOn` 및 `subscribeOn` 운영자는 모든 것이 완벽하게 작동하는지 확인합니다.

Rx가 직렬임을 증명할 수 있는 스케줄러를 사용하면 추가 최적화를 수행할 수 있습니다.

지금까지는 디스패치 대기열 스케줄러에 대한 최적화만 수행합니다.

직렬 디스패치 대기열 스케줄러의 경우, `observeOn`은 간단한 `dispatch_async` 호출에 최적화되어 있습니다.

## 사용자 정의 스케줄러

현재 스케줄러 외에도 자신만의 스케줄러를 작성할 수 있습니다.

누가 즉시 작업을 수행해야 하는지 설명하고 싶다면 `ImmediateScheduler` 프로토콜을 구현하여 자신만의 스케줄러를 만들 수 있습니다.

```swift
public protocol ImmediateScheduler {
    func schedule<StateType>(state: StateType, action: (/*ImmediateScheduler,*/ StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

시간 기반 작업을 지원하는 새로운 스케줄러를 생성하려면 `Scheduler` 프로토콜을 구현해야 합니다:

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

스케줄러에 주기적 스케줄링 기능만 있는 경우 `PeriodicScheduler` 프로토콜을 구현하여 Rx에 알릴 수 있습니다:

```swift
public protocol PeriodicScheduler : Scheduler {
    func schedulePeriodic<StateType>(state: StateType, startAfter: TimeInterval, period: TimeInterval, action: (StateType) -> StateType) -> RxResult<Disposable>
}
```

스케줄러가 `PeriodicScheduling` 기능을 지원하지 않는 경우, Rx는 주기적 스케줄링을 투명하게 모방하여 실행합니다.

## 내장 스케줄러

Rx는 모든 유형의 스케줄러를 사용할 수 있지만 스케줄러가 직렬이라는 증거가 있는 경우 몇 가지 추가 최적화를 수행할 수도 있습니다.

현재 지원되는 스케줄러는 다음과 같습니다:

### CurrentThreadScheduler (직렬 스케줄러)

현재 스레드에 대한 작업 단위를 예약합니다.
이것은 요소를 생성하는 연산자를 위한 기본 스케줄러입니다.

이 스케줄러는 때때로 "트램폴린 스케줄러"라고도 불린다.

`CurrentThreadScheduler.instance.schedule(state) { }` 가 어떤 스레드에서 처음으로 호출되면, 예약된 작업이 즉시 실행되고 모든 재귀적으로 예약된 작업이 일시적으로 대기열에 추가되는 숨겨진 대기열이 생성됩니다.

호출 스택의 일부 상위 프레임이 이미 `CurrentThreadScheduler.instance.schedule(state) { }` 를 실행 중인 경우, 예약된 작업은 현재 실행 중인 작업과 이전에 대기열에 있는 모든 작업의 실행이 완료되면 대기열에 추가되고 실행됩니다.

### MainScheduler (직렬 스케줄러)

`MainThread`에서 수행해야 하는 추상 작업. 메인 스레드에서 `schedule` 메서드가 호출되는 경우 스케줄링 없이 즉시 작업을 수행합니다.

이 스케줄러는 일반적으로 UI 작업을 수행하는 데 사용됩니다.

### SerialDispatchQueueScheduler (직렬 스케줄러)

특정 `dispatch_queue_t`에서 수행해야 하는 작업을 추상화합니다. 동시 디스패치 큐가 전달되더라도 직렬 디스패치 큐로 변환되도록 합니다.

직렬 스케줄러는 `observeOn`에 대한 특정 최적화를 가능하게 합니다.

메인 스케줄러는 `SerialDispatchQueueScheduler`의 인스턴스입니다.

### ConcurrentDispatchQueueScheduler (동시 스케줄러)

특정 `dispatch_queue_t`에서 수행해야 하는 작업을 추상화합니다. 직렬 디스패치 대기열을 전달할 수도 있습니다. 문제가 발생하지 않아야 합니다.

이 스케줄러는 백그라운드에서 일부 작업을 수행해야 할 때 적합합니다.

### OperationQueueScheduler (동시 스케줄러)

특정 `NSOperationQueue`에서 수행해야 하는 작업을 요약합니다.

이 스케줄러는 백그라운드에서 수행해야 하는 더 큰 작업이 있고 `maxConcurrentOperationCount`를 사용하여 동시 처리를 미세 조정하려는 경우에 적합합니다.
