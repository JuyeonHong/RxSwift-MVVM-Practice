![Swift](https://img.shields.io/static/v1?label=&message=Swift&color=E45530&logo=swift&logoColor=FFFFFF)

# [Rx (ReactiveX)](http://reactivex.io/documentation/ko/observable.html)
- Observable
- Operators
- Scheduler
- Subject
- Single

## 사용 이유
Async한 작업들을 간결하게 처리하기 위해서

## 참고할만한 비동기처리 라이브러리
- PromiseKit
- Bolts

## 사용 방법
### Async / Observable / Dispose
```
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
```
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

### Operators
#### - Create: 새로운 Observable을 생성
- just  
        새로 생성한 Observable이 특정 항목을 생성해야할 때 사용
        just에 넘겨준 element 그대로 전달
```
Observable.just("RxSwift")
            .subscribe(onNext: { str in
                print(str) // RxSwift
            })
            .disposed(by: disposeBag)
```
- from  
        새로 생성한 Observable이 특정 항목을 생성하고, 구독 시점에 호출된 함수 등을 통해 생성된 항목을 리턴해야할 때 사용  
        just와 다르게 array(sequence) 요소를 하나씩 전달해준다
```
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
```
Observable.just("hello")
            .map { "\($0) RxSwift !"}
            .subscribe(onNext: { str in
                print(str) // hello RxSwift !
            })
            .disposed(by: disposeBag)
```
```
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

**.subscribe(on: (Event<String>) -> Void)**    
- Event의 3가지 타입  
        1. next: 데이터 전달  
        2. error: **완료 X** | 스트림 종료, disposeBag에서 사라짐  
        3. completed: **완료 O** | 스트림 종료, disposeBag에서 사라짐  
- stream에서 operators를 다 사용한 후 최종적으로 데이터를 사용할 때 subscribe  
- 다른 operator들은 리턴 타입이 stream(observable)인데  
        subscribe는 리턴 타입이 disposable  
        => disposed 처리 필요        

```
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

출력결과  
```
데이터 전달! Hello
데이터 전달! World
데이터 전달! Hi
데이터 전달! RxSwift
완료!
```
        
1. switch 귀찮으니까 
```
Observable.from(["Hello", "World", "Hi", "RxSwift"])
            .subscribe(onNext: { str in
                print("next: \(str)")
            }, onCompleted: {
                print("complete")
            })
```

2. next를 따로 빼도 됨
```
func output(_ element: Any) {
        print("next: \(element)")
    }

Observable.from(["Hello", "World", "Hi", "RxSwift"])
            .subscribe(onNext: output(_:))
            .disposed(by: disposeBag)
```

        
## Scheduler  

1. observeOn(scheduler: ImmediateSchedulerType)  
        - 선언 위치 상관 O  
        - observable이 사용할 스레드가 어느 시점에서 할당되는지에 따라 그 후에 호출되는 operator가 영향을 받음  
        ```
        .observeOn(MainScheduler.instance)
        ```: main thread에서 실행  
        ```
        .observeOn(ConcurrentDispatchQueueScheduler(qos: .default))
        ```: concurrent queue에서 실행 (async)  

2. subscribeOn(scheduler: ImmediateSchedulerType) 
        - 선언 위치 상관 X  
        - Observable이 subscribe 될 때부터 스케줄러를 적용하겠다는 뜻  
        ```
        .subscribeOn(ConcurrentDispatchQueueScheduler(qos: .default)) 
        ```  
        
        
        
## Ref
http://reactivex.io/documentation  
https://www.youtube.com/channel/UCsrPur3UrxuwGmT1Jq6tkQw/videos
