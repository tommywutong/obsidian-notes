---
title: 【iOS】SDWebImage：下载、解码与两级缓存的完整链路
published: 2026-07-27
description: 把 5.21.7 整个编进一个 Catalyst 二进制跑起来。强制解码并不减少总耗时，它只是把 100 ms 从主线程挪走；真正省内存的是 downsample，同一张图 46.51 MB 变 0.458 MB。
tags:
  - iOS
  - SDWebImage
  - 图片
  - 缓存
  - 解码
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 34
draft: true
---
# SDWebImage：下载、解码与两级缓存的完整链路

我去年在 CSDN 写过一篇《SDWebImage 解析》。这次准备写源码级的续篇，第一件事是把 5.21.7 拉下来，拿我那篇里出现过的类名和方法名逐个 grep：

```text
SDImageCacheDelegate                       （无匹配）
SDWebImageDecoder                          （无匹配）
NSURLConnection                            （无匹配）
imageCache:didFindImage                    （无匹配）
notifyDelegate                             （无匹配）
shouldDecompressImages                     （无匹配）
maxCacheAge                                （无匹配）
maxCacheSize                               （无匹配）
SDCacheCostForImage                        （无匹配）
makeDiskCachePath                          （无匹配）
```

十个，一个都不在了。这不怪我。中文圈讲 SDWebImage 流程的文章基本都是从 2015 年那批 3.x 解析里传下来的。我抄了流程，代码块却贴的是 5.x 的现役实现，两边讲的压根不是一个东西。**那批文章描述的委托回调链、`SDWebImageDecoder`、`NSURLConnection`，在今天的源码里一个符号都对不上。**

所以这一篇只做两件事：跟着 5.21.7 的源码把链路走一遍，每一跳标出文件和行号；能测的全部测出真数字。

先把最有价值的两条结论摆在前面。第一，强制解码不减少总工作量，它把 100 ms 量级的解码从主线程搬到后台队列（第二节有耗时对照）。第二，同一张 4032×3024 的 JPEG，磁盘上 1.04 MB，全解码进内存是 46.51 MB。加一个 `SDWebImageContextImageThumbnailPixelSize` 之后是 0.458 MB。

---

## 一、链路：跟着源码走一遍，顺便验一遍队列

我没有靠读代码猜队列。SDWebImage 的三大组件在 5.x 都协议化了。`SDWebImageContextImageCoder` 和 `SDWebImageContextImageTransformer` 可以直接塞自己的实现进去。于是我写了两个只做转发、打印当前队列 label 的探针，编成 Mac Catalyst 二进制跑真实链路。输出：

```text
主线程发起 loadImageWithURL:（当前 主线程）
  [progress]     在 com.hackemist.SDWebImageDownloader.downloadQueue
  [coder]        在 com.hackemist.SDWebImageDownloaderOperation.coderQueue
  [transformer]  在 com.apple.root.user-initiated-qos
  [completed]    在 主线程
```

四个队列，和源码对得上。下面这张图每一跳都标了文件行号，是照着 5.21.7 走出来的：

```text
[imageView sd_setImageWithURL:url]
  │                                          UIImageView+WebCache.m:49 → :55
  ▼
-[UIView sd_internalSetImageWithURL:…]       UIView+WebCache.m:58          主线程
  │  按 operationKey 取消上一次任务、设占位图
  ▼
-[SDWebImageManager loadImageWithURL:…]      SDWebImageManager.m:189       主线程
  │  :245
  ▼
callCacheProcessForOperation                 SDWebImageManager.m:283
  │  :304  key = [self cacheKeyForURL:url context:context]
  │  :306  [imageCache queryImageForKey:…]
  ▼
-[SDImageCache queryCacheOperationForKey:…]  SDImageCache.m:576
  │  :595  先查内存，同步，就在调用线程上跑
  │        命中 → :621 doneBlock(image, nil, Memory)   全程没离开主线程
  │  :692  未命中 → dispatch_async(self.ioQueue)
  │        ioQueue = "com.hackemist.SDImageCache.ioQueue"   SDImageCache.m:131
  │  :674  diskImageForKey:data:options:context: 在 ioQueue 上解码
  │  :701  回调经 SDCallbackQueue.mainQueue 切回主线程
  ▼  两级都没有
callDownloadProcessForOperation              SDWebImageManager.m:403
  │  :449  [imageLoader requestImageWithURL:…]
  ▼
-[SDWebImageDownloader downloadImageWithURL:…]
  │  :222  查 URLOperations[url] 有没有在跑的同 URL operation
  │  :262  有 → 只往它身上挂一组 callback，不发新请求
  │  :233  无 → 建 operation，:257 [self.downloadQueue addOperation:]
  ▼
SDWebImageDownloaderOperation                downloadQueue，并发上限 6
  │  NSURLSession 收完数据
  │  :367  [self.coderQueue addOperationWithBlock:]
  │        coderQueue 是串行的（:117 maxConcurrentOperationCount = 1）
  │  :392  SDImageLoaderDecodeImageData(…)
  ▼
SDImageLoaderDecodeImageData                 SDImageLoader.m:36
  │  :74   [imageCoder decodedImageWithData:options:]     ImageIO 出一个 CGImage
  │  :89   [SDImageCoderHelper decodedImageWithImage:policy:]   强制解码
  ▼
回到 manager 的 completed block              SDWebImageManager.m:449
  │  :475 → callTransformProcessForOperation
  │         :525 transform 在 global HIGH 队列
  │  :552 / :607 存原图缓存 / 存最终缓存
  │         SDImageCache.m:254  写内存，同步
  │         SDImageCache.m:300  dispatch_async(ioQueue) 写磁盘
  ▼
completedBlock                               SDCallbackQueue.mainQueue → 主线程
  │  sd_setImage: 设置 image、跑 transition
  ▼
imageView.image = image
```

