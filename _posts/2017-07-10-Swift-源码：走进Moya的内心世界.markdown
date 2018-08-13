---
title:  "源码：走进Moya的内心世界"
date:   2017-07-10 17:46:01
categories: Swift 
---

<div align="center">
	<img style="height:160px" src="https://github.com/Moya/Moya/raw/master/web/logo_github.png" />
</div>

[Moya](https://github.com/Moya/Moya)是一个基于[Alamofire](https://github.com/Alamofire/Alamofire)的网络层封装，让我们不用关心Alamofire的内部实现，相对于为我们提供了更高等级的API。Moya在业务解耦，API管理，测试等方面都有不错的表现。








>注意：本文默认你已经熟悉Moya的基本使用，内容不会按照使用教学的顺序来，所以如果你还没有使用过Moya，那么建议你下载[Moya](https://github.com/Moya/Moya) 的Demo看看，或者去查看它的[文档](https://github.com/Moya/Moya/tree/master/docs)，内容还是很详细的。

## MoyaProvider

MoyaProvider是Moya的基础，它是你API的端点的管理者。Moya的所有功能都是通过MoyaProvider来使用。首先我们先来看一下它的定义：

```swift
/// Request provider class. Requests should be made through this class only.
open class MoyaProvider<Target: TargetType> {

    /// Closure that defines the endpoints for the provider.
    public typealias EndpointClosure = (Target) -> Endpoint<Target>

    /// Closure that decides if and what request should be performed
    public typealias RequestResultClosure = (Result<URLRequest, MoyaError>) -> Void

    /// Closure that resolves an `Endpoint` into a `RequestResult`.
    public typealias RequestClosure = (Endpoint<Target>, @escaping RequestResultClosure) -> Void

    /// Closure that decides if/how a request should be stubbed.
    public typealias StubClosure = (Target) -> Moya.StubBehavior

    open let endpointClosure: EndpointClosure
    open let requestClosure: RequestClosure
    open let stubClosure: StubClosure
    open let manager: Manager

    /// A list of plugins
    /// e.g. for logging, network activity indicator or credentials
    open let plugins: [PluginType]

    open let trackInflights: Bool

    open internal(set) var inflightRequests: [Endpoint<Target>: [Moya.Completion]] = [:]

    /// Initializes a provider.
    public init(endpointClosure: @escaping EndpointClosure = MoyaProvider.defaultEndpointMapping,
                requestClosure: @escaping RequestClosure = MoyaProvider.defaultRequestMapping,
                stubClosure: @escaping StubClosure = MoyaProvider.neverStub,
                manager: Manager = MoyaProvider<Target>.defaultAlamofireManager(),
                plugins: [PluginType] = [],
                trackInflights: Bool = false) {

        self.endpointClosure = endpointClosure
        self.requestClosure = requestClosure
        self.stubClosure = stubClosure
        self.manager = manager
        self.plugins = plugins
        self.trackInflights = trackInflights
    }

    /// Returns an `Endpoint` based on the token, method, and parameters by invoking the `endpointClosure`.
    open func endpoint(_ token: Target) -> Endpoint<Target> {
        return endpointClosure(token)
    }

    /// Designated request-making method. Returns a `Cancellable` token to cancel the request later.
    @discardableResult
    open func request(_ target: Target, completion: @escaping Moya.Completion) -> Cancellable {
        return self.request(target, queue: nil, completion: completion)
    }

    /// Designated request-making method with queue option. Returns a `Cancellable` token to cancel the request later.
    @discardableResult
    open func request(_ target: Target, queue: DispatchQueue?, progress: Moya.ProgressBlock? = nil, completion: @escaping Moya.Completion) -> Cancellable {
        return requestNormal(target, queue: queue, progress: progress, completion: completion)
    }

    /// When overriding this method, take care to `notifyPluginsOfImpendingStub` and to perform the stub using the `createStubFunction` method.
    /// Note: this was previously in an extension, however it must be in the original class declaration to allow subclasses to override.
    @discardableResult
    open func stubRequest(_ target: Target, request: URLRequest, completion: @escaping Moya.Completion, endpoint: Endpoint<Target>, stubBehavior: Moya.StubBehavior) -> CancellableToken {
        let cancellableToken = CancellableToken { }
        notifyPluginsOfImpendingStub(for: request, target: target)
        let plugins = self.plugins
        let stub: () -> Void = createStubFunction(cancellableToken, forTarget: target, withCompletion: completion, endpoint: endpoint, plugins: plugins, request: request)
        switch stubBehavior {
        case .immediate:
            stub()
        case .delayed(let delay):
            let killTimeOffset = Int64(CDouble(delay) * CDouble(NSEC_PER_SEC))
            let killTime = DispatchTime.now() + Double(killTimeOffset) / Double(NSEC_PER_SEC)
            DispatchQueue.main.asyncAfter(deadline: killTime) {
                stub()
            }
        case .never:
            fatalError("Method called to stub request when stubbing is disabled.")
        }

        return cancellableToken
    }
}
```
代码比较长，我们分开来进行分析。

### 定义
首先我们来看看MoyaProvider的定义如下：

```swift
open class MoyaProvider<Target: TargetType> {}
```
可以看到，MoyaProvider是一个使用了`泛型`的类。接收一个遵守[TargetType](#0)的类型。

## MoyaProvdier的属性
MoyaProvdier有如下属性：
### EndpointClosure

```swift
/// Closure that defines the endpoints for the provider.
    public typealias EndpointClosure = (Target) -> Endpoint<Target>
```
EndpointClosure属性是一个闭包，用于让我们对Moya生成的[Endpoint](#1)进行一些我们自己的定制然后返回一个Endpoint类，例如：我们想增加一个新的HttpHeader：

```swift
let endpointClosure = { (target: MyTarget) -> Endpoint<MyTarget> in
    let defaultEndpoint = MoyaProvider.defaultEndpointMapping(for: target)
    return defaultEndpoint.adding(newHTTPHeaderFields: ["APP_NAME": "MY_AWESOME_APP"])
}
let provider = MoyaProvider<GitHub>(endpointClosure: endpointClosure)
```
通过闭包对属性的操作可以很大程度上明确我们的代码结构并减少不必要的coding。

### RequestClosure && RequestResultClosure
```swift
/// Closure that decides if and what request should be performed
    public typealias RequestResultClosure = (Result<URLRequest, MoyaError>) -> Void
```
同上，这是个闭包属性，入参的类型可能有小伙伴不熟悉，这是使用了[Result](https://github.com/antitypical/Result)框架,用来将`throw`的方式换成`Result<data,error>`的方式返回。

利用这个闭包我们可以进行请求映射，在请求发起之前修改我们的请求然后将一个RequestResultClosure回调出去，例如：

```swift
let requestClosure = { (endpoint: Endpoint<GitHub>, done: MoyaProvider.RequestResultClosure) in
    var request: URLRequest = endpoint.urlRequest
    request.httpShouldHandleCookies = false
    done(.success(request))
}
let provider = MoyaProvider<GitHub>(requestClosure: requestClosure)
```
这样就让我们的请求不处理Cookies了.

## MoyaProvider的方法
具体的方法在文档和demo中都有体现，就不多讲了，都是基础的调用。

## MoyaProvider -> Request
重点来了，我们准备了那么多是不是要发请求了。我们配置的这些东西都是怎么工作的？下面我们走进Moya的内心世界

### MoyaProvider -> EndPoint
```swift
public final class func defaultEndpointMapping(for target: Target) -> Endpoint<Target> {
        return Endpoint(
            url: url(for: target).absoluteString,
            sampleResponseClosure: { .networkResponse(200, target.sampleData) },
            method: target.method,
            parameters: target.parameters,
            parameterEncoding: target.parameterEncoding
        )
    }
```
首先MoyaProvider通过这个方法生成了一个端点(Endpoint)。

### requestMap
随后Moya内部映射出一个URLRequest

```swift
public final class func defaultRequestMapping(for endpoint: Endpoint<Target>, closure: RequestResultClosure) {
        if let urlRequest = endpoint.urlRequest {
            closure(.success(urlRequest))
        } else {
            closure(.failure(MoyaError.requestMapping(endpoint.url)))
        }
    }
```

### MoyaProvider().request
代码很长，通过注释来看

```swift
func requestNormal(_ target: Target, queue: DispatchQueue?, progress: Moya.ProgressBlock?, completion: @escaping Moya.Completion) -> Cancellable {
        
        //生成端点
        let endpoint = self.endpoint(target)
        //生成测试表现
        let stubBehavior = self.stubClosure(target)
        //生成取消请求的Token
        let cancellableToken = CancellableWrapper()

        // 通过自定义的插件完善请求回调的闭包
        let pluginsWithCompletion: Moya.Completion = { result in
            let processedResult = self.plugins.reduce(result) { $1.process($0, target: target) }
            completion(processedResult)
        }
        //是否追踪运行中的请求
        if trackInflights {
            //进入原子操作
            objc_sync_enter(self)
            //保存请求端点
            var inflightCompletionBlocks = self.inflightRequests[endpoint]
            inflightCompletionBlocks?.append(pluginsWithCompletion)
            self.inflightRequests[endpoint] = inflightCompletionBlocks
            //退出原子操作
            objc_sync_exit(self)
            //如果用正在运行中的CompletionBlock则取消本次请求，反之则记录本次请求
            if inflightCompletionBlocks != nil {
                return cancellableToken
            } else {
                objc_sync_enter(self)
                self.inflightRequests[endpoint] = [pluginsWithCompletion]
                objc_sync_exit(self)
            }
        }
        //准备请求工作
        let performNetworking = { (requestResult: Result<URLRequest, MoyaError>) in
            //如果本次请求被取消了则返回
            if cancellableToken.isCancelled {
                self.cancelCompletion(pluginsWithCompletion, target: target)
                return
            }

            var request: URLRequest!
            //模式匹配取出request
            switch requestResult {
            case .success(let urlRequest):
                request = urlRequest
            case .failure(let error):
                pluginsWithCompletion(.failure(error))
                return
            }
            
            // 通过自定义插件完善请求的闭包
            let preparedRequest = self.plugins.reduce(request) { $1.prepare($0, target: target) }
            
            //根据测试行为定制回调
            switch stubBehavior {
            case .never:
                let networkCompletion: Moya.Completion = { result in
                    if self.trackInflights {
                        //移除追踪队列中的该请求
                        self.inflightRequests[endpoint]?.forEach { $0(result) }

                        objc_sync_enter(self)
                        self.inflightRequests.removeValue(forKey: endpoint)
                        objc_sync_exit(self)
                    } else {
                        pluginsWithCompletion(result)
                    }
                }
                //根据请求类型来通过Alamofire发起请求。
                switch target.task {
                case .request:
                    cancellableToken.innerCancellable = self.sendRequest(target, request: preparedRequest, queue: queue, progress: progress, completion: networkCompletion)
                case .upload(.file(let file)):
                    cancellableToken.innerCancellable = self.sendUploadFile(target, request: preparedRequest, queue: queue, file: file, progress: progress, completion: networkCompletion)
                case .upload(.multipart(let multipartBody)):
                    guard !multipartBody.isEmpty && target.method.supportsMultipart else {
                        fatalError("\(target) is not a multipart upload target.")
                    }
                    cancellableToken.innerCancellable = self.sendUploadMultipart(target, request: preparedRequest, queue: queue, multipartBody: multipartBody, progress: progress, completion: networkCompletion)
                case .download(.request(let destination)):
                    cancellableToken.innerCancellable = self.sendDownloadRequest(target, request: preparedRequest, queue: queue, destination: destination, progress: progress, completion: networkCompletion)
                }
            default:
                cancellableToken.innerCancellable = self.stubRequest(target, request: preparedRequest, completion: { result in
                    if self.trackInflights {
                        self.inflightRequests[endpoint]?.forEach { $0(result) }

                        objc_sync_enter(self)
                        self.inflightRequests.removeValue(forKey: endpoint)
                        objc_sync_exit(self)
                    } else {
                        pluginsWithCompletion(result)
                    }
                }, endpoint: endpoint, stubBehavior: stubBehavior)
            }
        }
        //调用我们队request的自定义设置
        requestClosure(endpoint, performNetworking)
		 //返回一个可以被取消的请求Token
        return cancellableToken
    }
```

<h1 id="0"></h1>
## TargetType
TargetType是一个协议，它要求遵守者提供一些只读属性，这些属性正好是我们网络请求所需要的参数元素。
属性如下：

* BaseURL
* Path
* Method 
* Parameters
* ParameterEncoding
* SampleData (测试打桩时返回的数据)
* Task （request、upload、download）
* Validate （是否对参数进行校验 ，一般有手机号、邮箱号校验之类的）

上面这些参数大家都不陌生，都是网络请求的基础要求参数。这样的设计使得我们在调用网络请求的时候不需要在方法中写上那么多入参。如：
common:

```swift
MyAPI.request(url:URL,parameters:[String:Any],method:Method)
```
Moya:

```swift
MoyaProvider<MyAPI>().request()
```
如果参数很多的话简直是灾难，不得不说这个设计很赞。

值得一提的是这里加入了两个很有意思的参数`SampleData`和`validate`,`SampleData`可以让我们在进行测试插桩的时候，剩下Mock返回数据的代码，只要我们设置了属性之后在测试的时候会自动返回。这样的设计真的很巧妙，让我们的测试变得更加轻松。（这也是我为什么选择使用Moya的原因，当然原因不止如此），另一个属性就是`Valiadate`，以前我们想要验证参数的合法性，可能会有如下方法：

* 在调用请求之前先校验，如果不合法就return
* 把一个这样的闭包：**(你的参数) -> (验证结果)**当做一个参数传进request方法中。（我就是这样做的）

如上两个方法都有各自的缺点。来看Moya的思路：

```swift
public var validate: Bool {
	let phoneNumber:String = self.parameters["phoneNumber"]
    return phoneNumber.characters.count == 11
}
```
通过闭包返回一个校验的值，一次性配置好以后都不用改了。实在是很便利。当然，看到这里你也许会有个疑惑，既然这个属性的值是`固定的`，那么是不是每一个API都要创建一个实例然后遵守TargetType，然后为了缩减代码还要创建一个`BaseAPI`之类的东西让其他的API对象来继承它？
NONONO~，如果说使用泛型给MoyaProvider的灵活性奠定了基础，那么使用枚举Enum来定义管理API的方式就给Moya的灵活赋予了灵性。

## 使用Enum来管理/定义我们的APIs
来看看Demo中的例子，Demo中有一个专门请求GithubAPI的模块。来看看是怎么定义这些API的：

```swift
public enum GitHub {
    case zen
    case userProfile(String)
    case userRepositories(String)
}
```
通过枚举我们可以很清晰看出Giuhub模块中有哪些API，这样可以使我们的业务逻辑变得更加清晰。使用Enum的关联值来表示各个API的具体参数。这样我们是不是以后不需要再写注释和看文档了。😂好吧~其实还是要看文档的，当然这样的写法在一定程度上免去了我们可能会出现的困惑，也许几个月过去了，回头看代码突然忘了这个api的参数的时候，可以很明白的看出它是什么类型和作用。写到这里请让我感叹一句：`Swift大法好！`

更多的枚举高级用法请看Swiftgg翻译的文章：[Swift 中枚举高级用法及实践](http://swift.gg/2015/11/20/advanced-practical-enum-examples/)。这篇文章我也读过，受益颇多，感谢Swiftgg翻译组。

<h1 id="1"></h1>
## 端点（Endpoint）

我们通过Moyaprovider实例而产生的一个API端点。通过这个端点我们可以发起请求，有点像AFN的`AFNSession`,借用Moya的文档原图，一次请求应该是这样发起的：

![Pipeline](https://raw.github.com/Moya/Moya/master/web/pipeline.png)

`Target`很好理解就是我们之前说的带着各种参数属性的泛型对象。通过它，Moya进一步生成了与之对应的API Endpoint，随后通过`Endpoint`发起`Requset`.

那么一个EndPoint包含什么呢？它包含着我们这次请求的所有信息，包括：
* url
* method
* parameters
* httpHeaderFields
* 等等

### Endpoint.request
```swift
/// Extension for converting an `Endpoint` into an optional `URLRequest`.
extension Endpoint {
    /// Returns the `Endpoint` converted to a `URLRequest` if valid. Returns `nil` otherwise.
    public var urlRequest: URLRequest? {
        guard let requestURL = Foundation.URL(string: url) else { return nil }

        var request = URLRequest(url: requestURL)
        request.httpMethod = method.rawValue
        request.allHTTPHeaderFields = httpHeaderFields

        return try? parameterEncoding.encode(request, with: parameters)
    }
}
```
在这段代码可以看到，Endpoint使用其内部的基本属性，将自己转换成了一个`URLRequest?`对象。

### EndpointA == EndpointB ?
```swift
/// Required for using `Endpoint` as a key type in a `Dictionary`.
extension Endpoint: Equatable, Hashable {
    public var hashValue: Int {
        return urlRequest?.hashValue ?? url.hashValue
    }

    public static func == <T>(lhs: Endpoint<T>, rhs: Endpoint<T>) -> Bool {
        if lhs.urlRequest != nil, rhs.urlRequest == nil { return false }
        if lhs.urlRequest == nil, rhs.urlRequest != nil { return false }
        if lhs.urlRequest == nil, rhs.urlRequest == nil { return lhs.hashValue == rhs.hashValue }
        return (lhs.urlRequest == rhs.urlRequest)
    }
}
```
遵守`Equatable`和`Hashable`协议并实现响应方法即可为自己的类提供`==`比较方法.这个对比方法用来追踪已经在请求中的request。排除多次无用请求。
### 可定制方法
其内部提供了几个API让我们可以为Endpoint新增`HttpHeader`、`NewParameter`、`newParameterEncoding`

### SampleResponseClosure
Endpoint有一个叫做SampleResponseClosure的属性，和之前的闭包类型参数差不多，这个参数用来返回定制的测试networkResponse。比如特定的`StautsCode`之类的。

```swift
/// Used for stubbing responses.
public enum EndpointSampleResponse {

    /// The network returned a response, including status code and data.
    case networkResponse(Int, Data)

    /// The network returned response which can be fully customized.
    case response(HTTPURLResponse, Data)

    /// The network failed to send the request, or failed to retrieve a response (eg a timeout).
    case networkError(NSError)
}
```
## Cancellable

```swift
public protocol Cancellable {
    var isCancelled: Bool { get }
    func cancel()
}

internal class CancellableWrapper: Cancellable {
    internal var innerCancellable: Cancellable = SimpleCancellable()

    var isCancelled: Bool { return innerCancellable.isCancelled }

    internal func cancel() {
        innerCancellable.cancel()
    }
}

internal class SimpleCancellable: Cancellable {
    var isCancelled = false
    func cancel() {
        isCancelled = true
    }
}

public final class CancellableToken: Cancellable, CustomDebugStringConvertible {
    let cancelAction: () -> Void
    let request: Request?
    public fileprivate(set) var isCancelled = false

    fileprivate var lock: DispatchSemaphore = DispatchSemaphore(value: 1)

    public func cancel() {
        _ = lock.wait(timeout: DispatchTime.distantFuture)
        defer { lock.signal() }
        guard !isCancelled else { return }
        isCancelled = true
        cancelAction()
    }

    public init(action: @escaping () -> Void) {
        self.cancelAction = action
        self.request = nil
    }

    init(request: Request) {
        self.request = request
        self.cancelAction = {
            request.cancel()
        }
    }

    public var debugDescription: String {
        guard let request = self.request else {
            return "Empty Request"
        }
        return request.debugDescription
    }

}

```
Cancellable赋予了遵守者可以取消请求的方法。

## StubBehavior
```swift
public enum StubBehavior {

    /// Do not stub.
    case never

    /// Return a response immediately.
    case immediate

    /// Return a response after a delay.
    case delayed(seconds: TimeInterval)
}
```
这个不用多说，测试插桩时集中请求表现。

## 一些类型别名
Moya是基于AF的，为了代码统一，所以针对一些AF的属性做了`typealias` 在`Moya+Alamofire`文件中可以看到：

```swift
public typealias Manager = Alamofire.SessionManager
internal typealias Request = Alamofire.Request
internal typealias DownloadRequest = Alamofire.DownloadRequest
internal typealias UploadRequest = Alamofire.UploadRequest
internal typealias DataRequest = Alamofire.DataRequest

internal typealias URLRequestConvertible = Alamofire.URLRequestConvertible

/// Represents an HTTP method.
public typealias Method = Alamofire.HTTPMethod

/// Choice of parameter encoding.
public typealias ParameterEncoding = Alamofire.ParameterEncoding
public typealias JSONEncoding = Alamofire.JSONEncoding
public typealias URLEncoding = Alamofire.URLEncoding
public typealias PropertyListEncoding = Alamofire.PropertyListEncoding

/// Multipart form
public typealias RequestMultipartFormData = Alamofire.MultipartFormData

/// Multipart form data encoding result.
public typealias MultipartFormDataEncodingResult = Manager.MultipartFormDataEncodingResult
public typealias DownloadDestination = Alamofire.DownloadRequest.DownloadFileDestination
```


