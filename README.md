![Swift](https://img.shields.io/static/v1?label=&message=Swift&color=E45530&logo=swift&logoColor=FFFFFF)

# [Rx (ReactiveX)](http://reactivex.io/documentation/ko/observable.html)

> 사용 이유: Async한 작업들을 간결하게 처리하기 위해서  

비동기적으로 생성되는 데이터를 completion같은 클로저로 전달하지 않고 리턴값으로 전달하기 위해서 만들어진 유틸리티
<br>

## Ex
### AS-IS
```swift
private func downloadJsonAsync(_ url: String, _ completion: @escaping (String?) -> Void) {
        DispatchQueue.global().async {
            let url = URL(string: url)!
            let data = try! Data(contentsOf: url)
            let json = String(data: data, encoding: .utf8)
            DispatchQueue.main.async {
                completion(json)
            }
        }
    }
```
```swift
private func onLoadAsync() {
        // main ---
        editView.text = ""
        self.setVisibleWithAnimation(self.activityIndicator, true)
        
        // global --
        self.downloadJsonAsync(MEMBER_LIST_URL) { json in
            self.editView.text = json
            self.setVisibleWithAnimation(self.activityIndicator, false)
        }
    }
```
### TO-BE  
1. 비동기로 생기는 데이터를 Observable로 감싸서 리턴하는 방법
```swift
private func downloadJsonRx(_ url: String) -> Observable<String?> {
        return Observable.create() { emitter in
            let url = URL(string: url)!
            let task = URLSession.shared.dataTask(with: url) { data, response, err in
                
                guard err == nil else {
                    emitter.onError(err!)
                    return
                }
                
                if let res = data, let json = String(data: res, encoding: .utf8) {
                    emitter.onNext(json)
                }
                
                emitter.onCompleted()
            }
            
            task.resume()
            
            return Disposables.create() {
                task.cancel()
            }
        }
    }
```

2. Observable로 오는 데이터를 받아서 처리하는 방법  
```swift
func onLoad() {
        editView.text = ""
        self.setVisibleWithAnimation(self.activityIndicator, true)
        
        // 1 downloadJsonRx부터 ConcurrentDispatchQueueScheduler로 실행
        downloadJsonRx(MEMBER_LIST_URL)
            .observeOn(MainScheduler.instance) // 2 여기 이후부터
            .subscribeOn(ConcurrentDispatchQueueScheduler(qos: .default)) // 1
            // 2 main thread에서 실행
            .subscribe(onNext: { json in
                self.editView.text = json
                self.setVisibleWithAnimation(self.activityIndicator, false)
        })
        .disposed(by: disposeBag)
    }
```

## Contents
- Observable
- Operators
- Scheduler
- Subject
- Single

### Observable  
- 생명주기
 1. create  
 2. subscribe  
 3. onNext  
 ------ 끝 ------  
 5. onCompleted / onError    
 6. Disposed   
- with Async & Dispose
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
subscribe이 호출되어야 observable(sequence)이 생성됨
<br>

### Operators
> Sugar API
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

  - **bind**를 사용하면 순환참조 문제 쉽게 해결 가능  
  ```swift
  .subscribe(onNext: { [weak self] in
                self?.totalPrice.text = $0
            })
  ```
  <center>⬇️</center>

  ```swift
  .bind(to: itemCountLabel.rx.text)
  ```

<br>
        
### Scheduler  
<img src = "https://user-images.githubusercontent.com/33366446/126285994-126ec63f-f320-4efa-b467-5f015ff09fb1.png" width="400px">  

[이미지 출처: ReactiveX](http://reactivex.io/documentation/scheduler.html)  


<br>

1. observeOn(scheduler: ImmediateSchedulerType)  
 - Observable이 작업할 스레드 명시
 - observeOn 이후부터 스레드 변경처리 -> 선언 위치 상관 O 즉, downStream에 영향  
 - observable이 사용할 스레드가 어느 시점에서 할당되는지에 따라 그 후에 호출되는 operator가 영향을 받음  

```swift
.observeOn(MainScheduler.instance) // : main thread에서 실행
```
        
```swift
.observeOn(ConcurrentDispatchQueueScheduler(qos: .default)) // concurrent queue에서 실행 (async) 
``` 

<br>

2. subscribeOn(scheduler: ImmediateSchedulerType) 
 - Observable을 구독할 때 작업할 스케줄러를 명시 ( = seqence가 생성될 때 사용할 스케줄러 지정)  
 - 선언 위치 상관 X  
 - Observable이 subscribe 될 때부터 스케줄러를 적용하겠다는 뜻 --> upstream에 영향  

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
 Observable과 Observer 역할 모두 수행 가능  
 Observable인데 스스로 데이터를 발생시킬 수 있음  
 여러 개의 Observer를 subscribe 가능 (multicast)
 데이터도 넣을 수 있고, subscribe도 가능  
 외부에서 통제가 가능한 Observable
 
<br>

- AsyncSubject  
  - 끝나야 데이터 전달
- BehaviorSubject  
  <img src = "https://user-images.githubusercontent.com/33366446/126441898-46f912f5-6ab8-4df9-960c-f7c5c1af7809.png" width="300px">  

  [이미지 출처: ReactiveX](http://reactivex.io/documentation/subject.html)  

  - 초기 값을 갖고 생성되고, subscribe한 시점 이후부터 발생한 이벤트만 전달 받음
  - subscribe 하자마자 초기 값을 내려줌
- PublishSubject  
   <img src = "http://reactivex.io/documentation/operators/images/S.PublishSubject.png" width="300px">  
   [이미지 출처: ReactiveX](http://reactivex.io/documentation/subject.html)  

  - 빈 값이 생성되고, subscribe한 시점 이후부터 발생한 이벤트만 전달 받음
- ReplaySubject  
  - bufferSize를 갖고 생성되고, BehaviorSubject와 유사하지만 bufferSize만큼 최신 이벤트를 전달 


### BehaviorRelay   
- Subject를 wrapping한 것
- Stream 종료 X (complete X, error X . accpet만 됨) -->  주로 UI 그릴 때 사용
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

### Driver
- Observable을 wrapping한 것
- Subject의 에러 안나는 용도로 만들어진게 Relay인 것처럼 Observable의 에러 안나는 용도로 만들어진 것  
- error 방출 안함 --> UI 그릴 때 & MainScheduler에서 사용
- ex 
 ```swift
pwField.rx.text.orEmpty
             .asDriver()
             .drive(onNext: ((String) -> Void)?, onCompleted: (() -> Void)?, onDisposed: (() -> Void)?)
             .disposed(by: disposeBag)
 ```
        
## Ref
[RxSwift 4시간에 끝내기 시즌0](http://reactivex.io/documentation)  
https://www.youtube.com/channel/UCsrPur3UrxuwGmT1Jq6tkQw/videos