有一处容易看漏。内存命中的那条路径完全同步，`queryCacheOperationForKey:` 的第 595 行直接在调用线程上查 `NSCache`，命中就在第 621 行原地调 `doneBlock` 并 `return nil`。我实测第二次加载耗时 0.03 ms，回调线程还是主线程。所以"SDWebImage 都是异步的"这句话不准，内存命中这一支是同步的，而且这正是它能撑住列表滚动的原因。

`loadImageWithURL:` 上方有一段注释，把整条流水线写死了（SDWebImageManager.m:232-244），值得抄原文：

```objc
// Steps without transformer:
// 1. query image from cache, miss
// 2. download data and image
// 3. store image to cache

// Steps with transformer:
// 1. query transformed image from cache, miss
// 2. query original image from cache, miss
// 3. download data and image
// 4. do transform in CPU
// 5. store original image to cache
// 6. store transformed image to cache
```

带 transformer 时是两次缓存查询、两次缓存写入。老文章里那张"查内存 → 查磁盘 → 下载"的三段图，对应的只是上面那个不带 transformer 的分支。

还有一处磁盘命中后的写回，位置和很多人记的不一样。它不在 `SDImageCache` 里，在 manager 里（SDWebImageManager.m:322-327）：

```objc
} else {
    // Write back the disk image into memory cache, with the correct key
    if (cacheType == SDImageCacheTypeDisk) {
        // Sync
        [imageCache storeImage:cachedImage imageData:nil forKey:key cacheType:SDImageCacheTypeMemory completion:nil];
    }
}
```

注释里 "with the correct key" 那句是关键。缩略图和 transform 都会改写 cache key，只有 manager 手上才有最终那个 key。2020 年前后那批文章贴的 `queryCacheOperationForKey:done:` 是在 `SDImageCache` 内部就 `[self.memCache setObject:diskImage forKey:key]`，那时候还没有 thumbnail 这一层。

---

## 二、解码：全文最重要的一节

### 强制解码到底在防什么

`decodedImageWithImage:` 的存在理由，通常被讲成"图片解码很慢，所以提前解好"。这句话对一半。

ImageIO 交出来的 `CGImage` 通常是懒的：它持有压缩数据和一个解码回调，真正的位图要等有人来读像素时才生成。谁来读？CoreAnimation 提交图层的时候。那是主线程。

SDWebImage 自己的头文件把这个权衡写得比多数文章清楚（SDImageCoder.h:67-75）：

> A BOOL value indicating whether to use lazy-decoding. … But however, the consumer may access bitmap buffer when running on main queue, like CoreAnimation layer render image. So this is a trade-off. You can force us to disable the lazy-decoding and always allocate bitmap buffer on RAM, but this may have higher ratio of OOM (out of memory).

它甚至明说了静态图 coder 的懒解码默认值是 YES。所以 5.x 的做法是：coder 先出一个懒的 `CGImage`，再由 `SDImageCoderHelper` 决定要不要按掉这个懒。

### 测一次

我拿一张 4032×3024 的 JPEG（磁盘 1.04 MB），在 macOS 原生下对照两条路。A 路只 `CGImageSourceCreateImageAtIndex`；B 路照抄 `CGImageCreateDecoded` 的做法再画一遍。然后各自量"第一次绘制"的耗时。为了不让两条路互相污染 malloc 的空闲块，两个模式跑在各自的进程里：

```text
[lazy]    4032x3024 bpr=16128 lazy=Y | 构建   1.9 ms
  绘制耗时(ms): 112.3  53.9  3.1  1.8  1.9  1.8  2.6  1.8
[decoded] 4032x3024 bpr=16128 lazy=N | 构建 110.5 ms
  绘制耗时(ms):   9.7   5.4  1.5  1.9  1.5  1.6  1.6  1.4
```

三轮下来，懒图第一次绘制在 99～308 ms 之间，强制解码过的稳定在 8.8～10.1 ms。差 12 到 30 倍。这个数字就是你在 Time Profiler 里看到的那根主线程尖峰。

但把两行加起来看：懒图是 1.9 + 112.3，解码过的是 110.5 + 9.7。总量差不多。**强制解码没有省掉任何工作，它只是把这 100 ms 从"CoreAnimation 提交时的主线程"挪到了 `coderQueue`。** 这是我这一节唯一想让人记住的话。`coderQueue` 被设成串行也就说得通了（SDWebImageDownloaderOperation.m:117 `maxConcurrentOperationCount = 1`）。这条队列上跑的是纯 CPU 的位图填充，开多路只会互相抢核。

