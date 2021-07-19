![Swift](https://img.shields.io/static/v1?label=&message=Swift&color=E45530&logo=swift&logoColor=FFFFFF)

# [Rx (ReactiveX)](http://reactivex.io/documentation/ko/observable.html)

> 사용 이유: Async한 작업들을 간결하게 처리하기 위해서  
<br>

## Contents
- Observable
- Operators
- Scheduler
- Subject
- Single

### Observable  
with Async & Dispose
```swift
func rxswiftLoadImage(from imageUrl: String) -> Observable<UIImage?> {
        return Observable.create { seal in
            asyncLoadImage(from: imageUrl) { image in
                seal.onNext(image)
                seal.onCompleted()
            }
            return Disposables.create()
        }
    }
```
```swift
rxswiftLoadImage(from: LARGE_IMAGE_URL)
            .observeOn(MainScheduler.instance) // DispatchQueue.main에서 실행하겠다~
            .subscribe({ result in
                switch result {
                case let .next(image):
                    self.imageView.image = image
                    
                case let .error(err):
                    print(err.localizedDescription)
                    
                case .completed:
                    break
                }
            })
            .disposed(by: disposeBag) // 변수로 받아서 disposeBag에 insert하지 않고 바로 연결해서 사용하는 방법
```
<br>

### Operators
#### - Create: 새로운 Observable을 생성
- just  
        새로 생성한 Observable이 특정 항목을 생성해야할 때 사용
        just에 넘겨준 element 그대로 전달
```swift
Observable.just("RxSwift")
            .subscribe(onNext: { str in
                print(str) // RxSwift
            })
            .disposed(by: disposeBag)
```
- from  
        새로 생성한 Observable이 특정 항목을 생성하고, 구독 시점에 호출된 함수 등을 통해 생성된 항목을 리턴해야할 때 사용  
        just와 다르게 array(sequence) 요소를 하나씩 전달해준다
```swift
Observable.from(["This", "is", "RxSwift"])
            .subscribe(onNext: { str in
                print(str)
                // This
                // is
                // RxSwift
            })
            .disposed(by: disposeBag)
```

#### - Transform: Observable이 배출한 항목들을 변환
- map  
        map -> subscribe -> dispose 이런 흐름을 보고 **stream**이라고 한다  
=> **Observable Streams**
```swift
Observable.just("hello")
            .map { "\($0) RxSwift !"}
            .subscribe(onNext: { str in
                print(str) // hello RxSwift !
            })
            .disposed(by: disposeBag)
```
```swift
Observable.from(["apple", "🍎"])
            .map { $0.count }
            .subscribe(onNext: { str in
                print(str)
                // 5
                // 1
            })
            .disposed(by: disposeBag)
```

#### - [More Opeators](http://reactivex.io/documentation/ko/operators.html)  

#### - [Marbles](https://rxmarbles.com/)

#### - Next / Error / Completed

- subscribe(on: (Event<String>) -> Void)  
  - Event의 3가지 타입  
  1. next: 데이터 전달  
  2. error: **완료 X** | 스트림 종료, disposeBag에서 사라짐  
  3. completed: **완료 O** | 스트림 종료, disposeBag에서 사라짐  

  - Observable이 배출하는 항목과 알림을 기반으로 동작  
  - stream에서 operators를 다 사용한 후 최종적으로 데이터를 사용할 때 subscribe  
  - 다른 operator들은 리턴 타입이 stream(observable)인데, subscribe는 리턴 타입이 disposable  
  => disposed 처리 필요  
        
  ```swift
  Observable.from(["Hello", "World", "Hi", "RxSwift"])
            .subscribe { event in
                switch event {
                case .next(let str):
                    print("데이터 전달! \(str)")
                    
                case .error(let error):
                    print("에러! \(error.localizedDescription)")
                    
                case .completed:
                    print("완료!")
                    
                }
            }
            .disposed(by: disposeBag)
  ```

  - 출력결과  
  ```
  데이터 전달! Hello
  데이터 전달! World
  데이터 전달! Hi
  데이터 전달! RxSwift
  완료!
  ```
        
  - switch 귀찮으니까 
  ```swift
  Observable.from(["Hello", "World", "Hi", "RxSwift"])
            .subscribe(onNext: { str in
                print("next: \(str)")
            }, onCompleted: {
                print("complete")
            })
  ```

  - next를 따로 빼도 됨
  ```swift
  func output(_ element: Any) {
        print("next: \(element)")
    }

  Observable.from(["Hello", "World", "Hi", "RxSwift"])
            .subscribe(onNext: output(_:))
            .disposed(by: disposeBag)
  ```
<br>
        
### Scheduler  

1. observeOn(scheduler: ImmediateSchedulerType)  
 - 옵저버가 어느 스케줄러에서 Observable을 관찰할지 명시
 - 선언 위치 상관 O  
 - observable이 사용할 스레드가 어느 시점에서 할당되는지에 따라 그 후에 호출되는 operator가 영향을 받음  

```swift
.observeOn(MainScheduler.instance) // : main thread에서 실행
```
        
```swift
.observeOn(ConcurrentDispatchQueueScheduler(qos: .default)) // concurrent queue에서 실행 (async) 
``` 

<br>

2. subscribeOn(scheduler: ImmediateSchedulerType) 
 - Observable을 구독할 때 사용할 스케줄러를 명시  
 - 선언 위치 상관 X  
 - Observable이 subscribe 될 때부터 스케줄러를 적용하겠다는 뜻  

```swift
.subscribeOn(ConcurrentDispatchQueueScheduler(qos: .default)) 
```  

        
### Side-Effect
- subscribeOn을 사용하거나 do를 사용
```swift
.do(onNext: { image in
                self.imageView.image = image
            })
```  
```swift
.subscribe(onNext: { image in
                self.imageView.image = image
            })
```  

        
### Subject 
 데이터도 넣을 수 있고, subscribe도 가능  
 외부에서 통제가 가능한 Observable
<br>

- AsyncSubject  
  - 끝이 나야 데이터 전달
- BehaviorSubject  
  - Observable인데 스스로 데이터를 발생시킬 수 있음
- PublishSubject  
- ReplaySubject  


### BehaviorRelay  
- Stream 종료 X (complete X, error X . accpet만 됨)  
-->  주로 UI 그릴 때 사용
- 반면, BehaviorSubject는 observable이니까 죽을 수 있음 (complete되거나 error나면 스트림 종료될 수 있기 때문)
- ex
 ```swift
let loginEnable: BehaviorRelay<Bool> = BehaviorRelay(value: false)

Observable.combineLatest(idValid, pwValid, resultSelector: { $0 && $1 })
            .subscribe(onNext: { b in
                self.loginEnable.accept(b)
            })
            .disposed(by: disposeBag)
 ```

## Ref
[RxSwift 4시간에 끝내기 시즌0](http://reactivex.io/documentation)  
https://www.youtube.com/channel/UCsrPur3UrxuwGmT1Jq6tkQw/videos
