# IntermediateRxCocoa

## RxSwift: Reactive Programming with Swift | raywenderlich.com
![image](https://user-images.githubusercontent.com/47273077/185172130-b3557025-c636-4a1b-8490-c900c8312b77.png)

### Showing an activity while searching

```swift
  override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
        style()
        
        let searchInput = searchCityName.rx
            .controlEvent(.editingDidEndOnExit)   
            .map { self.searchCityName.text ?? "" }
            .filter { !$0.isEmpty }
        
        let search = searchInput
          .flatMapLatest { text in
            ApiController.shared
              .currentWeather(for: text)
              .catchErrorJustReturn(.dummy)
          }
          .asDriver(onErrorJustReturn: .dummy)
        
        let running = Observable.merge(
            searchInput.map { _ in true },
            search.map { _ in false }.asObservable()
          )
          .startWith(true)
          .asDriver(onErrorJustReturn: false)

        running
          .skip(1)
          .drive(activityIndicator.rx.isAnimating)
          .disposed(by: bag)
        
        running
          .drive(tempLabel.rx.isHidden)
          .disposed(by: bag)

        running
          .drive(iconLabel.rx.isHidden)
          .disposed(by: bag)

        running
          .drive(humidityLabel.rx.isHidden)
          .disposed(by: bag)

        running
          .drive(cityNameLabel.rx.isHidden)
          .disposed(by: bag)
```
    
![image](https://user-images.githubusercontent.com/47273077/190880653-d15266d9-500d-46d4-80dd-de55ed8e5ea3.png)
  
<img width="300" src="https://user-images.githubusercontent.com/47273077/190880772-a1915f9d-976c-416b-8b02-a848d824ad91.gif">