再看 SDWebImage 判断"这个 CGImage 是不是懒的"那个招式。它没去读 CGImage 的私有结构体，而是解析 `CFCopyDescription` 的第一行找 `(IP)` / `(DP)`。实现在 SDImageCoderHelper.m:383-407，注释里写着 iOS 17.0 上那个字段在偏移 `0xd8`。我把三种 policy 各跑一遍：

```text
policy=Never（1）      sd_isDecoded=0  CGImage 描述首行=<CGImage 0x7368e4780> (IP) <JPEG>
policy=Automatic（0）  sd_isDecoded=1  CGImage 描述首行=<CGImage 0x7368e4c80> (DP)
policy=Always（2）     sd_isDecoded=1  CGImage 描述首行=<CGImage 0x7368e4780> (DP)
```

`(IP)` 是 ImageProvider，`(DP)` 是 DataProvider。这个字符串判据在 macOS 26 上依然成立，policy 也确实改变了输出对象的形态。

### 那几个 CGBitmapContextCreate 参数

强制解码的核心就一句（SDImageCoderHelper.m:467）：

```objc
CGBitmapInfo bitmapInfo = [SDImageCoderHelper preferredPixelFormat:hasAlpha].bitmapInfo;
CGContextRef context = CGBitmapContextCreate(NULL, newWidth, newHeight, 8, 0, [self colorSpaceGetDeviceRGB], bitmapInfo);
```

四个参数值得逐个说，因为老文章在这里普遍抄的是几年前的写法。

bytesPerRow 传 0。 不是 `width * 4`。传 0 的意思是让 CoreGraphics 自己算并自己对齐，这比手写一个乘法安全。很多 2015～2020 年的解析文章贴的是手动计算的版本，那是当时的源码。

bitmapInfo 是运行时问出来的，不是硬编码。 上面那行注释写得很直白：

```objc
// kCGImageAlphaNone is not supported in CGBitmapContextCreate.
// Check #3330 for more detail about why this bitmap is choosen.
// From v5.17.0, use runtime detection of bitmap info instead of hardcode.
```

`preferredPixelFormat:` 的做法是现画一张 1×1 的图，再把系统给它选的 `CGImageGetBitmapInfo` 抄回来。两张 dummy 图在 SDImageCoderHelper.m:77-111。写死 `kCGBitmapByteOrder32Host | kCGImageAlphaPremultipliedFirst` 是 5.17.0 之前的事了。

alignment 里那个 8 借自 FastImageCache。见 SDImageCoderHelper.m:320-322：

```objc
// https://github.com/path/FastImageCache#byte-alignment
// A properly aligned bytes-per-row value must be a multiple of 8 pixels × bytes per pixel.
size_t alignment = (bitsPerComponent / 8) * components * 8;
```

8 位分量、4 通道，算出来 32 字节。这个值只被 `CGImageIsHardwareSupported:` 用（SDImageCoderHelper.m:336），拿来判断一个已经不懒的 `CGImage` 是否还需要再解一次。判据两条：`bytesPerRow` 对不对齐、色彩空间是不是 sRGB/deviceRGB（P3 会被判成不支持）。任何一条不过，`shouldDecodeImage:` 就返回 YES，理由是渲染时会触发 `CA::copy_image` 这次额外拷贝。

所以 Automatic policy 的完整逻辑是这样（SDImageCoderHelper.m:961-1007）。动图、矢量图、HDR 图一律不解。已经解过的不重解。懒的必解。不懒但硬件不友好的，也解。

我实测的那张图 `bytesPerRow=16128`，正好是 4032×4，天然 32 字节对齐，所以它走的是"懒 → 必解"这一支。

---

## 三、downsample：这一节最实用

`SDWebImageContextImageThumbnailPixelSize` 是 5.x 才有的。它不走"先解全图再缩小"那条路。尺寸交给 ImageIO，解码阶段就只出小图（SDImageIOAnimatedCoder.m:502-517）：

```objc
decodingOptions[(__bridge NSString *)kCGImageSourceThumbnailMaxPixelSize] = @(maxPixelSize);
decodingOptions[(__bridge NSString *)kCGImageSourceCreateThumbnailFromImageAlways] = @(YES);
imageRef = CGImageSourceCreateThumbnailAtIndex(source, index, (__bridge CFDictionaryRef)[decodingOptions copy]);
```

`kCGImageSourceCreateThumbnailWithTransform` 传的是 `preserveAspectRatio`。开着的时候 EXIF 方向由 ImageIO 一并处理。源码随后就把 `exifOrientation` 重置成 Up，免得转两次（第 527-528 行）。

跑真实链路对照，同一个 URL、同一个 manager：

```text
不带 context           4032x3024   sd_memoryCost = 46.510 MB
带 400x400 缩略图       400x300    sd_memoryCost =  0.458 MB   耗时 16 ms
```

101 倍。 `sd_memoryCost` 不是估的，是 `CGImageGetBytesPerRow(imageRef) * CGImageGetHeight(imageRef)`（UIImage+MemoryCacheCost.m:18），也就是位图的真实字节数。

再看物理内存。我在独立进程里各建 6 张同源图，全部画一次逼它们落地：

```text
[decoded x6]  各画一次之后  footprint = 285.74 MB
[thumb   x6]  各画一次之后  footprint =   7.28 MB
```

283 MB 对 4.9 MB（都已扣掉起点的 2.4 MB）。一个头像列表如果每格都塞全尺寸解码图，六张就够触发 iOS 的 jetsam 了。

