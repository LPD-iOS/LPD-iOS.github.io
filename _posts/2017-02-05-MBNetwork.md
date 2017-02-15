---
layout: post
title: '造轮子 | 如何设计一个面向协议的 iOS 网络请求库'
date: '2017-02-12'
header-img: "img/post-bg-web.jpg"
tags:
     - iOS
     - POP
     - MBNetwork
author: 'MmoaaY'
---

最近开源了一个面向协议设计的网络请求库 [`MBNetwork`](https://github.com/mmoaay/MBNetwork)，基于 `Alamofire` 和 `ObjectMapper` 实现，目的是简化业务层的网络请求操作。

# 需要干些啥

对于大部分 App 而言，业务层做一次网络请求通常关心的问题有如下几个：

 - 如何在任意位置发起网络请求。
 - 表单创建。包含请求地址、请求方式（GET／POST／……）、请求头等……
 - 加载遮罩。目的是阻塞 UI 交互，同时告知用户操作正在进行。比如提交表单时在提交按钮上显示 “菊花”，同时使其失效。
 - 加载进度展示。下载上传图片等资源时提示用户当前进度。
 - 断点续传。下载上传图片等资源发生错误时可以在之前已完成部分的基础上继续操作，这个 `Alamofire` 可以支持。
 - 数据解析。因为目前主流服务端和客户端数据交换采用的格式是 `JSON`，所以我们暂时先考虑 `JSON` 格式的数据解析，这个 `ObjectMapper` 可以支持。
 - 出错提示。发生业务异常时，直接显示服务端返回的异常信息。前提是服务端异常信息足够友好。
 - 成功提示。请求正常结束时提示用户。
 - 网络异常重新请求。显示网络异常界面，点击之后重新发送请求。

# 为什么是 `POP` 而不是 `OOP`

关于 `POP` 和 `OOP` 这两种设计思想及其特点的文章很多，所以我就不废话了，主要说说为啥要用 `POP` 来写 `MBNetwork`。

 - 想尝试一下**一切皆协议**的设计方式。所以这个库的设计只是一次极限尝试，并不代表这就是最完美的设计方式。
 - 如果以 `OOP` 的方式实现，使用者需要通过继承的方式来获得某个类实现的功能，如果使用者还需要另外某个类实现的功能，就会很尴尬。而 `POP` 是通过对协议进行扩展来实现功能，使用者可以同时遵循多个协议，轻松解决 `OOP` 的这个硬伤。
 - `OOP` 继承的方式会使某些子类获得它们不需要的功能。
 - 如果因为业务的增多，需要对某些业务进行分离，`OOP` 的方式还是会碰到子类不能继承多个父类的问题，而 `POP` 则完全不会，分离之后，只需要遵循分离后的多个协议即可。
 - `OOP` 继承的方式入侵性比较强。
 - `POP` 可以通过扩展的方式对各个协议进行默认实现，降低使用者的学习成本。
 - 同时 `POP` 还能让使用者对协议做自定义的实现，保证其高度可配置性。

# 站在 `Alamofire` 的肩膀上

很多人都喜欢说 `Alamofire` 是 `Swift` 版本的 `AFNetworking`，但是在我看来，`Alamofire` 比 `AFNetworking` 更纯粹。这和 `Swift` 语言本身的特性也是有关系的，`Swift` 开发者们，更喜欢写一些轻量的框架。比如 `AFNetworking` 把很多 UI 相关的扩展功能都做在框架内，而 `Alamofire` 的做法则是放在另外的扩展库中。比如 [`AlamofireImage`](https://github.com/Alamofire/AlamofireImage) 和 [`AlamofireNetworkActivityIndicator`](https://github.com/Alamofire/AlamofireNetworkActivityIndicator)

而 `MBNetwork` 就可以当做是 `Alamofire` 的一个扩展库，所以，`MBNetwork` 很大程度上遵循了 `Alamofire` 接口的设计规范。一方面，降低了 `MBNetwork` 的学习成本，另一方面，从个人角度来看，`Alamofire` 确实有很多特别值得借鉴的地方。

## `POP`

首先当然是 `POP` 啦，`Alamofire` 大量运用了  `protocol` + `extension` 的实现方式。

## `enum`

做为检验写 `Swift` 姿势正确与否的重要指标，`Alamofire` 当然不会缺。

## 链式调用

这是让 `Alamofire` 成为一个优雅的网络框架的重要原因之一。这一点 `MBNetwork` 也进行了完全的 Copy。

## `@discardableResult`

在 `Alamofire` 所有带返回值的方法前面，都会有这么一个标签，其实作用很简单，因为在 `Swift` 中，返回值如果没有被使用，Xcode 会产生告警信息。加上这个标签之后，表示这个方法的返回值就算没有被使用，也不产生告警。

# 当然还有 `ObjectMapper` 

引入 `ObjectMapper` 很大一部分原因是需要做错误和成功提示。因为只有解析服务端的错误信息节点才能知道返回结果是否正确，所以我们引入  `ObjectMapper` 来做 `JSON` 解析。 而只做 `JSON` 解析的原因是目前主流的服务端客户端数据交互格式是 `JSON`。

这里需要提到的就是另外一个 `Alamofire` 的扩展库 [`AlamofireObjectMapper`](https://github.com/tristanhimmelman/AlamofireObjectMapper)，从名字就可以看出来，这个库就是参照 `Alamofire` 的 API 规范来做  `ObjectMapper` 做的事情。这个库的代码很少，但实现方式非常 `Alamofire`，大家可以拜读一下它的源码，基本上就知道如何基于 `Alamofire` 做自定义数据解析了。

> 注：被 @Foolish 安利，正在接入 `ProtoBuf` 中…

# 一步一步来

## 表单创建

`Alamofire` 的请求有三种： `request`、`upload` 和 `download`，这三种请求都有相应的参数，`MBNetwork` 把这些参数抽象成了对应的协议，具体内容参见：[MBForm.swift](https://github.com/mmoaay/MBNetwork/blob/master/MBNetwork/Classes/Form/MBForm.swift)。这种做法有几个优点：

 1. 对于类似 `headers` 这样的参数，一般全局都是一致的，可以直接 extension 指定。
 2. 通过协议的名字即可知道表单的功能，简单明确。

下面是 `MBNetwork` 表单协议的用法举例：

指定全局 `headers` 参数：

``` swift
extension MBFormable {
    public func headers() -> [String: String] {
        return ["accessToken":"xxx"];
    }
}
```

创建具体业务表单：

``` swift
struct WeatherForm: MBRequestFormable {
    var city = "shanghai"

    public func parameters() -> [String: Any] {
        return ["city": city]
    }

    var url = "https://raw.githubusercontent.com/tristanhimmelman/AlamofireObjectMapper/2ee8f34d21e8febfdefb2b3a403f18a43818d70a/sample_keypath_json"
    var method = Alamofire.HTTPMethod.get
}
```

> 表单协议化可能有过度设计的嫌疑，有同感的仍然可以使用 `Alamofire` 对应的接口去做网络请求，不影响 `MBNetwork` 其他功能的使用。

## 基于表单请求数据

表单已经抽象成协议，现在就可以基于表单发送网络请求了，因为之前已经说过需要在任意位置发送网络请求，而实现这一点的方法基本就这几种：

 - 单例。
 - 全局方法，`Alamofire` 就是这么干的。
 - 协议扩展。

`MBNetwork` 采用了最后一种方法。原因很简单，`MBNetwork` 是以**一切皆协议**的原则设计的，所以我们把网络请求抽象成 `MBRequestable` 协议。

首先，`MBRequestable` 是一个空协议 。

``` swift
///  Network request protocol, object conforms to this protocol can make network request
public protocol MBRequestable: class {

}
```

为什么是空协议，因为不需要遵循这个协议的对象干啥。

然后对它做 `extension`，实现网络请求相关的一系列接口：

``` swift
func request(_ form: MBRequestFormable) -> DataRequest

func download(_ form: MBDownloadFormable) -> DownloadRequest

func download(_ form: MBDownloadResumeFormable) -> DownloadRequest

func upload(_ form: MBUploadDataFormable) -> UploadRequest

func upload(_ form: MBUploadFileFormable) -> UploadRequest

func upload(_ form: MBUploadStreamFormable) -> UploadRequest

func upload(_ form: MBUploadMultiFormDataFormable, completion: ((UploadRequest) -> Void)?)
```

这些就是网络请求的接口，参数是各种表单协议，接口内部调用的其实是 `Alamofire` 对应的接口。注意它们都返回了类型为 `DataRequest`、`UploadRequest` 或者 `DownloadRequest` 的对象，通过返回值我们可以继续调用其他方法。

到这里 `MBRequestable` 的实现就完成了，使用方法很简单，只需要设置类型遵循 `MBRequestable` 协议，就可以在该类型内发起网络请求。如下：

``` swift
class LoadableViewController: UIViewController, MBRequestable {
    override func viewDidLoad() {
        super.viewDidLoad()

        // Do any additional setup after loading the view.
        request(WeatherForm())
    }
}
```

## 加载

对于加载我们关心的点有如下几个：

 - 加载开始需要干啥。
 - 加载结束需要干啥。
 - 是否需要显示加载遮罩。
 - 在何处显示遮罩。
 - 显示遮罩的内容。

对于这几点，我对协议的划分是这样的：

 - `MBContainable` 协议。遵循该协议的对象可以做为加载的容器。
 - `MBMaskable` 协议。遵循该协议的 `UIView` 可以做为加载遮罩。
 - `MBLoadable` 协议。遵循该协议的对象可以定义加载的配置和流程。

### `MBContainable`

遵循这个协议的对象只需要实现下面的方法即可：

``` swift
func containerView() -> UIView?
```

这个方法返回做为遮罩容器的 `UIView`。做为遮罩的 `UIView` 最终会被添加到 `containerView` 上。

不同类型的容器的 `containerView` 是不一样的，下面是各种类型容器 `containerView` 的列表：

|容器| `containerView`|
|:-:|:-:|
|`UIViewController`| `view`|
|`UIView`| `self`|
|`UITableViewCell`| `contentView`|
|`UIScrollView`| 最近一个不是 `UIScrollView` 的 `superview`|

`UIScrollView` 这个地方有点特殊，因为如果直接在 `UIScrollView` 上添加遮罩视图，遮罩视图的中心点是非常难控制的，所以这里用了一个技巧，递归寻找 `UIScrollView` 的 `superview`，发现不是 `UIScrollView` 类型的直接返回即可。代码如下：

``` swift
public override func containerView() -> UIView? {
    var next = superview
    while nil != next {
        if let _ = next as? UIScrollView {
            next = next?.superview
        } else {
            return next
        }
    }
    return nil
}
```

最后我们对 `MBContainable` 做 `extension`，添加一个 `latestMask` 方法，这个方法实现的功能很简单，就是返回 `containerView` 上最新添加的、而且遵循 `MBMaskable` 协议的 `subview`。

### `MBMaskable`

协议内部只定义了一个属性 `maskId`，作用是用来区分多种遮罩。

`MBNetwork` 内部实现了两个遵循 `MBMaskable` 协议的 `UIView`，分别是 [`MBActivityIndicator`](https://github.com/mmoaay/MBNetwork/blob/master/MBNetwork/Classes/View/MBActivityIndicator.swift) 和 [`MBMaskView`](https://github.com/mmoaay/MBNetwork/blob/master/MBNetwork/Classes/View/MBMaskView.swift)，其中 `MBMaskView` 的效果是参照 `MBProgressHUD` 实现，所以对于大部分场景来说，直接使用这两个 `UIView` 即可。

>注：`MBMaskable` 协议唯一的作用是与 `containerView` 上其它 `subview` 做区分。

### `MBLoadable`

做为加载协议的核心部分，`MBLoadable` 包含如下几个部分：

 - `func mask() -> MBMaskable?`：遮罩视图，可选的原因是可能不需要遮罩。
 - `func inset() -> UIEdgeInsets`：遮罩视图和容器视图的边距，默认值 `UIEdgeInsets.zero`。
 - `func maskContainer() -> MBContainable?`：遮罩容器视图，可选的原因是可能不需要遮罩。
 - `func begin()`：加载开始回调方法。
 - `func end()`：加载结束回调方法。

然后对协议要求实现的几个方法做默认实现：

``` swift
func mask() -> MBMaskable? {
    return MBMaskView() // 默认显示 MBProgressHUD 效果的遮罩。
}

 func inset() -> UIEdgeInsets {
    return UIEdgeInsets.zero // 默认边距为 0 。
}

func maskContainer() -> MBContainable? {
    return nil // 默认没有遮罩容器。
}

func begin() {
    show() // 默认调用 show 方法。
}

func end() {
    hide() // 默认调用 hide 方法。
}
```

上述代码中的 `show` 方法和 `hide` 方法是实现加载遮罩的核心代码。

`show` 方法的内容如下：

``` swift
func show() {
    if let mask = self.mask() as? UIView {
        var isHidden = false
        if let _ = self.maskContainer()?.latestMask() {
            isHidden = true
        }
        self.maskContainer()?.containerView()?.addMBSubView(mask, insets: self.inset())
        mask.isHidden = isHidden

        if let container = self.maskContainer(), let scrollView = container as? UIScrollView {
            scrollView.setContentOffset(scrollView.contentOffset, animated: false)
            scrollView.isScrollEnabled = false
        }
    }
}
```

这个方法做了下面几件事情：

 - 判断 `mask` 方法返回的是不是遵循 `MBMaskable` 协议的 `UIView`，因为如果不是 `UIView`，不能被添加到其它的 `UIView` 上。
 - 通过 `MBContainable` 协议上的 `latestMask` 方法获取最新添加的、且遵循 `MBMaskable` 协议的 `UIView`。如果有，就把新添加的这个遮罩视图隐藏起来，再添加到 `maskContainer` 的 `containerView` 上。为什么会有多个遮罩的原因是多个网络请求可能同时遮罩某一个 `maskContainer`，另外，多个遮罩不能都显示出来，因为有的遮罩可能有半透明部分，所以需要做隐藏操作。至于为什么都要添加到 `maskContainer` 上，是因为我们不知道哪个请求会最后结束，所以就采取每个请求的遮罩我们都添加，然后结束一个请求就移除一个遮罩，请求都结束的时候，遮罩也就都移除了。
 - 对 `maskContainer` 是 `UIScrollView` 的情况做特殊处理，使其不可滚动。

然后是 `hide` 方法，内容如下：

``` swift
func hide() {
    if let latestMask = self.maskContainer()?.latestMask() {
        latestMask.removeFromSuperview()

        if let container = self.maskContainer(), let scrollView = container as? UIScrollView {
            if false == latestMask.isHidden {
                scrollView.isScrollEnabled = true
            }
        }
    }
}
```

相比 `show` 方法，`hide` 方法做的事情要简单一些，通过 `MBContainable` 协议上的 `latestMask` 方法获取最新添加的、且遵循 `MBMaskable` 协议的 `UIView`，然后从 `superview` 上移除。对 `maskContainer` 是 `UIScrollView` 的情况做特殊处理，当被移除的遮罩是最后一个时，使其可以再滚动。

### `MBLoadType`

为了降低使用成本，`MBNetwork` 提供了 `MBLoadType` 枚举类型。

``` swift
public enum MBLoadType {
    case none
    case `default`(container: MBContainable)
}
```

`none`：表示不需要加载。
`default`：传入遵循 `MBContainable` 协议的 `container` 附加值。

然后对 `MBLoadType` 做 `extension`，使其遵循 `MBLoadable` 协议。

``` swift
extension MBLoadType: MBLoadable {
    public func maskContainer() -> MBContainable? {
        switch self {
        case .default(let container):
            return container
        case .none:
            return nil
        }
    }
}
```

这样对于不需要加载或者只需要指定 `maskContainer` 的情况（PS：比如全屏遮罩），就可以直接用 `MBLoadType` 来代替 `MBLoadable`。

### 常用控件支持

#### `UIControl`

 - `maskContainer` 就是本身，比如 `UIButton`，加载时直接在按钮上显示“菊花”即可。
 - `mask` 需要定制下，不能是默认的 `MBMaskView`，而应该是 `MBActivityIndicator`，然后 `MBActivityIndicator` “菊花”的颜色和背景色应该和 `UIControl` 一致。
 - 加载开始和加载全部结束时需要设置 `isEnabled`。


#### `UIRefreshControl`

 - 不需要显示加载遮罩。
 - 加载开始和加载全部结束时需要调用 `beginRefreshing` 和 `endRefreshing`。

#### `UITableViewCell`

 - `maskContainer` 就是本身。
 - `mask` 需要定制下，不能是默认的 `MBMaskView`，而应该是 `MBActivityIndicator`，然后 `MBActivityIndicator` “菊花”的颜色和背景色应该和 `UIControl` 一致。

### 结合网络请求

至此，加载相关协议的定义和默认实现都已经完成。现在需要做的就是把加载和网络请求结合起来，其实很简单，之前 `MBRequestable` 协议扩展的网络请求方法都返回了类型为 `DataRequest`、`UploadRequest` 或者 `DownloadRequest` 的对象，所以我们对它们做 `extension`，然后实现下面的 `load` 方法即可。

``` swift
func load(load: MBLoadable = MBLoadType.none) -> Self {
    load.begin()
    return response { (response: DefaultDataResponse) in
        load.end()
    }
}
```

传入参数为遵循 `MBLoadable` 协议的 `load` 对象，默认值为 `MBLoadType.none`。请求开始时调用其 `begin` 方法，请求返回时调用其 `end` 方法。

### 使用方法

#### 基础用法

##### 在 `UIViewController` 上显示加载遮罩

![](http://img.blog.csdn.net/20170109145516357?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

``` swift
request(WeatherForm()).load(load: MBLoadType.default(container: self))
```

##### 在 `UIButton` 上显示加载遮罩

![](http://img.blog.csdn.net/20170109145608670?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

``` swift
request(WeatherForm()).load(load: button)
```

##### 在 `UITableViewCell` 上显示加载遮罩

![](http://img.blog.csdn.net/20170109145921301?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


``` swift
override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    tableView .deselectRow(at: indexPath, animated: false)
    let cell = tableView.cellForRow(at: indexPath)
    request(WeatherForm()).load(load: cell!)
}
```

##### `UIRefreshControl`

![](http://img.blog.csdn.net/20170109152146307?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

``` swift
refresh.attributedTitle = NSAttributedString(string: "Loadable UIRefreshControl")
refresh.addTarget(self, action: #selector(LoadableTableViewController.refresh(refresh:)), for: .valueChanged)
tableView.addSubview(refresh)
     
func refresh(refresh: UIRefreshControl) {
    request(WeatherForm()).load(load: refresh)
}
```

#### 进阶

除了基本的用法，`MBNetwork` 还支持对加载进行完全的自定义，做法如下：

![](http://img.blog.csdn.net/20170109145842909?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

首先，我们创建一个遵循 `MBLoadable` 协议的类型 `LoadConfig`。

``` swift
class LoadConfig: MBLoadable {
    init(container: MBContainable? = nil, mask: MBMaskable? = MBMaskView(), inset: UIEdgeInsets = UIEdgeInsets.zero) {
        insetMine = inset
        maskMine = mask
        containerMine = container
    }
    
    func mask() -> MBMaskable? {
        return maskMine
    }
    
    func inset() -> UIEdgeInsets {
        return insetMine
    }
    
    func maskContainer() -> MBContainable? {
        return containerMine
    }
    
    func begin() {
        show()
    }
    
    func end() {
        hide()
    }
    
    var insetMine: UIEdgeInsets
    var maskMine: MBMaskable?
    var containerMine: MBContainable?
}
```

然后我们就可以这样使用它了。


``` swift
let load = LoadConfig(container: view, mask:MBEyeLoading(), inset: UIEdgeInsetsMake(30+64, 15, UIScreen.main.bounds.height-64-(44*4+30+15*3), 15))
request(WeatherForm()).load(load: load)
```

你会发现所有的东西都是可以自定义的，而且使用起来仍然很简单。

下面是利用 `LoadConfig` 在 `UITableView` 上显示自定义加载遮罩的的例子。

![](http://img.blog.csdn.net/20170109145905425?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```

let load = LoadConfig(container:self.tableView, mask: MBActivityIndicator(), inset: UIEdgeInsetsMake(UIScreen.main.bounds.width - self.tableView.contentOffset.y > 0 ? UIScreen.main.bounds.width - self.tableView.contentOffset.y : 0, 0, 0, 0))
request(WeatherForm()).load(load: load)
        
```
 
## 加载进度展示

进度的展示比较简单，只需要有方法实时更新进度即可，所以我们先定义 `MBProgressable` 协议，内容如下：

``` swift
public protocol MBProgressable {
    func progress(_ progress: Progress)
}
```

因为一般只有上传和下载大文件才需要进度展示，所以我们只对 `UploadRequest` 和 `DownloadRequest` 做 `extension`，添加 `progress` 方法，参数为遵循 `MBProgressable` 协议的 `progress` 对象 ：

``` swift
func progress(progress: MBProgressable) -> Self {
    return uploadProgress { (prog: Progress) in
        progress.progress(prog)
    }
}

```

### 常用控件支持

既然是进度展示，当然得让 `UIProgressView` 遵循 `MBProgressable` 协议，实现如下：

``` swift
// MARK: - Making `UIProgressView` conforms to `MBLoadProgressable`
extension UIProgressView: MBProgressable {

    /// Updating progress
    ///
    /// - Parameter progress: Progress object generated by network request
    public func progress(_ progress: Progress) {
        self.setProgress(Float(progress.completedUnitCount).divided(by: Float(progress.totalUnitCount)), animated: true)
    }
}

```

然后我们就可以直接把 `UIProgressView` 对象当做 `progress` 方法的参数了。

![](http://img.blog.csdn.net/20170109152204833?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

``` swift
download(ImageDownloadForm()).progress(progress: progress)
```

## 信息提示

信息提示包括两个部分，出错提示和成功提示。所以我们先抽象了一个 `MBMessageable` 协议，协议的内容仅仅包含了显示消息的容器。

``` swift
public protocol MBMessageable {
    func messageContainer() -> MBContainable?
}
```

毫无疑问，返回的容器当然也是遵循 `MBContainable` 协议的，这个容器将被用来展示出错和成功提示。

### 出错提示

出错提示需要做的事情有两步：
 
1. 解析错误信息
2. 展示错误信息

首先我们来完成第一步，解析错误信息。这里我们把错误信息抽象成协议 `MBErrorable`，其内容如下：

``` swift
public protocol MBErrorable {

    /// Using this set with code to distinguish successful code from error code
    var successCodes: [String] { get }

    /// Using this code with successCodes set to distinguish successful code from error code
    var code: String? { get }

    /// Corresponding message
    var message: String? { get }
}
```

其中 `successCodes` 用来定义哪些错误码是正常的； `code` 表示当前错误码；`message` 定义了展示给用户的信息。

具体怎么使用这个协议后面再说，我们接着看 JSON 错误解析协议 `MBJSONErrorable`。

``` swift
public protocol MBJSONErrorable: MBErrorable, Mappable {

}
```

注意这里的 `Mappable` 协议来自 `ObjectMapper`，目的是让遵循这个协议的对象实现 `Mappable` 协议中的 `func mapping(map: Map)` 方法，这个方法定义了 JSON 数据中错误信息到 `MBErrorable` 协议中 `code` 和 `message` 属性的映射关系。

假设服务端返回的 JSON 内容如下:

``` swift
{
    "data": {
        "code": "200",    
        "message": "请求成功"
    }
}
```

那我们的错误信息对象就可以定义成下面的样子。

``` swift
class WeatherError: MBJSONErrorable {
    var successCodes: [String] = ["200"]

    var code: String?
    var message: String?

    init() { }

    required init?(map: Map) { }

    func mapping(map: Map) {
        code <- map["data.code"]
        message <- map["data.message"]
    }
}
```

`ObjectMapper` 会把 `data.code` 和 `data.message` 的值映射到 `code` 和 `message` 属性上。至此，错误信息的解析就完成了。

然后是第二步，错误信息展示。定义 `MBWarnable` 协议：

``` swift
public protocol MBWarnable: MBMessageable {
    func show(error: MBErrorable?)
}
```

这个协议遵循 `MBMessageable` 协议。遵循这个协议的对象除了要实现 `MBMessageable` 协议的 `messageContainer` 方法，还需要实现 `show` 方法，这个方法只有一个参数，通过这个参数我们传入遵循错误信息协议的对象。

现在我们就可以使用 `MBErrorable` 和 `MBWarnable` 协议来进行出错提示了。和之前一样我们还是对 `DataRequest` 做 extension。添加 `warn` 方法。

``` swift
func warn<T: MBJSONErrorable>(
        error: T,
        warn: MBWarnable,
        completionHandler: ((MBJSONErrorable) -> Void)? = nil
        ) -> Self {

    return response(completionHandler: { (response: DefaultDataResponse) in
        if let err = response.error {
            warn.show(error: err.localizedDescription)
        }
    }).responseObject(queue: nil, keyPath: nil, mapToObject: nil, context: nil) { (response: DataResponse<T>) in
        if let err = response.result.value {
            if let code = err.code {
                if true == error.successCodes.contains(code) {
                    completionHandler?(err)
                } else {
                    warn.show(error: err)
                }
            }
        }
    }
}
```

这个方法包括三个参数：

- `error`：遵循 `MBJSONErrorable` 协议的泛型错误解析对象。传入这个对象到 `AlamofireObjectMapper` 的 `responseObject` 方法中即可获得服务端返回的错误信息。
- `warn`：遵循 `MBWarnable` 协议的错误展示对象。 
- `completionHandler`：返回结果正确时调用的闭包。业务层一般通过这个闭包来做特殊错误码处理。

做了如下的事情：

- 通过 `Alamofire` 的 `response` 方法获取非业务错误信息，如果存在，则调用 `warn` 的 `show` 方法展示错误信息，这里大家可能会有点疑惑：为什么可以把 `String` 当做 `MBErrorable` 传入到 `show` 方法中？这是因为我们做了下面的事情：

  ``` swift
  extension String: MBErrorable {
        public var message: String? {
            return self
        }
  }
  ```

- 通过 `AlamofireObjectMapper` 的 `responseObject` 方法获取到服务端返回的错误信息，判断返回的错误码是否包含在 `successCodes` 中，如果是，则交给业务层处理；（PS：对于某些需要特殊处理的错误码，也可以定义在 `successCodes` 中，然后在业务层单独处理。）否则，直接调用 `warn` 的 `show` 方法展示错误信息。 
 
### 成功提示

相比错误提示，成功提示会简单一些，因为成功提示信息一般都是在本地定义的，不需要从服务端获取，所以成功提示协议的内容如下：

``` swift
public protocol MBInformable: MBMessageable {
    func show()

    func message() -> String
}
```

包含两个方法， `show` 方法用于展示信息；`message` 方法定义展示的信息。

然后对 `DataRequest` 做扩展，添加 `inform` 方法：

``` swift
func inform<T: MBJSONErrorable>(error: T, inform: MBInformable) -> Self {

    return responseObject(queue: nil, keyPath: nil, mapToObject: nil, context: nil) { (response: DataResponse<T>) in
        if let err = response.result.value {
            if let code = err.code {
                if true == error.successCodes.contains(code) {
                    inform.show()
                }
            }
        }
    }
}
```

这里同样也传入遵循 `MBJSONErrorable` 协议的泛型错误解析对象，因为如果服务端的返回结果是错的，则不应该提示成功。还是通过 `AlamofireObjectMapper` 的 `responseObject` 方法获取到服务端返回的错误信息，判断返回的错误码是否包含在 `successCodes` 中，如果是，则通过 `inform` 对象 的 `show` 方法展示成功信息。

### 常用控件支持

观察目前主流 App，信息提示一般是通过 `UIAlertController` 来展示的，所以我们通过 extension 的方式让 `UIAlertController` 遵循 `MBWarnable` 和 `MBInformable` 协议。

``` swift
extension UIAlertController: MBInformable {
    public func show() {
        UIApplication.shared.keyWindow?.rootViewController?.present(self, animated: true, completion: nil)
    }
}

extension UIAlertController: MBWarnable{
    public func show(error: MBErrorable?) {
        if let err = error {
            if "" != err.message {
                message = err.message
                
                UIApplication.shared.keyWindow?.rootViewController?.present(self, animated: true, completion: nil)
            }
        }
    }
}
```

发现这里我们没有用到 `messageContainer`，这是因为对于 `UIAlertController` 来说，它的容器是固定的，使用 `UIApplication.shared.keyWindow?.rootViewController?` 即可。注意对于`MBInformable`，直接展示 `UIAlertController`， 而对于 `MBWarnable`，则是展示 `error` 中的 `message`。

下面是使用的两个例子：

![这里写图片描述](http://img.blog.csdn.net/20170109152218771?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20170109152233912?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW1vYWF5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

``` swift
let alert = UIAlertController(title: "Warning", message: "Network unavailable", preferredStyle: .alert)
alert.addAction(UIAlertAction(title: "Ok", style: UIAlertActionStyle.cancel, handler: nil))
        
request(WeatherForm()).warn(
    error: WeatherError(),
    warn: alert
)

let alert = UIAlertController(title: "Notice", message: "Load successfully", preferredStyle: .alert)
alert.addAction(UIAlertAction(title: "Ok", style: UIAlertActionStyle.cancel, handler: nil))
request(WeatherForm()).inform(
    error: WeatherInformError(),
    inform: alert
)
```

这样就达到了业务层定义展示信息，`MBNetwork` 自动展示的效果，是不是简单很多？至于扩展性，我们还是可以参照 `UIAlertController` 的实现添加对其它第三方提示库的支持。

## 重新请求

开发中……敬请期待



> 如有任何知识产权、版权问题或理论错误，还请指正。
>
> 转载请注明原作者及以上信息。
