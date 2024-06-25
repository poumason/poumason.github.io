---
title: 實現 iOS 在 UIActivityViewController 加入行事曆
date: 2020-08-16 22:26:19
categories: Flutter
tags:
    - Flutter
    - iOS
---
如果要在 Flutter 呼叫分享很簡單，利用 [share](https://pub.dev/packages/share) plugin 一下就完成，開發過 Flutter 的人一定都知道。
剛好今天有需求是在 iOS 分享活動連結時可以多顯示一個自訂的按鈕，讓使用者可以加入到行事曆。
<!--more-->
要做到 iOS 分享時加入自訂的按鈕，有兩個重要的元素： [UIActivityController](http://developer.apple.com/documentation/uikit/uiactivityviewcontroller) 與 [UIActivity](http://developer.apple.com/documentation/uikit/uiactivity) 。

## UIActivityController
系統提供多種 services (例如：複製內容到剪貼簿，分享文字到 social media 或 email ...等)。也提供開發者自訂 service 來提供服務。

[UIActivityController](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller) 集合這些 services 來呈現在畫面上供用戶選擇。可設定該 View Controller 傳遞的資料結構與對應的 services。

最簡單的分享如下：
```
let shareItems = ["Hello"]
let activityVC = UIActivityViewController(activityItems: shareItems, applicationActivities: nil)

self.presentViewController(activityVC, animated: true, completion: nil)
```
- [activityItems](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller/1622019-init) ：資料物件的 array，可以是字串，圖片或自訂的資料內容。
- [applicationActivities](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller/1622019-init) [：UIActivity](https://developer.apple.com/documentation/uikit/uiactivity) 的 array，放有自訂 service 的 [UIActivity](http://developer.apple.com/documentation/uikit/uiactivity) 。

## UIActivity
該類別配合 [UIActivityController](http://developer.apple.com/documentation/uikit/uiactivityviewcontroller) 使用，如果想要提供自訂的 service 給用戶使用，需要繼承該類別來實作並處理用戶傳入的資料做互動。

需要 override 幾個地方：

- [activityType](https://developer.apple.com/documentation/uikit/uiactivity/1620671-activitytype) ：代表該 service 的識別。例如：通常使用 [Bundle.main.bundleIdentifier](https://developer.apple.com/documentation/foundation/nsbundle/1418023-bundleidentifier) 加一些自訂的值。
- [activityTitle](https://developer.apple.com/documentation/uikit/uiactivity/1620674-activitytitle) ：顯示在 [UIActivityController](http://developer.apple.com/documentation/uikit/uiactivityviewcontroller) 的名稱，要記得做多國語系的處理。
- [activityImage](https://developer.apple.com/documentation/uikit/uiactivity/1620658-activityimage) ：顯示在 [UIActivityController](http://developer.apple.com/documentation/uikit/uiactivityviewcontroller) 的圖示。需要處理不同 iOS 版本需要的圖示大小不一樣。如果使用 iOS 內建系統圖示的話，在 iOS13 可以額外設定大小：UIImage.SymbolConfiguration。
- [activityCategory](https://developer.apple.com/documentation/uikit/uiactivity/1620677-canperform) ：定義該 service 的類型，[UIActivity.Category](https://developer.apple.com/documentation/uikit/uiactivity/category) 提供了 action 與 share，自訂的 service 要給 action。
- [canPerform](https://developer.apple.com/documentation/uikit/uiactivity/1620677-canperform) ：可以根據傳入的 data 先做檢查，如果可以處理就回傳 true。
- [prepare](https://developer.apple.com/documentation/uikit/uiactivity/1620668-prepare) ：在用戶選擇該 service 時會呼叫該 method，該 method 則實際處理資料被如何使用。如果需要而外的 UI 互動，也是在這裡準備好需要的 view controller。
 

操作行事曆則需要： [EKEventStore](http://developer.apple.com/documentation/eventkit/ekeventstore)，[EKEventEditViewController](http://developer.apple.com/documentation/eventkitui/ekeventeditviewcontroller)。

## EKEventStore
管理存取行事曆與提醒權限的元件。需要在初始化後，利用 [requestAccess(to:completion:)](https://developer.apple.com/documentation/eventkit/ekeventstore/1507547-requestaccess) 來取得存取權限。

需要在 Info.plist 加入  [NSRemindersUsageDescription](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW16) 與 [NSCalendarsUsageDescription](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW15) 的宣告，才能使用  EKEventStore。

## EKEventEditViewController
view controller 用來建立，編輯或刪除行事曆的活動。使用該 view congtroller 的 class 需要實作 [EKEventEditViewDelegate](https://developer.apple.com/documentation/eventkitui/ekeventeditviewdelegate)。

 
## 範例
介紹完 iOS 怎麼使用分享介面([UIActivityViewController](http://developer.apple.com/documentation/uikit/uiactivityviewcontroller)) 與操作行事曆([EKEventEditViewController](https://developer.apple.com/documentation/eventkitui/ekeventeditviewcontroller))後，下面串起來從 Flutter 利用 method channel 通知 iOS 顯示分享的介面。

1. 實作 EventActivity 繼承 [UIActivity](https://developer.apple.com/documentation/uikit/uiactivity) 處理傳入的活動資訊，並呼叫行事曆([EKEventEditViewController](https://developer.apple.com/documentation/eventkitui/ekeventeditviewcontroller))
Info.plist 加入存取行事曆的宣告。
```
<plist version="1.0">
  <dict>
    ...

    <key>NSCalendarsUsageDescription</key>
    <string>to add this event to your calendar</string>
  </dict>
</plist>
```
EventActivity.swift
```
override func prepare(withActivityItems activityItems: [Any]) {
  // 利用 EKEventStore 請求操作 Calendar 的權限
  let eventStore = EKEventStore()
  eventStore.requestAccess(to: .event) { (granted, error) in
    if (granted && error == nil) {
      // 先關閉 UIActivityViewController 再開啟 EKEventEditViewController
      DispatchQueue.main.asyncAfter(deadline: .now() + 0.7) {
        // 把 activityItems 帶入的參數包裝成 EKEvent 
        let event = self.genereateEvent(eventStore: eventStore, arguments: activityItems as NSArray);

        if (event == nil) {
          return
        }
        // 利用 EKEventStore 把 EKEvent 加入             
        self.insertEvent(event: event!, eventStore: eventStore)
      }
    } else {
      // 如果被取消授權要顯示訊息告訴使用者
      self.showAccessDeinedOrRestricted()
    }
  }
}
```
利用 [EKEventStore](https://developer.apple.com/documentation/eventkit/ekeventstore) 請求並取得權限之後，利用  [DispatchQueue.main.asyncAfter(deadline: .now() + 0.7)](https://developer.apple.com/documentation/dispatch/dispatchqueue/2300020-asyncafter) 延後 7 秒的方式來開啟 [EKEventEditViewController](https://developer.apple.com/documentation/eventkitui/ekeventeditviewcontroller)。

為什麼需要延後？
**因為 rootViewController 開啟了 UIActivityViewController ，無法再從 UIActivityViewController 開一個 ViewController，需要先關它後才能再開啟 EKEventEditViewController。**

2. iOS 定義 method channel 接受來自 Flutter 的傳入活動資訊
打開 AppDelegate.swift 加入下面的 code：
```
override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
  
  guard let controller = window?.rootViewController as? FlutterViewController else {
    fatalError("rootViewController is not type FlutterViewController")
  }
  
  // 定義要處理的 method channel      
  let methodChannel = FlutterMethodChannel(name: "sample.poumason.dev/channels", binaryMessenger: controller.binaryMessenger)
        
  methodChannel.setMethodCallHandler({
    (call: FlutterMethodCall, result: @escaping FlutterResult) -> Void in
    // 處理 shared 的 method       
    if (call.method == "shared") {
      // 呼叫自訂的 UIActivity 
      self.showSharedActivityViewController(arguments: call.arguments)
        result("OK")
        return
    }
             
    result(FlutterMethodNotImplemented)
  })
        
  GeneratedPluginRegistrant.register(with: self)
  return super.application(application, didFinishLaunchingWithOptions: launchOptions)        
}
```
詳細介紹整合 method channel 的說明可以參考：[Writing custom platform-specific code](https://flutter.dev/docs/development/platform-integration/platform-channels)。

3. 在 AppDelegate.swift 實現 showSharedActivityViewController method 來呼叫 [UIActivityViewController](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller)，並加入自訂的 [UIActivity](https://developer.apple.com/documentation/uikit/uiactivity)。
```
// 要實現 EKEventEditViewDelegate 接受關閉 EKEventEditViewController 的事件
@objc class AppDelegate: FlutterAppDelegate, EKEventEditViewDelegate  {

  func eventEditViewController(_ controller: EKEventEditViewController, didCompleteWith action: EKEventEditViewAction)
  {
      print(action)
      controller.dismiss(animated: true, completion: nil)
  }

  private func showSharedActivityViewController(arguments: Any?) {
    if let args = arguments as? Dictionary<String, Any?> , !args.isEmpty {
            
      guard let url = args["url"] as? String, !url.isEmpty else {
        print("no any be shared data")
        return
      }
            
      // 把傳入的資料裝到一個自訂的 Event 資料結構
      let event = Event.init(title: args["title"] as? String,
                             location: args["location"] as? String,
                             url: args["url"] as? String,
                             startDate: args["startDate"] as? Double,
                             endDate: args["endDate"] as? Double)
            
      let items: [Any]
      let activities: [UIActivity]?
      // 判斷如果是 Event 類型才呼叫自訂的 EventActivity，不然視為一般的分享
      if (event.isValidated()) {
        items = [ url, event ]
        activities = [ EventActivity() ]
      } else {
        items = [ url ]
        activities = nil
      }
            
      let activityVC = UIActivityViewController(activityItems: items, applicationActivities: activities)
      self.window.rootViewController?.present(activityVC, animated: true, completion:  nil)
    }
  }
}
```

4. 從 Flutter 使用 method channel 送出資料
```
Future<void> _sharedEvent() async {
  // 建立相同 name 的 method channel
  final platform = const MethodChannel('sample.poumason.dev/channels');
  try {
    // 呼叫定義好的 shared method name
    var result = await platform.invokeMethod('shared', {
      'url': _urlKey.currentState.value,
      'title': _titleKey.currentState.value,
      'location': _addressKey.currentState.value,
      'startDate':
         (_startKey.currentState.value.millisecondsSinceEpoch / 1000).roundToDouble(),
      'endDate': (_endKey.currentState.value.millisecondsSinceEpoch / 1000).roundToDouble(),
      });
    print(result);
  } on PlatformException catch (e) {
    print(e.message);
  }
}
```
method channel 傳遞參數的類型是有限制的，可以參考 [Platform channel data types support and codecs](https://flutter.dev/docs/development/platform-integration/platform-channels) 的定義。

5. 範例結果 
   |    |    |  |
   | -- | -- |--|
   |   ![](1597566128.png)|   ![](1597566139.png)|   ![](1597566151.png)|
   
   範例程式：[share_calednar](https://github.com/poumason/flutter_samples/tree/master/share_calendar)。
---
以上是介紹如何從 Flutter 加入活動資料到 iOS 的行事曆。希望對大家有所幫助，謝謝。
 

## 參考資料
- [[iOS] Get the name of the running application](https://www.itread01.com/content/1545857656.html)
- [CFBundleDisplayName](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundledisplayname)
- [[iOS] swift : use of %s in String(format: …)](https://stackoverflow.max-everyday.com/2018/01/ios-swift-use-of-s-in-stringformat/)
- [Remove HTML tags from a String in Dart](https://stackoverflow.com/questions/51593790/remove-html-tags-from-a-string-in-dart)
- [Manage calendar events with EventKit and EventKitUI with Swift](https://medium.com/@fede_nieto/manage-calendar-events-with-eventkit-and-eventkitui-with-swift-74e1ecbe2524)
- [iOS 的原生分享， UIActivityViewController](https://medium.com/@zhongwei0717/ios-%E7%9A%84%E5%8E%9F%E7%94%9F%E5%88%86%E4%BA%AB-uiactivityviewcontroller-f73d272dc2a3)
- [How to use EKEventEditViewController in Swift to let user save event to iOS calendar](https://dev.to/nemecek_f/how-to-use-ekeventeditviewcontroller-in-swift-to-let-user-save-event-to-ios-calendar-d8)
- [Swift — 玩玩 UIActivityViewController](https://medium.com/@JJeremy.XUE/swift-%E7%8E%A9%E7%8E%A9-uiactivityviewcontroller-5995bb80ff68)
- [How to use SF Symbols in SwiftUI](https://www.simpleswiftguide.com/how-to-use-sf-symbols-in-swiftui/)
- [UIActivityViewController by example](https://www.hackingwithswift.com/articles/118/uiactivityviewcontroller-by-example)
- [【Flutter基礎概念與實作】 Day18–Flutter測試框架以及Mockito Package使用範例介紹](https://ithelp.ithome.com.tw/articles/10223393)
- [SF Symbols in iOS 13](https://medium.com/@craiggrummitt/sf-symbols-in-ios-13-55e5febf6db6)
- [初學者指南：使用社交框架與 UIActivityViewController](https://www.appcoda.com.tw/social-framework-introduction/)
- [Sample Form — Part 1— Flutter](https://medium.com/flutterpub/sample-form-part-1-flutter-35664d57b0e5)
- [datetime_picker_formfield](https://pub.dev/packages/datetime_picker_formfield)
