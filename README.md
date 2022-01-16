# 感谢

创建VideoCacheX组件之前，要感谢之前长期使用国外大神的 AndroidVideoCache( https://github.com/danikula/AndroidVideoCache )，启蒙了自己如何实现边下边播的设计思路。

# 为何要写VideoCacheX

- 分析AndroidVideoCache(以下简称为AVC)：

1. AVC相对来说功能的耦合还是比较重的。当出现问题现象时，问题可能发生在任何一个环节当中。
2. AVC扩展性比较有限。因为AVC只负责处理将在线文件写到本地文件的事情，但是国内有多CDN的情况，如何做CDN调度切换等，在AVC中很不好植入，因为随时会影响原本的设计，容易出bug。
3. AVC的文件存储策略不是最优。使用RandomAccessFile方式写入方式，IO频繁，不是最优解。

# 通过使用方式扩展讲述VideoCacheX的特点


```kotlin
class MainActivity : FragmentActivity() {

    companion object {
        // 播放地址
        private const val url =
            "https://alimov2.a.yximgs.com/upic/2021/06/23/08/BMjAyMTA2MjMwODIwMjJfNTIzMjA2MTIzXzUxOTAxNjYxOTc2XzFfMw==_b_B83e0b595fb2bb54ff88fa81db21715a8.mp4?clientCacheKey=3xje85nzn89339e_b.mp4&tt=b&di=751ea781&bp=13380"
    }

    private lateinit var videoView: VideoView
    
    // ProxyServer是一个接口，开发者可自定义实现Socket代理服务。这边演示是使用内部提供的便捷Builder方案。
    private val cacheServer: ProxyServer by lazy {
        VideoCacheX.getProxyServer(
            ProxyServerBuilder.Builder(applicationContext)
                .cacheDirectory(applicationContext.externalCacheDir ?: applicationContext.cacheDir)
                .maxCacheSize(1024 * 1024 * 100) // 本地cache文件夹最大缓存字节数，超过会使用LRU按时间升序清理
                .build()
        )
    }
    // Request是一个类似OKHttp的Url Builder
    private val playRequest: Request by lazy {
        Request.fromSourceUrl(url)
            .build()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        videoView = findViewById(R.id.video_view)
        // 1.初始化VideoCacheX组件
        initVideoCacheX()

        videoView.setOnErrorListener { mp, what, extra ->
            Toast.makeText(applicationContext, "error:$what ,$extra ", Toast.LENGTH_SHORT).show()
            true
        }
        
        // 2.类似OKHttp一样发起边下边播请求
        cacheServer.newCall(playRequest) { proxyUrl ->
            // 3. 内部server启动完毕后，会callback一个本机代理的url，此时可以设置给播放器，就可以使得VideoView跟VideoCache建立Socket连接啦
            videoView.setVideoURI(Uri.parse(proxyUrl))
            videoView.start()
        }
    }

    private fun initVideoCacheX() {
        VideoCacheX.initGlobalConfig(
            // VideoCacheXConfig是一个Builder类，可对功能进行动态开/关。比如这边配置了`preload`，说明要开启视频预下载功能
            VideoCacheXConfig(applicationContext)
                .preload(
                    maxTaskCount = 2,//最大允许预下载任务数。如果传0则关闭预下载
                    maxThreads = 5, // 最大并发数,不超过最大任务数量
                    preloadBytes = 512 * 1024L, // 每次预下载多少字节数
                    overflowPolicy = PreloadOverflowPolicy.WAIT, // 当添加任务数已超过最大任务数时，采取排队等待策略等
                    order = PreloadOrder.FILO // 使用后进先出的排序，后发起下载请求的任务优先响应
                )
                .logEnable(true) // VideoCacheX的log功能
        )
    }

    override fun onDestroy() {
        super.onDestroy()
        cacheServer.stopRequest(playRequest)
        cacheServer.shutdown()
    }
}
```
## 核心功能描述

### 模板方法

VideoCacheX 采用`抽象流程`方式运作，内部提供了一个抽象流程模板（`abstract class AbsFlow`），为具体流程实现类提供好了5个方法模板: 整体流程开始、整体流程结束、每个子流程start、每个子流程结束以及子流程发生错误的回调。所以使用者根据不同场景，只需要继承AbsFlow即可，不需要额外处理工作流程事情。示例:

```kotlin
class CustomFlow :  AbsFlow  {

    override fun onChainStart(chain: Chain) {
        // 当某个子流程启动时，会实现类会收到该回调
    }

    override fun onChainComplete(chain: Chain) {
         // 当某个子流程完成时，会实现类会收到该回调
    }

    override fun onChainInterrupt(chain: Chain, flowExecuteListener: OnFlowExecuteListener) {
        // 当某个子流程正常/异常中断时，会实现类会收到该回调，可以做一些错误上报、流程重试等策略
    }
}
```

### 责任链模式

为了将流程中每个子流程进行解耦，我设计成责任链(`Chain`)方式，每个Chain负责单独任务并将结果传递给下一个Chain。

```kotlin
class CustomFlow : AbsFlow {

    // 抽象流程模板提供了一套很简单的chain数据，能实现针对原始文件进行边下边播功能。但是如果涉及到有CDN调度需求时，就需要重写chain，并插入一个ExchangeUrlChain.
    override val chain: Chain = HeadRequestChain(context, this) // 获取原始文件的Content-Length模块
        .next(ExchangeUrlChain(context, this)) // CDN调度切换模块
        .next(QueryFileSizeChain(context, this)) // 获取最终CDN调度后的文件Content-Length模块
        .next(DownloadFileChain(context, this, cacheDir, fileStoragePool)) // 使用CDN调度url边下边播模块
    }
}
```

### 命令模式

每个Chain任务很独立，但是总会遇到某个Chain要抛出错误信息来中断播放器进行播放的功能时，就需要设计一套消息机制。因此我在`Chain`类中设计一个信号量，示例:

```kotlin
object VideoSignal {
    /**
     *  无法获取文件基础信息，导致播放器后续无法发起socket请求
     */
    const val SIGNAL_LOST_FILE_INFO = 1
}
// Chain
open class Chain(...) {
    internal var signal: Int = VideoSignal.SIGNAL_NOTHING
}
// AbsFlow
abstract AbsFlow {
    override fun onChainInterrupt(chain: Chain, flowExecuteListener: OnFlowExecuteListener) {
        when (chain.signal) {
            VideoSignal.SIGNAL_LOST_FILE_INFO -> {
                // 比如遇到文件404等异常，抽象流程回调就会收到signal信息，来做决策
                flowExecuteListener.onFileReadError()
            }
        }
    }
}
```

### 桥接模式

每个Chain任务都很独立，但也会存在需要互相交互的情况。比如 `ExchangeUrlChain`和 `DownloadFileChain`。假设下载过程中突然遇到url过期了，需要重新换个CDN情况，就需要相互通信了。
为此，我为每个`Chain`设计了一套`Bridge`方案，将各种case的具体实现放在2个chain之外，避免污染原有chain的独立逻辑。