这里我踩了一个坑，值得单独说，因为它差点让我写出一条假结论。

我第一版把三种模式写在同一个进程里顺序跑，量出来"强制解码后的图只涨 0.16 MB"。这显然荒唐——46 MB 的位图不可能不占内存。原因是前一个模式画图时 malloc 已经向内核要过并弄脏了 46 MB，后一个模式直接复用了那块空闲内存，`phys_footprint` 当然不涨。这个数字量的是分配器的状态，不是对象的成本。

改成一个模式一个进程之后还有第二层：只建好对象不碰它，`phys_footprint` 依然几乎不涨（6 张 46 MB 的图只涨 4 MB）。位图的 backing store 在被真正读写之前不计进 footprint。必须画一次，数字才对得上理论值。

> 待真机补测：以上 footprint 全部来自 macOS arm64 原生进程。iOS 上 `phys_footprint` 是 jetsam 的判据，会话内存压力和 compressor 行为都不同。复现方法：同一段代码编成 iOS target，用 `SDWebImageContextImageThumbnailPixelSize` 开关做 A/B，在真机上读 `task_vm_info` 的 `phys_footprint`，同时开 Instruments 的 Allocations 做 Generation 对比。footprint 的口径见 [[iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint]]。

另外，`SDWebImageScaleDownLargeImages` 走的是另一条路，`decodedAndScaledDownImageWithImage:limitBytes:`（SDImageCoderHelper.m:721）。它按字节上限反推目标像素数，再分块（tile）画。上限是编译期常量（SDImageCoderHelper.m:125-131）：

```c
#if SD_MAC
static CGFloat kDestImageLimitBytes = 90.f * kBytesPerMB;
#elif SD_UIKIT
static CGFloat kDestImageLimitBytes = 60.f * kBytesPerMB;
#elif SD_WATCH
static CGFloat kDestImageLimitBytes = 30.f * kBytesPerMB;
#endif
```

分块那段的注释解释了 tile 的宽度为什么必须等于原图整宽。iOS 从磁盘解码是整条带（band）解的。就算你把上下文裁到带内的一小块，那一整条也照样解了一遍。所以按整宽切才不浪费。

这两条路的取舍很清楚。`limitBytes` 只保证不超过某个内存上限，出来的尺寸取决于原图；thumbnail 直接指定你要的像素尺寸。做列表就用后者。

---

## 四、两级缓存

### key 怎么算

内存和磁盘用同一个 key，来自 `cacheKeyForURL:context:`（SDWebImageManager.m:136-183）。默认就是 `url.absoluteString`，然后按顺序追加两层后缀：缩略图尺寸、transformer 标识。

```text
cacheKey        = http://127.0.0.1:8899/big.jpg
缩略图 cacheKey  = http://127.0.0.1:8899/big-Thumbnail(%7B400.000000,400.000000%7D,1).jpg
```

缩略图那串是 `SDThumbnailedKeyForKey` 拼的（SDImageTransformer.m:42-45）。格式是 `Thumbnail({宽,高},保持比例)`，宽高用 `%f` 打印所以带六位小数。它插在扩展名之前，`.jpg` 留在最后。那个 `%7B` 是 `{` 被 `NSURLComponents` 转义的结果，因为拼接走的是 URL 组件而不是字符串拼接。

磁盘文件名是这个 key 的 MD5 加扩展名（SDDiskCache.m:360-387）。我把库算出来的和命令行算的对了一遍：

```text
cacheKey             = http://127.0.0.1:8899/big.jpg
库算出的文件名        = a42d8bea8413da8e23c4d08fc068a9c2.jpg
$ echo -n "http://127.0.0.1:8899/big.jpg" | md5
a42d8bea8413da8e23c4d08fc068a9c2
```

