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
    private var isPrepared = false

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
        val mediaController = MediaController(this)
        mediaController.setMediaPlayer(videoView)
        videoView.setMediaController(mediaController)

        videoView.setOnPreparedListener {
            isPrepared = true
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

    override fun onResume() {
        super.onResume()
        videoView.resume()
    }

    override fun onPause() {
        super.onPause()
        videoView.pause()
    }

    override fun onDestroy() {
        super.onDestroy()
        videoView.stopPlayback()
        cacheServer.stopRequest(playRequest, forceStopClient = true)
        cacheServer.shutdown()
    }
}
```
## 核心功能描述

### 模板方法


### 责任链模式


### 命令模式
