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

-------

## Extending CLLocationManager to get the current position

CLLocationManager+Rx
```swift
import Foundation
import CoreLocation
import RxSwift
import RxCocoa

extension CLLocationManager: HasDelegate {}

class RxCLLocationManagerDelegateProxy: DelegateProxy<CLLocationManager, CLLocationManagerDelegate>, DelegateProxyType, CLLocationManagerDelegate {
    weak public private(set) var locationManager: CLLocationManager?
    
    public init(locationManager: ParentObject) {
        self.locationManager = locationManager
        super.init(parentObject: locationManager,
                   delegateProxy: RxCLLocationManagerDelegateProxy.self)
    }
    
    static func registerKnownImplementations() {
        register { RxCLLocationManagerDelegateProxy(locationManager: $0) }
    }
}

public extension Reactive where Base: CLLocationManager {
    var delegate: DelegateProxy<CLLocationManager, CLLocationManagerDelegate> {
        RxCLLocationManagerDelegateProxy.proxy(for: base)
    }
    
    var didUpdateLocations: Observable<[CLLocation]> {
      delegate.methodInvoked(#selector(CLLocationManagerDelegate.locationManager(_:didUpdateLocations:)))
        .map { parameters in
          parameters[1] as! [CLLocation]
        }
    }
}
```

ViewController
```swift
    override func viewDidLoad() {
        super.viewDidLoad()
        
                geoLocationButton.rx.tap
          .subscribe(onNext: { [weak self] _ in
            guard let self = self else { return }

            self.locationManager.requestWhenInUseAuthorization()
            self.locationManager.startUpdatingLocation()
          })
          .disposed(by: bag)
        
        locationManager.rx.didUpdateLocations
          .subscribe(onNext: { locations in
            print(locations)
          })
          .disposed(by: bag)
```

![image](https://user-images.githubusercontent.com/47273077/190884928-e788d36f-ebbd-4709-bb36-3c17bd7b4693.png)


------------

## Updating the weather with the current data

CLLocationManager+Rx
```swift
public extension Reactive where Base: CLLocationManager {

    var authorizationStatus: Observable<CLAuthorizationStatus> {
      delegate.methodInvoked(#selector(CLLocationManagerDelegate.locationManager(_:didChangeAuthorization:)))
        .map { parameters in
          CLAuthorizationStatus(rawValue: parameters[1] as! Int32)!
        }
        .startWith(CLLocationManager.authorizationStatus())
    }
    
    func getCurrentLocation() -> Observable<CLLocation> {
        let location = authorizationStatus
            .filter { $0 == .authorizedWhenInUse || $0 == .authorizedAlways } // 1
            .flatMap { _ in self.didUpdateLocations.compactMap(\.first) } // 2
            .take(1) // 3
            .do(onDispose: { [weak base] in base?.stopUpdatingLocation() })
        
        base.requestWhenInUseAuthorization()
        base.startUpdatingLocation()
        
        return location // 4
    }
```

ViewController
```swift
    override func viewDidLoad() {
        super.viewDidLoad()
        
          let searchInput = searchCityName.rx
            .controlEvent(.editingDidEndOnExit)
            .map { self.searchCityName.text ?? "" }
            .filter { !$0.isEmpty }
        
        
        
          let geoSearch = geoLocationButton.rx.tap
          .flatMapLatest { _ in self.locationManager.rx.getCurrentLocation() }
          .flatMapLatest { location in
            ApiController.shared
              .currentWeather(at: location.coordinate)
              .catchErrorJustReturn(.dummy)
          }
        
        let textSearch = searchInput.flatMap { city in
          ApiController.shared
            .currentWeather(for: city)
            .catchErrorJustReturn(.dummy)
        }
        
        let search = Observable
          .merge(geoSearch, textSearch)
          .asDriver(onErrorJustReturn: .dummy)
          
                  let running = Observable.merge(
          searchInput.map { _ in true },
          geoLocationButton.rx.tap.map { _ in true },
          search.map { _ in false }.asObservable()
        )
        .startWith(true)
        .asDriver(onErrorJustReturn: false)
```

![image](https://user-images.githubusercontent.com/47273077/190885360-85eed7e8-4c34-448d-b3a0-71c89640b026.png)

![image](https://user-images.githubusercontent.com/47273077/190885363-eab46c25-3e0b-4d55-8747-e9926bc6c16d.png)



          
