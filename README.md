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
#### Observable을 create해주는 방법
1. just  
just에 넘겨준 element 그대로 전달
```
Observable.just("RxSwift")
            .subscribe(onNext: { str in
                print(str) // RxSwift
            })
            .disposed(by: disposeBag)
```
2. from  
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

3. map  
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
