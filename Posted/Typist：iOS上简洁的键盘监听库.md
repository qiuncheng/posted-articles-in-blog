## 前言
在iOS想要监听键盘的话是通过注册通知、接收键盘发来的通知实现的，虽然步骤不是很多，不过由于键盘的状态很多（有6种），如果需求要监听所有的状态，想想你要把相同的代码写6次，是有多烦人。而且适配的系统在iOS 9以下还要把一个个通知移除。这是有多不简洁...不过自从我遇到了`Typist`, 感觉优雅简洁到要死。
## Typist简介
> Typist is a small, drop-in Swift UIKit keyboard manager for iOS apps. It helps you manage keyboard's screen presence and behavior without notification center and Objective-C.

也就是说Typist主要方便了键盘显示的事件，那么他究竟是怎么方便的呢？下面这段代码就是Typist的简单使用：
```Swift
 let keyboard = Typist.shared
keyboard
    .on(event: .didShow) { (options) in
        print("New Keyboard Frame is \(options.endFrame).")
    }
    .on(event: .didHide) { (options) in
        print("It took \(options.animationDuration) seconds to animate keyboard out.")
    }
    .start()
```

Oh, my god.怎么看上去和Rx有点像呀。哈哈，确实是的，不过这里作者只是运用了方法的链式调用。并没有Rx那么高大上哈。  
如何你想要移除键盘监听，那么直接调用`keyboard.clear()`即可。  
**不过这里有点需要注意下，当有多个不同的对象都要监听键盘时，不要使用该单例。也就说这种情况下就需要你自己去创建Typist对象了，不要使用`Typist.shared`这个会导致你其中某些对象接受不到通知。**
## 从键盘监听说起
在不使用Typist的情况下，我们是怎么做键盘监听的呢？
![](http://images.vsccw.com/WX20170225-191427@2x.png)
具体步骤如图所示。不过在如果你的应用支持iOS 9+以上，那么就可以不用去移除通知，这个在Apple文档里也有说明。
可能只是一张图你看不出来有多烦人，那么我用代码演示下是这样的。
![](http://images.vsccw.com/WX20170225-192355@2x.png)
而且这仅仅只是监听了键盘的实现方法，就已经需要这么多代码了。那么我们要是遇到需求要监听所有的键盘通知呢？那么上面代码就成了这样：
![](http://images.vsccw.com/WX20170225-192738@2x.png)
一堆样式相同的模板代码，有人会说，我需要一个函数封装下，不过还是还是不够`Typist`简洁。

## 那么Typiest是如何做键盘监听的
```
let keyboard = Typist.shared // #1
 
keyboard
    .on(event: .didShow) { (options) in // #2
        print("New Keyboard Frame is \(options.endFrame).")
    }
    .on(event: .didHide) { (options) in // #3
        print("It took \(options.animationDuration) seconds to animate keyboard out.")
    }
    .start() // #4
```
还是从简介中的这段代码说起，**#1**处是使用单例创建一个Typist对象；**#2** 处监听了键盘显示结束，闭包回调里的options包含了一些键盘信息，也就是`NotificationCenter`里面的`info`信息；**#3** 处监听了键盘的隐藏结束，闭包回调同上；**#4** 在这里开始监听，相当与我们使用的addobserver: 方法。
具体的逻辑如下图所示：
![](http://images.vsccw.com/WX20170225-194337@2x.png)
那么它内部究竟是怎么做的呢？

### `public func on(event: KeyboardEvent, do callback: TypistCallback?) -> Self` 方法
这个方式的实现很简单，仅仅是将callbac回调放入一个dict里面，dict的key是event，然后返回自己。
```Swift
public func on(event: KeyboardEvent, do callback: TypistCallback?) -> Self {
        callbacks[event] = callback
        return self
}
```
### KeyboardEvent
作者在这里定义了一个枚举来封装了各个不同的键盘监听，然后我们在on方法里直接用传入枚举值就可以。

```
public enum KeyboardEvent {
    /// Event raised by UIKit's `.UIKeyboardWillShow`.
    case willShow
        
    /// Event raised by UIKit's `.UIKeyboardDidShow`.
    case didShow
        
    /// Event raised by UIKit's `.UIKeyboardWillShow`.
    case willHide
        
    /// Event raised by UIKit's `.UIKeyboardDidHide`.
    case didHide
        
    /// Event raised by UIKit's `.UIKeyboardWillChangeFrame`.
    case willChangeFrame
        
    /// Event raised by UIKit's `.UIKeyboardDidChangeFrame`.
    case didChangeFrame
}
```
### start 方法
start在这里仅仅是注册了键盘通知，注意下addObserver方法的参数。这里的event就是上面的KeyboardEvent，通过event映射到NSNotification.Name 和 SEL。
```
/// Starts listening to events and calling corresponding events handlers.
    public func start() {
        let center = NotificationCenter.`default`
        
        for event in callbacks.keys {
            center.addObserver(self, selector: event.selector, name: event.notification, object: nil)
        }
    }
```

### event.selector 与 event.notification 属性
这两个KeyboardEvent扩展下的私有属性分别返回上面提到的 NSNotification.Name 和 SEL
```
fileprivate extension Typist.KeyboardEvent {
    var notification: NSNotification.Name {
        get {
            switch self {
            case .willShow:
                return .UIKeyboardWillShow
            case .didShow:
                return .UIKeyboardDidShow
            case .willHide:
                return .UIKeyboardWillHide
            case .didHide:
                return .UIKeyboardDidHide
            case .willChangeFrame:
                return .UIKeyboardWillChangeFrame
            case .didChangeFrame:
                return .UIKeyboardDidChangeFrame
            }
        }
    }
    
    var selector: Selector {
        get {
            switch self {
            case .willShow:
                return #selector(Typist.keyboardWillShow(note:))
            case .didShow:
                return #selector(Typist.keyboardDidShow(note:))
            case .willHide:
                return #selector(Typist.keyboardWillHide(note:))
            case .didHide:
                return #selector(Typist.keyboardDidHide(note:))
            case .willChangeFrame:
                return #selector(Typist.keyboardWillChangeFrame(note:))
            case .didChangeFrame:
                return #selector(Typist.keyboardDidChangeFrame(note:))
            }
        }
    }
}
```

### SEL 在哪里实现
关于上面的`event.selector`是如何实现调用的。实际上是通过闭包的形式调用，作者在这里实现了每一个的监听响应方法，然后在该方法里去调用闭包来做。具体闭包的实现是使用者来实现的，然后通过key为event的dict来获取闭包。
举例代码：
```
internal func keyboardWillShow(note: Notification) {
    if let callback = callbacks[.willShow] {
        callback(keyboardOptions(fromNotificationDictionary: note.userInfo))
    }
}
```
### KeyboardOptions
这里主要是一些与键盘有关的信息。
![](http://images.vsccw.com/WX20170226-000006@2x.png)

## 总结
![](http://images.vsccw.com/WX20170225-234151@2x.png)

简单总结下该优雅的逻辑：

1. `let keyboard = Typist.shared`
2. 调用`func on(event: KeyboardEvent, do callback: TypistCallback?)`,
    - 该方法中仅仅做了一件事：`callbacks[event] = callback`
    - 显然这个闭包是我们去实现的。
3. 调用`start()`方法
    - 遍历了`callbacks`, 依次实现了addObserve方法，主要两个参数是`selector: event.selector, name: event.notification`
    - 其中`event.notification` event 和keyboard NSNotification.Name 的映射, `event.selector`中去调用了callbacks中的闭包，也就是我们的实现事件。
4. 可以调用stop()方法移除通知。
    - 本质上是调用了`removeObserver`方法.


## 参考
[Typist GitHub Repo](https://github.com/totocaster/Typist)