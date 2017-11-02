## KTVHTTPCache - 音视频在线播放缓存框架

KTVHTTPCache 是一个处理 HTTP 网络缓存的框架。设计之初是为了解决音视频在线播放的缓存问题。但其应用场景不仅限于音视频在线播放，也可以用于图片加载、文件下载、普通网络请求等场景。


### 技术背景

对于有重度音视频在线播放需求的应用，缓存无疑是必不可少的功能。目前常用的方案有 Local HTTP Server 和 AVAssetResourceLoader 两种。二者实现及原理虽有不同，但本质都是要 Hook 到播放器资源加载的请求，从而接管资源加载逻辑。根据缓存状态，自行决定是否需要通过网络加载资源。从应用场景的角度看，二者有一个比较大的差异是前者可以搭配任意前端播放器，而后者只能配合 AVPlayer 使用。

个人认为，由于 AVAssetResourceLoader 是黑盒且会干预 AVPlayer 本身的播放逻辑，导致坑多且难排查。并且不同的版本之间会有行为差异（例如近期发现在最新的 iOS 11 系统中，原本工作正常的代码，因为一个细小的行为变化，引发了一个 Bug），去适配它的逻辑会有不小的工作量。相反 Local HTTP Server 是完全 Open Source，我们能够全面接管资源加载逻辑，可以尽可能的规避缓存策略的引入带来的风险。


### 功能特点

- 支持相同 URL 并发操作且线程安全。
- 全路径 Log，支持控制台打印和输出到文件，可准确定位问题。
- 细粒度的缓存管理，可精确查看指定 URL 的完整缓存信息。
- 模块相互独立，提供使用不同 Level 的接口。
- 下载层高度可配置。
- 低耦合，集成简单。


###  结构设计 & 工作流程

KTVHTTPCache 由 HTTP Server 和 Data Storage 两大模块组成。前者负责与 Client 交互，后者负责资源加载及缓存处理。为方便拓展，Data Storage 为独立模块，也可直接与 Client 交互（例如可与 AVAssetResourceLoader 配合使用）。

##### 结构及工作流程图如下：

![KTVHTTPCache Flow Chart](http://oxl6mxy2t.bkt.clouddn.com/changba/KTVHTTPCache-flow-chart.jpeg)

##### 下面简述一下工作流程：
1. Client 发出的请求被 HTTP Srever 接收到，HTTP Server 通过分析 HTTP Request 创建用于访问 Data Storage 的 Data Request 对象。
2. HTTP Server 使用 Data Request 创建 Data Reader，并以此作为从 Data Storage 获取数据的通道。
3. Data Reader 分析 Data Request 中的 Range 创建对应的网络数据源 Data Network Source 和文件数据源 Data File Source，并通过 Data Sourcer 进行管理。
4. Data Sourcer 开始加载数据。
5. Data Reader 从 Data Sourcer 读取数据并通过 HTTP Server 回传给 Client。


### 缓存策略
以网络使用最小化为原则，设计了分片加载数据的功能。有 Network Source 和 File Source 两种用于加载数据的 Source，分别用于下载网络数据和读取本地数据。通过分析 Data Request 的 Range 和本地缓存状态来对应创建。

例如一次请求的 Range 为 0-999，本地缓存中已有 200-499 和 700-799 两段数据。那么会对应生成 5 个 Source，分别是：
1. Data Network Source: 0-199
2. Data File Source: 200-499
3. Data Network Source: 500-699
4. Data File Source: 700-799
5. Data Network Source: 800-999

它们由 Data Sourcer 进行管理，对外仅暴露一个 Read Data 的接口，根据当前的 Read Offset 自行选择向外界提供数据的 Source。


### 使用示例

```
// 使用简单，基本可以忽略集成成本

// 启动（全局启动一次即可）
NSError * error;
[KTVHTTPCache proxyStart:&error];

// 使用
NSString * URLString = [KTVHTTPCache proxyURLStringWithOriginalURLString:@"原始 URL"];
AVPlayer * player = [AVPlayer playerWithURL:[NSURL URLWithString:URLString]]
```


### 最后

- GitHub 地址：https://github.com/ChangbaDEV/KTVHTTPCache
- 如果遇到任何问题可以在 GitHub 提 Issue 给我。