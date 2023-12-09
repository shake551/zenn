---
title: "FlutterのiOSアプリでアプリが起動していなくても位置情報を取得する"
emoji: "🏃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "iOS", "Dart", "Swift", "個人開発"]
published: false
---

# はじめに
個人開発でFlutterを使って「Kaimono」というiOSアプリを作っています．
https://apps.apple.com/jp/app/kaimono-%E5%BF%98%E3%82%8C%E3%81%95%E3%81%9B%E3%81%AA%E3%81%84%E8%B2%B7%E3%81%84%E7%89%A9%E3%83%AA%E3%82%B9%E3%83%88%E3%82%A2%E3%83%97%E3%83%AA/id6470928346
このアプリでは位置情報を監視して、登録している場所に近づくと通知を送信するという機能があります．
今回この機能を実現するために必要だった、iOSでのアプリが起動していない状態での位置情報の更新を取得する方法についてまとめようと思います．

# 前提
まず前提としてアプリの状態を以下の３状態と定義します．

- Foreground
  - アプリが起動した状態
- Background
  - アプリがバックグラウンドで動いている状態
- Terminated
  - アプリが起動していない状態（タスクキルされた状態など）

:::message
厳密にはもう少し細かく分けることもできますが、今回の内容ではそこまで細かい分類は必要ないため３状態としています．
細かい状態を確認したい場合はまず以下のドキュメントを読むと良いかと思います．
:::
https://developer.apple.com/documentation/uikit/app_and_environment/managing_your_app_s_life_cycle

今回やりたいこととしては、上記の３状態のうち、「Terminated」の時に位置情報の変更を受け取り、Flutter側のコードを実行して通知を送信するということですが、今回の記事では位置情報の変更を受け取る部分のみ取り上げたいと思います．

# 位置情報の変更を監視する方法
位置情報の変更を監視する方法についてはFlutterのpackageを探しましたが、あまり更新されていないようなものが多かったこと、アプリがkillされた状態で動作するものがなさそうだったことからこの部分だけSwiftで書くことにしました．

Core Location を使用します．
Core Location を使うことで、デバイスの位置情報などを取得することができます．より詳細な情報は以下のドキュメントにあります．
https://developer.apple.com/documentation/coreLocation

Core Locationサービスを設定・開始・停止する CLLocationManagerのオブジェクトはいくつかのロケーション関連のアクティビティをサポートしますが、今回は significant-change location service を使用しました．
significant-change location serviceは文字通り、重大な位置情報の変更を検知し、通知してくれます．
`startMonitoringSignificantLocationChanges()` を呼ぶことで位置情報の監視を始めることができ、アプリがどのような状態でも位置情報の変更を受け取ることができます．
CLLocationManagerのdistanceFilterというプロパティで更新イベントが発生する最小の移動距離を設定することができるのですが、significant-change location serviceではdistanceFilterには依存せずに更新イベントが生成されます．

# 位置情報の変更を監視する
まず、全体像は以下のようになっています．

```swift
import UIKit
import Flutter
import CoreLocation

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate, CLLocationManagerDelegate {
    var locationManager = CLLocationManager()

    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        GeneratedPluginRegistrant.register(with: self)

        if #available(iOS 10.0, *) {
            UNUserNotificationCenter.current().delegate = self as? UNUserNotificationCenterDelegate
        }
        locationManager.requestAlwaysAuthorization()
        locationManager.delegate = self

        if launchOptions?[UIApplication.LaunchOptionsKey.location] != nil {
            locationManager.startMonitoringSignificantLocationChanges()
        }

        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }

    override func applicationDidEnterBackground(_ application: UIApplication) {
        if CLLocationManager.significantLocationChangeMonitoringAvailable() {
            locationManager.startMonitoringSignificantLocationChanges()
        }
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        // 位置情報が更新された際に実行したい処理
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        // エラーが発生した際に実行したい処理
    }
}
```

## アプリ起動時
アプリがTerminatedの状態からsignificant-change location serviceによって起動される場合は `application(_:willFinishLaunchingWithOptions:)` や `application(_:didFinishLaunchingWithOptions:)` のオプションにlocationが入った状態で起動されます．
また、アプリが起動した後、引き続き位置情報の更新を監視したい場合は再度 `startMonitoringSignificantLocationChanges()` を呼び出す必要があります．
そのため、`application(_:didFinishLaunchingWithOptions:)`に以下のような記述を追加します．
```swift
        if launchOptions?[UIApplication.LaunchOptionsKey.location] != nil {
            locationManager.startMonitoringSignificantLocationChanges()
        }
```


## バックグラウンドへの移行時
バックグラウンドに移行した際には以下の部分が呼ばれます．
アプリが終了した際にsignificant-change location serviceなどが実行されていた場合、位置情報の変更を検知するとアプリを再起動してくれます．
したがって、アプリがバックグラウンドに移行する際にモニタリングを開始しておくとアプリのタスクがkillされるなどしてアプリが終了した場合でも位置情報の変更を検知して処理をすることができるようになります．
```swift
    override func applicationDidEnterBackground(_ application: UIApplication) {
        if CLLocationManager.significantLocationChangeMonitoringAvailable() {
            locationManager.startMonitoringSignificantLocationChanges()
        }
    }
```

## 位置情報の更新時
位置情報が更新された際は以下の部分が呼ばれます．
そのため、位置情報が更新された際に実行したい処理をここに書くと良いです．
```swift
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        // 位置情報が更新された際に実行したい処理
    }
```
この `locations` には複数のlocationが含まれることがあり、最新のlocationは配列の末尾にあります．

また、ドキュメントにもあるようにデリゲートオブジェクトは `locationManager(_:didUpdateLocations:)` を実装することに加えて、エラーハンドリングをするために `locationManager(_:didFailWithError:)` も実装する必要があります．
```swift
    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        // エラーが発生した際に実行したい処理
    }
```

# まとめ
アプリが起動していない状態でも位置情報の更新を取得する方法として、significant-change location serviceを使う方法をご紹介しました．
アプリが起動していなくても簡単に位置情報の更新を取得することができますが、この方法だと精度に問題があります．更新を受け取る間隔をこちらから指定することはできず、明確な基準も示されていません．（一応ログなどで確認したところ約500mおきに更新されているようでしたが、環境依存の可能性が高いです）そのため、もう少し精度よく取れるような方法を模索していきたいと思います．
今後の候補として、iOS17から追加されたCLLocationUpdateの `liveUpdates` や `CLBackgroundActivitySession` が挙げられます．
詳細は以下のビデオで説明されているので興味がある方はそちらも確認してみてください．
https://developer.apple.com/videos/play/wwdc2023/10180/

最後までご覧いただきありがとうございました！