对上了。扩展名那部分比老文章多了两道处理。一是 `SDSanitizeFileNameString` 剔掉 `\0` 和 `:`，注释说这是 Apple 文件系统上唯一非法的字符，`/` 和 `\` 反而合法。二是一个长度闸门：

```c
#define SD_MAX_FILE_EXTENSION_LENGTH (NAME_MAX - CC_MD5_DIGEST_LENGTH * 2 - 1)
```

`NAME_MAX` 减去 32 个十六进制字符再减一个点。扩展名超了就整个不要。构造一个 query string 很长的 URL 就能触发。

默认目录也换过。现在是 `~/Library/Caches/com.hackemist.SDImageCache/<namespace>`（SDImageCache.m:94-99），老的是 `~/Library/Caches/<ns>/com.hackemist.SDWebImageCache.<ns>`。`migrateDiskCacheDirectory` 专门留着从旧路径搬家（SDImageCache.m:192-205）。老文章里贴的 `makeDiskCachePath:` 算出来的正是那个待迁移的旧路径。

### 什么时候写

写内存是同步的，就在调用线程上（SDImageCache.m:249-255）：

```objc
if (image && toMemory && self.config.shouldCacheImagesInMemory) {
    NSUInteger cost = image.sd_memoryCost;
    [self.memoryCache setObject:image forKey:key cost:cost];
}
```

写磁盘要经过 `ioQueue`。手上只有 `UIImage` 没有原始 `NSData` 时，还得先编码一次。编码跑在 global HIGH 队列（SDImageCache.m:272-309）。编码格式的选择逻辑也变了。老版本一律 PNG，现在先看 `image.sd_imageFormat`，未知时按有没有 alpha 通道决定 PNG 还是 JPEG（第 285-287 行）。

缩略图有个特例：不存 data（SDImageCache.m:264-266）。

```objc
if (image.sd_isThumbnail && SDIsThumbnailKey(key)) {
    // Currently we have no solid way to store thumbnail image's correct data
    data = nil;
}
```

`SDIsThumbnailKey` 的实现是在 key 里搜 `-Thumbnail(` 这个子串（SDImageCache.m:22-27）。字符串判据，谈不上优雅，但这也说明缩略图那套 key 拼法是有约定意义的，不能随便改。

### NSCache 的驱逐，实测

`SDMemoryCache` 继承自 `NSCache`，自己只做三件事。把 `config` 的两个上限同步进 `totalCostLimit` / `countLimit`，并用 KVO 保持跟随（SDMemoryCache.m:59-60）。维护一张 strong-weak 的 `NSMapTable` 做二级恢复（第 66 行）。监听内存警告（第 78-81 行）。真正的驱逐策略全在 `NSCache` 里。

那就直接测 `NSCache`。macOS 原生，Foundation 就够。

先是默认值：

```text
totalCostLimit = 0    countLimit = 0    evictsObjectsWithDiscardedContent = 1
```

两个上限默认都是 0，也就是不限。塞 200 个 1 MB 进去，200 个全在。而 `SDImageCacheConfig` 的 `maxMemoryCost` 和 `maxMemoryCount` 默认也是 0（SDImageCacheConfig.m 的 `init` 里没有赋值）。**所以开箱即用的 SDWebImage，内存缓存是没有容量上限的**，能不能清全看系统内存压力和它自己那个内存警告回调。

`evictsObjectsWithDiscardedContent` 默认为 YES，这一点很多资料没提。我拿一个实现了 `NSDiscardableContent` 的对象验了一下：主动 `discardContentIfPossible` 之后，开关为 YES 时 cache 里当场取不到，为 NO 时还在。

然后是驱逐顺序。设 `totalCostLimit = 20 MB`，逐个塞 60 个 1 MB 的对象：

```text
A. 中途不做任何读取
   存活: 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59

B. 每插一个就把已有的 key 全读一遍
   存活: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 59
```

两组只差一个"读"的动作，存活集合几乎完全相反，而且各跑三次结果一模一样。**`NSCache` 的驱逐顺序会被读取动作改变，谁在什么时候 `objectForKey:` 过，直接决定了谁被扔掉。**

我第一版实验就是 B 那种写法：为了统计存活数，在插入循环里调了一个遍历所有 key 的辅助函数。测出"活下来的是最早插入的那批"，正准备去解释 `NSCache` 为什么反 LRU。幸好同一份程序里另一组 `countLimit` 的测试给出了完全相反的顺序，两条结论打架，才发现是我的探测函数改了被观察对象。

读取确实起保护作用，单独设计一组就能看出来：

```text
C. countLimit=10，先插 0-9，再读 0-4，再插 10-14
   存活: 0 1 2 3 4 10 11 12 13 14
```

被读过的 0-4 活下来了，没被读过的 5-9 被扔了。所以它有访问顺序的概念。但这个顺序怎么和插入顺序、cost 一起作用，Apple 没有文档承诺。我也不打算从三组实验里推一套机制出来。

另外两条一起记住：

```text
D. countLimit=3，插满后再插第 4 个，立刻查 → 0 号当场就没了（驱逐是同步的）
E. totalCostLimit=10MB，塞一个 cost=50MB 的对象 → 放不进去，取不到
```

还有一条最容易踩：

```text
cost 全传 0，totalCostLimit = 20 MB，塞 100 个 1 MB → 100 个全在
```

**cost 传 0 等于没有 `totalCostLimit`。** `NSCache` 不会去问对象有多大。SDWebImage 每次写内存都显式算了 `image.sd_memoryCost` 再传进去（SDImageCache.m:253-254）。这不是可有可无的讲究。

至于内存警告，我只能验到通知这一层：手动 post `UIApplicationDidReceiveMemoryWarningNotification`，缓存里的图当场取不到了，符合 `[super removeAllObjects]` 的实现（注意它故意不清 `weakCache`）。

> 待真机补测：`NSCache` 在真实内存压力下的自动驱逐，以及系统何时发出内存警告，在 Catalyst 上测不了（macOS 用虚拟内存，`SDMemoryCache` 的这段代码本身就用 `#if SD_UIKIT` 圈起来只在 iOS/tvOS 编译）。复现方法：真机上开一个持续分配的后台任务把可用内存压下去，观察 `SDImageCache.memoryCache` 的存活数量与 `didReceiveMemoryWarning:` 的调用时机。

### 磁盘什么时候清

两条独立规则，都在 `SDDiskCache.removeExpiredData`（SDDiskCache.m:151-244）。

第一条按时间。`maxDiskAge` 默认一周（SDImageCacheConfig.m:14，`60 * 60 * 24 * 7`），比较的日期字段由 `diskCacheExpireType` 决定，默认是访问时间。这一点很关键：`dataForKey:` 每次读完都会把 `NSURLContentAccessDateKey` 刷成当前时间（SDDiskCache.m:79）。所以常被读到的图永远不过期。

把 `maxDiskAge` 调成 2 秒实测：

```text
写入 10 个后         totalDiskCount = 10   totalDiskSize = 10.36 MB
等 3 秒，其中 3 个刚被读过
清理后               totalDiskCount = 3
  0.jpg 还在: 1   1.jpg 还在: 1   2.jpg 还在: 1
  3.jpg 还在: 0   4.jpg 还在: 0   …   9.jpg 还在: 0
```

第二条按大小。`maxDiskSize` 默认 0，含义是不限。超了之后按日期从旧到新删，但目标不是删到上限，是删到上限的一半：

```objc
// Target half of our maximum cache size for this cleanup pass.
const NSUInteger desiredCacheSize = maxDiskSize / 2;
```

实测把上限设成 8 MB：

```text
清理前 10.36 MB / 10 个  →  清理后 3.11 MB / 3 个
```

3.11 MB 已经低于 4 MB，循环就停了。这个"砍到一半"的设计是为了避免每写一张就触发一次清理。

清理的触发时机只有三个：进后台、将退出、手动调。`shouldRemoveExpiredDataWhenEnterBackground` 和 `shouldRemoveExpiredDataWhenTerminate` 默认都是 YES。App 一直在前台跑，磁盘缓存就一直不清。

---

## 五、并发与去重

下载队列的默认值在 `SDWebImageDownloaderConfig.m:27-29`：

```objc
_maxConcurrentDownloads = 6;
_downloadTimeout = 15.0;
_executionOrder = SDWebImageDownloaderFIFOExecutionOrder;
```

6 这个数字被 `SDWebImageDownloader.m:102` 赋给 `downloadQueue.maxConcurrentOperationCount`，并且用 KVO 跟着 config 变。我改一下 config 再读队列，确实同步了：

```text
队列名 = com.hackemist.SDWebImageDownloader.downloadQueue
maxConcurrentOperationCount = 6
改 config 之后（KVO 同步）= 2
```

去重靠 `URLOperations` 这张以 `NSURL` 为 key 的字典（SDWebImageDownloader.m:222-264）。同 URL 第二次进来，只要那个 operation 既没 finish 也没 cancel，就只往它身上挂一组回调：

```objc
@synchronized (operation) {
    downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock decodeOptions:decodeOptions];
}
```

我起了一个本地 `python3 -m http.server`，对同一个 URL 连发 20 个下载请求，然后数服务端日志：

```text
提交 20 个下载后 URLOperations 里的 operation 数 = 1
20 个 completion 全部回调完毕（cbs = 20）
服务端日志里该 URL 的 GET 请求数 = 1
```

一次 HTTP 请求，20 个回调。

有个细节容易忽略：合并的粒度是 URL，但回调携带各自的 `decodeOptions`（第 220 行算出来的）。同一个 URL 要两种不同缩略图尺寸时，共用一次下载，解码分别做。注释写得很明白：`When different thumbnail size download with same url, we need to make sure each callback called with desired size`。

LIFO 的实现有点意思（SDWebImageDownloader.m:391-398）。`NSOperationQueue` 没有 LIFO 模式。SDWebImage 的做法是每加一个新 operation，就让队列里所有还没跑的 operation 依赖它：

```objc
for (NSOperation *pendingOperation in self.downloadQueue.operations) {
    [pendingOperation addDependency:operation];
}
```

注释里说光让最后一个依赖新的解决不了问题，得全挂上。列表往下滚的场景开 LIFO 是有收益的，代价是队列里 operation 一多，这个 O(n) 的挂依赖就不便宜。

> 待真机补测：`UITableView` 快速滚动时的重复请求、取消时机、以及 cell 复用导致的错图，在 Catalyst 命令行进程里做不出可比的场景。复现方法：真机跑一个 200 行的图片列表，`SDWebImageDownloader` 的 `downloadTimeout` 调到 60 s 便于观察，在 `sd_cancelImageLoadOperationWithKey:` 和 `addHandlersForProgress:` 上各下一个断点计数，配合 Network 模板看实际发出的请求数。

---

## 六、今天已经不成立的说法

按证据强度排，前几条我都能拿 grep 或运行结果顶上。

- "图片下载由 `NSURLConnection` 完成，下载完交给 `SDWebImageDecoder` 解码。" 两个符号在 5.21.7 里都搜不到。现在是 `NSURLSession`（`SDWebImageDownloader.m:139`）加 `SDImageCoder` 协议族。
- "缓存命中通过 `SDImageCacheDelegate` 回调 `imageCache:didFindImage:forKey:userInfo:`。" 整套委托协议在 5.x 已经不存在，全部换成 block。搜不到 `SDImageCacheDelegate`，也搜不到 `notifyDelegate`。
- "`SDImageCache` 上有 `shouldDecompressImages` / `maxCacheAge` / `maxCacheSize` / `maxMemoryCost` 四个属性。" 全部搬进了 `SDImageCacheConfig`，而且改了名：`maxCacheAge` → `maxDiskAge`，`maxCacheSize` → `maxDiskSize`。`shouldDecompressImages` 直接删了，对应能力变成 `SDImageCoderHelper` 的 `SDImageForceDecodePolicy`。
- "`SDWebImageManager` 上调 `downloadImageWithURL:options:progress:completed:`。" 这个签名不在了，入口是 `loadImageWithURL:options:context:progress:completed:`。多出来的 `context` 参数是 5.x 一半新能力的载体。
- `SDWebImageOptions` 那张表从第三位起全部错位。 `SDWebImageCacheMemoryOnly`（原 `1 << 2`）被删了，能力挪到 `SDWebImageContextStoreCacheType`；`SDWebImageProgressiveDownload` 改名 `SDWebImageProgressiveLoad` 并占了 `1 << 2`。于是后面每一条的位都往前挪了一格：`RefreshCached` 从 `1<<4` 变 `1<<3`，`HighPriority` 从 `1<<8` 变 `1<<7`，`DelayPlaceholder` 从 `1<<9` 变 `1<<8`。老文章列 11 条，现在是 26 条。
- "磁盘缓存目录是 `~/Library/Caches/<ns>/com.hackemist.SDWebImageCache.<ns>`。" 那是被 `migrateDiskCacheDirectory` 迁移走的旧路径。现在是 `~/Library/Caches/com.hackemist.SDImageCache/<ns>`。
- "强制解码用手算的 `bytesPerRow` 和硬编码的 `bitmapInfo`。" 5.17.0 起 `bytesPerRow` 传 0 交给 CoreGraphics，`bitmapInfo` 靠画一张 1×1 探测图运行时问出来。
- "内存缓存有 LRU。" `NSCache` 的驱逐顺序没有文档承诺，我实测它会被读取动作改变，别按 LRU 去推理命中率。
- "`SDMemoryCache` 的 `weakCache` 默认开着。" `shouldUseWeakMemoryCache` 默认是 NO（SDImageCacheConfig.m 的 `init`）。而且 `UIView+WebCache.m` 里那段为它做的额外查询带着一句注释：`in the future the weak cache feature may be re-design or removed`。

---

## 七、我的判断：哪些还值得学，哪些已经被系统接管

先说被接管的那部分。

`decodedImageWithImage:` 这套用 `CGBitmapContextCreate` 重画一遍的手法，在 iOS 15 之后已经不是首选路径了。SDWebImage 自己就先试系统 API（SDImageCoderHelper.m:29-41）：

```objc
if (@available(iOS 15, tvOS 15, *)) {
    UIImage *decodedImage = [image imageByPreparingForDisplay];
    ...
}
```

downsample 那一路同理，优先 `imageByPreparingThumbnailOfSize:`（第 43-58 行）。而且 Automatic 方案只对 JPEG 和硬件支持的 HEIF 走这条（第 660-667 行）。原因是注释里那个 issue：`CMPhoto iOS 15 only supports JPEG/HEIF format, or it will print an error log`。

所以如果你今天要自己写图片解码，我的做法是：iOS 15+ 直接用 `imageByPreparingForDisplay` 和 `imageByPreparingThumbnailOfSize:`，别去手写 bitmap context。要在解码阶段就控制尺寸，用 `CGImageSourceCreateThumbnailAtIndex` 配 `kCGImageSourceThumbnailMaxPixelSize`，三个 key 的组合抄 SDWebImage 那三行就行。手写 `CGBitmapContextCreate` 唯一还必要的场合，是你需要精确控制 `bitmapInfo` 或色彩空间——比如把 P3 的图强制拍平成 sRGB。

还值得学的是另外三样，都不是解码本身。

第一是那套 key 的设计。缩略图尺寸和 transformer 标识拼进 cache key。同一张原图的不同派生物于是能共存于一个缓存空间，还能各自命中。这个思路和图片库没关系，任何带派生结果的缓存都能用。

第二是"合并粒度和处理粒度分开"。20 个请求合并成 1 次下载，但每个回调带自己的 `decodeOptions`。很多人写去重会把整个下游也一起合并掉，然后发现两个调用方要的东西不一样。

第三是 `NSCache` 那几个坑本身。cost 不传就等于没有上限、默认无上限、驱逐顺序不承诺。这三条和 SDWebImage 无关，是所有用 `NSCache` 的代码都要面对的。

至于分块解码（tile）那一大段，我的判断是今天基本不用看了。它解决的是 iPad 1 和 iPhone 3GS 时代一次性分配几十 MB 会失败的问题。源码注释里那三行 "Suggested value for iPad1 and iPhone 3GS: 60" 就是化石。真要处理超大图，`CGImageSourceCreateThumbnailAtIndex` 从一开始就不会把全图解出来，比分块画完再缩干净得多。

最后一条是方法论。这一篇里所有"网上说的不对"，没有一条靠的是读更多文章。要么是 `grep -rl` 一次告诉我那个符号不存在，要么是把库编进二进制跑一遍打印出来。第五节那两组顺序相反的 `NSCache` 结果最说明问题。同一个下午我差点写下一条完全错误的机制解释，救回来的不是知识，是"两条结论打架时先怀疑仪器"这个习惯。

---

## 总结

链路的骨架是四个队列。主线程发起并接收结果。磁盘 I/O 在 `com.hackemist.SDImageCache.ioQueue`，解码在串行的 `coderQueue`，transform 在 global HIGH。内存命中这一支是同步的，不切队列，这是它撑得住列表滚动的原因。

强制解码不减少总耗时。实测懒图第一次绘制 99～308 ms，预解码过的 8.8～10.1 ms，但预解码本身要花 110 ms。省下的是主线程，不是 CPU。

真正省内存的是 downsample。同一张 4032×3024 的图，全解码 46.51 MB，`SDWebImageContextImageThumbnailPixelSize` 到 400 之后 0.458 MB。六张的物理内存差是 283 MB 对 4.9 MB。

`NSCache` 的默认上限是 0（不限），cost 传 0 会让 `totalCostLimit` 完全失效，驱逐顺序会被读取动作改变。SDWebImage 显式传 `sd_memoryCost` 并自己监听内存警告，就是在补这三个洞。

那批 2015 年前后的中文解析文章，今天已经不能当流程图用了——十个类名方法名 grep 下来一个都不剩。这不是版本号从 4 跳到 5 的小改，是委托改 block、类改协议、职责重新切过一遍。

下一篇 [[iOS 架构模式：MVC 到 MVVM，以及它们各自解决不了的问题]]。

## 参考资料

### 一手

- [SDWebImage 5.21.7 源码](https://github.com/SDWebImage/SDWebImage)：本文所有行号来自 `git clone --depth 50` 拿到的 `c3ad5e1`，`SDWebImage.podspec` 里 `s.version = '5.21.7'`
- [5.6 Code Architecture Analysis（官方 Wiki）](https://github.com/SDWebImage/SDWebImage/wiki/5.6-Code-Architecture-Analysis)：协议化那一步的官方说明，原话是 "the protocolization for the core classes is the biggest change in the 5.x SD version"
- [imageByPreparingForDisplay](https://developer.apple.com/documentation/uikit/uiimage/3750834-imagebypreparingfordisplay) / [imageByPreparingThumbnailOfSize:](https://developer.apple.com/documentation/uikit/uiimage/3750835-imagebypreparingthumbnailofsize)：iOS 15 起 SDWebImage 优先走的两个入口
- [NSCache](https://developer.apple.com/documentation/foundation/nscache)：`evictsObjectsWithDiscardedContent` 和 cost 的语义；驱逐顺序文档没有承诺
- [FastImageCache — Byte Alignment](https://github.com/path/FastImageCache#byte-alignment)：`SDImageCoderHelper.m:320` 注释直接引的出处

### 二手（读了，但都基于旧版本）

- [southpeak — SDWebImage 实现分析](http://southpeak.github.io/2015/02/07/sourcecode-sdwebimage/)：2015 年，3.x，委托回调链的中文源头
- [SDWebImage 实现原理与源码简析](https://www.cnblogs.com/zhangzhang-y/p/13584570.html)：2020 年，5.0 前后。`makeDiskCachePath:`、`SDCacheCostForImage`、`SDWebImageCodersManager` 这几个符号今天都没了
- [The Architecture of SDWebImage v5.6](https://looseyi.github.io/post/sourcecode-ios/source-code-sdweb-en1/)：5.6，是二手材料里最接近现状的

### 本地

- [[iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint]]
- [[iOS GCD：队列不是线程，以及死锁的准确边界]]
- 我自己 2025 年那篇《SDWebImage 解析》（`90 素材/csdn-import`）：使用方式和整体印象仍然成立，第二节那段流程叙述基于 3.x，以本文为准

---

实验环境：macOS 26.5.2，arm64 Apple Silicon。Xcode 26.5 SDK，Apple clang 21.0.0。

分三类。第一类是纯 Foundation / CoreGraphics / ImageIO 的实验。`NSCache` 驱逐、解码耗时与内存、downsample 内存对照都在这一类。它们直接编成 macOS 原生二进制跑，`clang -fobjc-arc -framework Foundation -framework CoreGraphics -framework ImageIO`。第二类要用 SDWebImage 本体。把 `SDWebImage/Core` 和 `SDWebImage/Private` 下全部 72 个 `.m` 编成 Mac Catalyst target，再链进测试程序：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -c -fobjc-arc -O1 -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -ISDWebImage/Core -ISDWebImage/Private SDWebImage/Core/*.m SDWebImage/Private/*.m
```

跑出来是原生 macOS 进程，加载真正的 `UIKitCore`，全程没有 boot 任何模拟器。第三类是网络。本地 `python3 -m http.server 8899` 喂一张自己生成的 4032×3024 JPEG，去重的证据是服务端访问日志。

`#if SD_UIKIT` 圈起来的代码在 Catalyst 上确实参与编译，内存警告那一节能跑通就是证明。但 Catalyst 桥接层的行为不等于 iOS。凡是本文标了 `待真机补测` 的，都是 Catalyst 结果不可外推的地方。有四处：真机内存压力下的 `NSCache` 驱逐，`phys_footprint` 与 jetsam 的关系，`UITableView` 滚动复用，弱网下载。

> 待真机补测：`SDImageCoderHelper.decodedImageWithImage:` 在 iOS 15+ 真机上走的是 `imageByPreparingForDisplay`（CMPhoto）分支，本文第二节量的是 CoreGraphics 分支。复现方法：真机上分别设 `SDImageCoderHelper.defaultDecodeSolution` 为 `UIKit` 和 `CoreGraphics`，对同一张 JPEG 各量一次首帧绘制耗时，看两条路径的差距有多大。
