---
title: 【iOS】网络分层：URLSession 之上该有几层
published: 2026-07-27
description: 用同一个 URL 连发三次 dataTask，本地服务端只收到一次请求。URLSession 默认就带一层 HTTP 缓存，而且它的缓存 key 不包含 Authorization 头。
tags:
  - iOS
  - Networking
  - URLSession
  - HTTP
  - AFNetworking
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 37
draft: true
---
# 网络分层：URLSession 之上该有几层

先看一段没有任何技巧的代码。用 `NSURLSession.sharedSession` 对同一个 URL 连发三次 `dataTask`，每次都等上一次回来再发下一次：

```text
第 1 次：status=200  14 字节  236.4 ms
第 2 次：status=200  14 字节  1.7 ms
第 3 次：status=200  14 字节  0.3 ms
```

同一时刻本地 HTTP 服务端的访问日志：

```text
1785110117.919775 REQ GET /cacheable
```

**三次 dataTask，服务端只收到一次请求。** 服务端唯一做的事是在响应里带了一行 `Cache-Control: max-age=60`。

这篇的所有结论都来自这台机器上跑出来的输出。方法很土。写一个几十行的 Python HTTP server，精确控制响应头、延迟、状态码、重定向，并把每一次真实到达的请求写进日志。客户端用 `clang -framework Foundation` 编成原生 macOS 二进制直接跑。服务端日志是唯一可信的裁判，它不受客户端任何抽象层的影响。

分层这个题目最容易写空。写成"请求层-业务层-数据层"三个方框加箭头，谁都没法反驳，也谁都用不上。我的办法是反过来：先把 `URLSession` 这一层实际提供了什么、没提供什么、提供得对不对全测一遍，剩下没被覆盖的部分才是你需要自己写的层。

架构模式本身（MVC 怎么切、MVVM 的绑定往哪走）不在这篇，在 [[iOS 架构模式：MVC 到 MVVM，以及它们各自解决不了的问题]]。图片的下载与两级缓存也不在这篇，在 [[iOS SDWebImage：下载、解码与两级缓存的完整链路]]。

---

## 一、你已经有一层缓存了

`NSURLRequest` 的默认 `cachePolicy` 是 `NSURLRequestUseProtocolCachePolicy`，值为 0。`defaultSessionConfiguration` 的 `URLCache` 默认就是 `NSURLCache.sharedURLCache`，指针相等。这两件事合起来的效果就是开头那段输出。

打印一下 `sharedURLCache` 在这台机器上的默认容量：

```text
memoryCapacity = 512000 bytes (0.5 MB)
diskCapacity   = 20000000 bytes (19.1 MB)
```

磁盘部分不是一堆散文件。它是 `~/Library/Caches/<进程名或 bundle id>/Cache.db`，一个 SQLite 库：

```text
$ sqlite3 ~/Library/Caches/exp1/Cache.db ".tables"
cfurl_cache_blob_data      cfurl_cache_response     cfurl_cache_receiver_data

$ sqlite3 ~/Library/Caches/exp1/Cache.db "select entry_ID, request_key from cfurl_cache_response;"
1|http://127.0.0.1:8931/cacheable
2|http://127.0.0.1:8931/nocache
3|http://127.0.0.1:8931/nohdr
4|http://127.0.0.1:8931/etag
```

`request_key` 那一列后面还要用到。先看不同的 `Cache-Control` 各自是什么行为。每个 URL 发两次，只看服务端收到几次：

| 服务端响应头 | 第 2 次是否发出真实请求 | URLCache 里有条目 | 说明 |
|---|---|---|---|
| `max-age=60` | 否 | 有 | 直接读缓存，回调里的 status 仍是 200 |
| `no-store` | 是 | 无 | 连条目都不建 |
| `no-cache` | 是 | 有 | 存了，但每次都回源 |
| 什么都不给 | 是 | 有 | 存了，但没有新鲜度依据，回源 |
| `ETag` + `max-age=0` | 是，且带上 `If-None-Match` | 有 | 服务端答 304 |

最后一行值得单独看。服务端第二次收到的是：

```text
REQ GET /etag hdrs={"if-none-match": "\"v1\"", "connection": "keep-alive"}
```

服务端回的是 `304 Not Modified`，`Content-Length: 0`。但客户端 completion 里拿到的是：

```text
2nd /etag  status=200 bytes=12  cacheEntry=YES
```

status 是 200，body 是完整的 12 字节。条件请求和 304 合并这套流程被 `URLSession` 整个吞掉了，你的 `completionHandler` 永远看不到 304。这也意味着，如果你在业务层写了一句 `if (statusCode == 304) { ... }`，那段代码永远不会执行。

### cachePolicy 那几个枚举实测

在 `/cacheable` 已经有缓存的前提下逐个打一遍：

| cachePolicy | 服务端收到请求 | 客户端结果 |
|---|---|---|
| `UseProtocolCachePolicy` | 否 | 200 |
| `ReloadIgnoringLocalCacheData` | 是 | 200 |
| `ReturnCacheDataElseLoad` | 否 | 200 |
| `ReturnCacheDataDontLoad` | 否 | 200 |

`ReturnCacheDataDontLoad` 打在没有缓存的 URL 上时：

```text
DontLoad 打在没缓存的 /nostore  status=0  bytes=0  err=resource unavailable
```

`resource unavailable` 是 `NSURLErrorResourceUnavailable`，值 -1008。做离线模式时这个组合很有用，它保证一个字节都不出去。

---

## 二、缓存的 key 是什么

这一节是我认为整篇最该先读的部分，因为它直接决定"缓存到底能不能放在 `URLCache` 这一层"。

**同一个 URL，不同的 `Authorization` 头，共用同一个缓存条目。**

```text
用户 A 用 Bearer USER-A 请求 /cacheable（真实发出）
用户 B 用 Bearer USER-B 请求 /cacheable，cachePolicy = ReturnCacheDataDontLoad
  -> status=200  body=cacheable-body
  -> 服务端共收到 1 次
```

用户 B 拿到了用户 A 那次响应的 body，请求根本没出去。在一个支持账号切换的 App 里，这是一条能直接上线上事故报告的路径。防它只有两个办法：服务端对所有带认证的响应打 `Cache-Control: no-store`，或者客户端不用 `URLCache` 存业务数据。我自己走第二条，理由在第十节说。

方法（method）确实在 key 里面：

```text
GET 打底           status=200  缓存条目=有
POST + DontLoad    status=200  缓存条目=无
GET + DontLoad     status=200  缓存条目=有
服务端共收到 2 次
```

`cachedResponseForRequest:` 对 POST 请求返回 nil，说明 GET 的条目不会被 POST 命中。但同一行还暴露了另一件事：POST 请求上的 `ReturnCacheDataDontLoad` 没有报错，而是照样走了网络。这个策略在非 GET 请求上实测是被忽略的。

再两条容易踩的：

```
404 + Cache-Control: max-age=60   ->  两次请求，服务端只收到 1 次
301 + Cache-Control: max-age=60   ->  /301cacheable 收到 1 次，/target 收到 2 次
```

404 会被缓存。很多人默认只有 2xx 进缓存，`URLCache` 不是这么做的。而缓存住的 301 只省掉了重定向那一跳，目标 URL 该请求还是要请求。

最后是所有权。两个各自 `sessionWithConfiguration:` 出来的 `NSURLSession` 实例：

```text
s1 请求过之后，s2 用 DontLoad 拿到 status=200；服务端共收到 1 次
```

它们共享缓存。两份 configuration 的 `URLCache` 属性都指向同一个 `sharedURLCache`。"给每个业务模块建一个自己的 session 来做隔离"，在缓存这个维度上无效。除非你同时给每个 configuration 换一个独立的 `NSURLCache` 实例。

---

## 三、三种 configuration 的实际差别

```text
default      URLCache=有  mem=512000  disk=20000000  identifier=-
ephemeral    URLCache=有  mem=512000  disk=0         identifier=-
background   URLCache=nil mem=0       disk=0         identifier=com.tw.exp2.bg
```

`ephemeral` 的 `URLCache` 不是 nil。它的 `diskCapacity` 是 0，`memoryCapacity` 和 default 一样是 512000。所以"ephemeral 不缓存"这个说法是错的，准确的说法是它不往磁盘写。实测：

```text
[开跑前] ~/Library/Caches/exp2 = (目录不存在)
  ephemeral 1st  status=200  63ms
  ephemeral 2nd  status=200  7ms      <- 命中了内存缓存，服务端没收到
[ephemeral 之后] ~/Library/Caches/exp2 = (目录不存在)
  default 1st    status=200  63ms
  default 2nd    status=200  2ms
[default 之后]   ~/Library/Caches/exp2 = Cache.db-shm, Cache.db-wal, Cache.db
```

目录是在 default session 发出第一个请求之后才出现的。Apple 文档对 ephemeral 的措辞和这个结果一致：

> session-related data is stored in RAM. The only time an ephemeral session writes data to disk is when you tell it to write the contents of a URL to a file.

### background 在命令行程序里跑得起来

我原本以为跑不起来，`backgroundSessionConfigurationWithIdentifier:` 在没有 app bundle 的进程里应该会出问题。结果是能跑，而且跑得比预期更彻底。

程序起一个 background session，下载一个会持续吐 10 秒数据的 URL，3 秒后进程主动退出：

```text
进程将在 3 秒后 exit
exit
--- 进程已退出，等 12 秒看服务端 ---
1785110087.249060 REQ GET /dribble?t=10
1785110097.349223 DRIBBLE-COMPLETE /dribble?t=10 ua=bgtest (unknown version) CFNetwork/3860.600.21 Darwin/25.5.0
```

请求发出在 087，传输完成在 097，中间隔了 10 秒。进程在 090 前后就没了。传输是 `nsurlsessiond` 这个系统守护进程接着做完的，本进程只是发起方。`pgrep nsurlsessiond` 也能看到它一直在跑。

这条对分层的意义很直接：background session 的任务不属于你的进程，因此也不属于你任何一个"管理器"对象。App 重新启动之后要靠 identifier 把 session 重新连回去，再从 delegate 里领结果。任何把 completionHandler 存在内存里的封装，在 background session 上都是失效的。

> 待真机补测：background session 的真正价值（App 被挂起/杀死后继续传输、`sessionSendsLaunchEvents` 唤醒 App、`discretionary` 让系统挑时机）依赖 iOS 的进程生命周期，macOS 命令行程序测不出来。复现方法：真机上做一个 background download，按 Home 键退到后台并等待 App 被回收，看 `application:handleEventsForBackgroundURLSession:completionHandler:` 是否被调起。

---

## 四、超时有两个，很多人只知道一个

`defaultSessionConfiguration` 的默认值：

```text
timeoutIntervalForRequest  = 60.0
timeoutIntervalForResource = 604800.0     // 7 天
```

Apple 对第一个的定义写得很清楚：

> The request timeout interval controls how long (in seconds) a task should wait for additional data to arrive before giving up. The timer associated with this value is reset whenever new data arrives.

**`timeoutIntervalForRequest` 是"多久没有新数据"的空闲计时器，不是这次请求的总时长上限。** 这个区别可以直接测出来。服务端有两个端点：`/slowheader?d=N` 收到请求后静默 N 秒才发响应头；`/dribble?t=N` 立刻开始每 0.5 秒吐 16 字节、持续 N 秒。

```text
A 服务端静默8s / reqTO=3 resTO=604800    耗时= 3.3s  bytes=0    error -1001
B 持续吐数据12s / reqTO=3 resTO=604800   耗时=12.6s  bytes=352  成功 status=200
C 持续吐数据12s / reqTO=3 resTO=5        耗时= 5.7s  bytes=0    error -1001
D 服务端静默8s / reqTO=60 resTO=4        耗时= 5.0s  bytes=0    error -1001
E 连一个没人监听的端口                   耗时= 0.1s  error -1004
```

B 是关键那行。`timeoutIntervalForRequest` 设成 3 秒，这次传输跑了 12.6 秒，成功了。因为数据一直在来，空闲计时器一直在被重置。你以为设了 3 秒超时的接口，实际上可以卡住任意长的时间，只要对端隔一会儿吐一个字节。

要限制总时长得用 `timeoutIntervalForResource`，它的默认值是 7 天。C 和 D 两行都是它触发的。

### 两种超时的 error 长得一样

C 和 D 报的都是 `NSURLErrorDomain -1001 The request timed out.`。code 完全相同，`localizedDescription` 也相同。各跑三次，唯一稳定的差别在 userInfo 的 key 集合上：

```text
request 超时   userInfo keys = ... NSUnderlyingError ... _kCFStreamErrorDomainKey
resource 超时  userInfo keys = ...（没有 NSUnderlyingError）... _kCFStreamErrorDomainKey
```

request 超时带 `NSUnderlyingError`，resource 超时不带。这是私有实现细节，我不会拿它写生产代码里的分支，但排查线上超时的时候可以当线索。

还有一个精度问题。`timeoutIntervalForResource` 设 2/4/8/16 秒，实测触发时刻分别是 2.50 / 4.93 / 8.82 / 16.92 秒。稳定地晚 0.5 到 0.9 秒，跟设定值大小没有比例关系。它不是一个精确定时器。相比之下 `timeoutIntervalForRequest` 设 3 秒时三次跑出来是 3.1 / 3.0 / 3.0，准得多。

---

## 五、重定向，以及我在这里翻的一次车

服务端准备了 `/r301` `/r302` `/r307` `/r308`，都返回 `Location: /target`。`/target` 会把自己收到的 method、body 长度、`Content-Length` 原样回写进 body。客户端不设任何 delegate。

| 状态码 | 原请求 GET | 原请求 POST |
|---|---|---|
| 301 | GET /target | GET /target，body 丢弃 |
| 302 | GET /target | GET /target，body 丢弃 |
| 307 | GET /target | POST /target，body 保留 |
| 308 | GET /target | POST /target，body 保留 |

`URLSession` 默认自动跟随。这个行为和 RFC 9110 一致：301/302 允许把 POST 改写成 GET，属历史遗留；307/308 明确要求方法和 body 都不许变。

现在说我翻的车。第一次跑这个实验，四行输出全都是 `len=7`，也就是 301/302 转成 GET 之后 body 居然还在。我当时差点写下"URLSession 降级了方法但保留了 body"这条"发现"。

问题出在服务端。我的 Python handler 在 `do_POST` 里把 body 存进 `self.body`，`do_GET` 里不动它。而 HTTP/1.1 的 keep-alive 让同一个连接上的多个请求复用同一个 handler 实例。于是重定向后那次 GET 读到的是上一次 POST 留下的 `self.body`。改成每次请求开头显式清空、并且同时打印 `Content-Length` 请求头之后：

```text
POST /r301  ->  GET /target  bodylen=0 clen=None
POST /r307  ->  POST /target bodylen=7 clen=7
```

这就是规范里那条"先怀疑仪器，再怀疑结论"的一次现场版。我的观察手段（服务端 handler 的实例状态）被 keep-alive 污染了，而被污染的方向恰好指向一个听起来很有料的假结论。

### 拦下来

实现 `URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:` 就能接管：

```text
[delegate] 302 /r302 -> newRequest: GET  http://127.0.0.1:8931/target
[delegate] 307 /r307 -> newRequest: POST http://127.0.0.1:8931/target
```

传给你的 `newRequest` 已经是系统按 RFC 算好方法的那一个。`completionHandler(nil)` 表示不跟随：

```text
GET /r302 -> 最终 status=302  URL=/r302  body=[]  err=-
```

拿到的是 302 响应本身，没有 error。要读 `Location` 就从 `response.allHeaderFields` 里取。

### 重定向之后 Authorization 就没了

这条我本来打算照流传的说法写"跨主机重定向会丢 `Authorization`，同域名下保留"。测完发现不对。

服务端加一个 `/echohdr`，把收到的 `Host` / `Authorization` / 自定义头原样回写。客户端带 `Authorization: Bearer SECRET` 和 `X-My-Header: v1` 发出去：

```text
不重定向，直接打            -> host=127.0.0.1:8931 auth=Bearer SECRET custom=v1
302 -> 同 host（绝对 URL）  -> host=127.0.0.1:8931 auth=None         custom=v1
302 -> 同 host（相对 URL）  -> host=127.0.0.1:8931 auth=None         custom=v1
307 -> 同 host（相对 URL）  -> host=127.0.0.1:8931 auth=None         custom=v1
302 -> 换 host（localhost） -> host=localhost:8931 auth=None         custom=v1
```

同主机、相对 Location、307 全都丢。换主机当然也丢。而自定义头 `X-My-Header` 五种情况全部保留。跑两遍结果一致。

所以"同域名下 `Authorization` 会保留"这个说法在这台机器上不成立。区分标准不是主机是否相同，而是这个头是不是 `Authorization`。`URLSession` 对它做了特殊处理，任何一次重定向都会剥掉。

这就是 `willPerformHTTPRedirection` 最实在的一个用途：把 `newRequest` 拷成 mutable，补回 `Authorization` 再放行。这个动作只能在这个回调里做，请求层往上都没有机会。要不要补回去是个安全判断，得先确认 `newRequest.URL.host` 还是你自己的域名。

---

## 六、并发：这个数字是每 session 的

`HTTPMaximumConnectionsPerHost` 三种 configuration 的默认值都是 6。发 20 个各耗时 1 秒的请求，看服务端观察到的并发峰值：

```text
limit=默认  20 个请求 -> 服务端并发峰值 = 6   总耗时 4.0s
limit=2     20 个请求 -> 服务端并发峰值 = 2   总耗时 10.1s
limit=12    20 个请求 -> 服务端并发峰值 = 12  总耗时 2.0s
limit=默认  20 个请求 -> 服务端并发峰值 = 6   总耗时 4.0s
```

峰值和设定值严格相等，耗时也对得上（20÷6 向上取整是 4 轮，20÷2 是 10 轮）。最后一行是复跑的默认值，确认不是噪声。

Apple 文档明说了这个限制的作用域：

> This limit is per session, so if you use multiple sessions, your app as a whole may exceed this limit.

所以一个 App 里图片库一个 session、API 一个 session、埋点一个 session，对同一个域名的实际并发上限就是 18，不是 6。在"要不要给每个模块独立 session"这个问题上，这条比缓存隔离更实在。

---

## 七、cancel 之后

```text
2 秒后调 cancel，此时 countOfBytesReceived=64 state=0
completion 被调用了：2.0s  data=nil(0字节)  response=非nil  error=-999 cancelled
task.state=3  task.error.code=-999  countOfBytesReceived=64
```

三件事：completion 一定会被调用；`error.code` 是 `NSURLErrorCancelled`，值 -999；`data` 参数是 nil，尽管 `countOfBytesReceived` 明确说已经收到 64 字节。

completionHandler 版把已收到的数据丢了。delegate 版没有：

```text
[delegate] didCompleteWithError: code=-999 (cancelled)  已经交付给我 64 字节 / 4 次回调
```

`didReceiveData:` 被调了 4 次，64 字节早就交到我手上了。要做断点续传或者"取消但保留已下载部分"，只能走 delegate 路线（下载任务还有 `cancelByProducingResumeData:`）。

几个边界：

```text
二次 cancel 没崩，state=3
对已完成的 task 调 resume，state=3（没有重新发起）
resume 之前就 cancel：completion 依然被调用，error=-999
invalidateAndCancel：in-flight 任务收到 error=-999
```

刚 `dataTaskWithURL:` 出来的 task，`state` 是 1（`Suspended`）。此时 cancel 掉，completion 照样会以 -999 调用一次。这条对封装很重要。只要 task 被创建出来，它的回调就一定会走一次，没有"悄无声息地消失"这种情况。所以封装层不需要为"回调没来"准备兜底逻辑，需要准备的是"回调来了但携带的是 -999"。

---

## 八、重试该放在哪一层

先把材料铺开。同一套探测跑一遍各种失败：

| 情况 | status | error |
|---|---|---|
| 200 正常 | 200 | nil |
| 404 | 404 | nil |
| 500 | 500 | nil |
| 503 | 503 | nil |
| 请求超时 | 0 | -1001 |
| 端口没人监听 | 0 | -1004 |
| 自签证书 HTTPS | 0 | -1202 `serverCertificateUntrusted` |
| 对明文端口发 HTTPS | 0 | -1200 `secureConnectionFailed` |
| 手动 cancel | 200 | -999 |

第一件要记住的事：HTTP 层面的失败在 `URLSession` 眼里不是失败。404、500、503 的 `error` 全是 nil。`URLSession` 的职责边界画在"这次 HTTP 事务有没有正常完成"，服务端说了什么不归它管。所以任何 `if (error) { 失败 } else { 成功 }` 的写法，在 500 上会走成功分支。这是需要你自己补的第一件事，也是 AFNetworking 那类库最先提供的价值（见第九节）。

最后一行也有意思：手动 cancel 的那次，`response` 已经拿到了，status 是 200，但 `error` 是 -999。判断成功失败时先看 error 再看 status，顺序不能反。

### 写一个最小的重试封装，然后它出了 bug

规则很朴素。error 在一个白名单里就重试，status 在另一个白名单里就重试。非幂等方法只在服务端给了 `Retry-After` 时才重试。指数退避加抖动，最多 4 次。

```objc
static BOOL shouldRetry(NSURLRequest *req, NSHTTPURLResponse *resp, NSError *err) {
    if (err) {
        if (err.code == NSURLErrorCancelled) return NO;
        return [retriableErrorCodes() containsObject:@(err.code)];
    }
    if (![retriableStatus() containsObject:@(resp.statusCode)]) return NO;
    BOOL idempotent = [@[@"GET", @"HEAD", @"PUT", @"DELETE", @"OPTIONS"] containsObject:req.HTTPMethod];
    return idempotent || resp.allHeaderFields[@"Retry-After"] != nil;
}
```

跑起来，服务端日志作为裁判：

```text
A. 503 两次然后成功（GET）      3 次尝试，服务端实际收到 3 次，最终 200
B. 404（GET）                   1 次尝试，服务端实际收到 1 次
C. 请求超时（GET）              4 次尝试，服务端实际收到 4 次，用时 9.6s
D. 503 + POST + Retry-After     3 次尝试，服务端实际收到 3 次，最终 200
F. 500 一直不好（GET）          4 次尝试后放弃，服务端实际收到 4 次
```

然后是 E：

```text
E. 503 期间用户 cancel
    第 1 次：status=503
    -> 判定可重试，等 0.30s
    第 2 次：status=503
    最终：status=503  用时 0.3s
    服务端实际收到 /flaky 的请求：2 次
```

用户在第一次失败后 0.15 秒就取消了，服务端却收到了两次请求。原因是取消和重试活在两条不同的时间线上：我的 `cancelAll` 只做了两件事，把 `stopped` 置位、对"当前 task"调 cancel。可当时那个 task 已经完成了（它返回了 503），而下一次请求还躺在 `dispatch_after` 的队列里等着发。cancel 落在了两个 task 之间的空档上。

修法是在退避定时器的回调开头再检查一次：

```objc
dispatch_after(..., ^{
    if (ss.stopped) { done(nil, nil, cancelledError); return; }
    [ss send:req done:done];
});
```

修完连跑三次，每次都是"服务端实际收到 1 次"。

这个 bug 不是笔误，它是分层没画对的必然结果。重试引入了一个比单个 `NSURLSessionTask` 更长的生命周期，而 `cancel` 是挂在 task 上的。**只要重试和取消不在同一层，中间那段退避窗口就一定是个漏洞。** 我的结论：重试必须由一个持有"整轮请求"句柄的对象来做。这个句柄要覆盖所有尝试和所有退避间隔。业务层拿到的取消入口必须是它，不能是某个 task。

### 和 Alamofire 的默认策略对一遍账

Alamofire 的 `RetryPolicy.swift` 把每一个 `URLError.Code` 都列了出来，启用的写成代码，禁用的注释掉并写明理由。数一下：18 个启用，30 个明确禁用。

`.cancelled` 的那条注释是：

```swift
// [Client] An asynchronous load has been canceled.
//   - [Disabled] Request was cancelled by the client.
// .cancelled,
```

和我的判断一致。但有两条我和它不一样。

一是它的可重试状态码只有 408、500、502、503、504，没有 429。429 是 "Too Many Requests"，几乎总是伴随 `Retry-After`，是最应该按服务端指示重试的那个码。我把 429 放进了白名单。

二是它的退避是 `pow(base, retryCount) * scale`，base 2、scale 0.5，也就是 0.5s / 1s / 2s。全程没有抖动，我 grep 了整个文件，`random` 和 `jitter` 一个都没有。服务端抖一下，一批客户端同时失败，它们会在完全相同的时刻一起重试回来。我在自己的实现里加了最多 0.1 秒的随机量，这个量对单机实验没有意义，对线上有。

还有一条我改了主意的。动笔前我列的提纲里写着"SSL 错误不该重试"。Alamofire 的白名单里有 `.secureConnectionFailed`（-1200）、`.serverCertificateHasBadDate`、`.serverCertificateNotYetValid`，禁用的是 `.serverCertificateUntrusted`（-1202）、`.serverCertificateHasUnknownRoot`、`.clientCertificateRejected`。这个切法是对的：证书日期不对可能是设备时钟没同步，用户改完时间再试就好了；证书根不受信任重试一万次也是一万次失败。我上面那张表里两个 SSL 错误正好落在这条界线的两边。所以"SSL 错误一律不重试"这个说法我不同意。该区分的是"这次失败的原因有没有可能在几秒内改变"。

### 一条白名单靠不住的证据

我在这台机器上没能测出 `-1003 NSURLErrorCannotFindHost`。请求一个不存在的域名，拿到的是：

```text
跟随系统代理（默认）                    status=502  error=nil
connectionProxyDictionary=@{} 绕开代理   status=0    error=-1005
```

这台机器开着一个系统级 HTTP 代理（`scutil --proxy` 里 `HTTPEnable=1`，端口 7897），并且它用的是 fake-IP DNS。`URLSession` 的 `connectionProxyDictionary` 默认是 nil，含义是跟随系统设置。于是一次 DNS 失败被代理翻译成了一个货真价实的 HTTP 502 响应，`error` 是 nil。绕开代理之后也不是 -1003，而是 -1005（因为域名被 fake-IP 解析到了一个连不上的地址）。

同一个"域名解析不了"的事实，在这台机器上有两种完全不同的表现，都不是教程里写的那个 code。用户手机上装了 VPN、企业 MDM 配了 PAC、运营商做了 DNS 劫持，都会产生类似的效果。所以按 error code 白名单做重试是必要的，但它是尽力而为，不要指望它覆盖你没见过的网络环境。真正的兜底是幂等性和上限次数。

---

## 九、AFNetworking 在 URLSession 之上加了什么

先把状态说清楚。**AFNetworking 已经归档，最后一次功能提交在 2020-12-20，2023-01-17 那次提交叫 "Mark for deprecation"。** README 第一行就是 "AFNetworking is Deprecated"，官方给的迁移路径是 Alamofire。`AFNetworking/` 目录下 14 个文件（6 个 .m 加 8 个 .h）共 6534 行。

把那 6 个 .m 用当前 SDK 编一遍：0 error，15 个 deprecation warning。它今天还能编。用到的过时 API 集中在两处：

- `AFNetworkReachabilityManager.m` 里 6 处 `SCNetworkReachability*`。这套 API 在 macOS 14.4 / iOS 17.4 被标弃用，SDK 头文件给的替代说明是：`Use URLSession or NWConnection to create connections that dynamically handle changing networks. Use NWPathMonitor to enumerate available network interfaces.`
- `AFSecurityPolicy.m` 里 4 处 `SecTrustGetCertificateAtIndex` / `SecTrustCopyPublicKey`。另外它对 `SecTrustEvaluate` 的调用（第 65、91、127 行）被 `#pragma clang diagnostic ignored "-Wdeprecated-declarations"` 包住，所以没进警告统计——那个函数在 iOS 13 就该换成 `SecTrustEvaluateWithError` 了。

下面是它真正提供的东西，按我认为的价值排序。

#### 响应校验与序列化

`AFHTTPResponseSerializer` 的 init：

```objc
self.acceptableStatusCodes = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(200, 100)];
self.acceptableContentTypes = nil;
```

`validateResponse:data:error:` 里，status 不在 200..<300 就构造一个 error，域是 `AFURLResponseSerializationErrorDomain`，code 是 `NSURLErrorBadServerResponse`，原始 response 和 data 一并塞进 userInfo。content-type 不在集合里则用 `NSURLErrorCannotDecodeContentData`。`AFJSONResponseSerializer` 把 `acceptableContentTypes` 设成 `application/json` / `text/json` / `text/javascript`。

这就是第八节开头那个缺口的补法。它把"HTTP 层面的失败"翻译成了 `NSError`，让 `if (error)` 这个写法重新变得正确。我认为这是 AFNetworking 最大的一份价值，也是最容易自己写的一份，五十行以内。

#### 请求序列化

`AFPercentEscapedStringFromString` 那段实现值得一看：

```objc
static NSString * const kAFCharactersGeneralDelimitersToEncode = @":#[]@";
static NSString * const kAFCharactersSubDelimitersToEncode = @"!$&'()*+,;=";
...
static NSUInteger const batchSize = 50;
...
range = [string rangeOfComposedCharacterSequencesForRange:range];   // To avoid breaking up character sequences such as 👴🏻👮🏽
```

它没有直接调 `stringByAddingPercentEncodingWithAllowedCharacters:`。而是分 50 个字符一批，再用 `rangeOfComposedCharacterSequencesForRange:` 对齐到完整的字素簇。源码里那行 FIXME 指向的 PR #3028，说的是老系统上处理大量 emoji 时的崩溃。这类东西是库的真实价值：它替你踩过了你不会想到的坑。

`AFHTTPRequestSerializer` 还用 KVO 观察自己的六个属性并同步到每个 `NSMutableURLRequest` 上：

```objc
_AFHTTPRequestSerializerObservedKeyPaths = @[allowsCellularAccess, cachePolicy,
    HTTPShouldHandleCookies, HTTPShouldUsePipelining, networkServiceType, timeoutInterval];
```

多媒体上传那部分（multipart 边界、`UTTypeCreatePreferredIdentifierForTag` 推 MIME、流式 body）是自己写起来最烦的一块。

#### 安全策略

`AFSecurityPolicy` 提供三种 `AFSSLPinningMode`：`None` / `PublicKey` / `Certificate`，默认 `None`、`validatesDomainName = YES`。证书 pinning 这件事今天有了替代品。在 Info.plist 的 `NSAppTransportSecurity` 下配一个 `NSPinnedDomains`，把 CA 或叶证书公钥的 SPKI SHA-256 摘要写进去就行，声明式、不写代码、系统实现。新项目我不会为了 pinning 引 AFNetworking。

#### 队列管理

`AFURLSessionManager` 干的事一句话讲得完。把 `URLSession` 的"一个 delegate 服务所有 task"改造成"每个 task 一个 delegate"：

```objc
self.operationQueue = [[NSOperationQueue alloc] init];
self.operationQueue.maxConcurrentOperationCount = 1;   // 第 494 行
...
self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];
self.lock = [[NSLock alloc] init];                      // 第 506 行
```

`delegateForTask:` / `setDelegate:forTask:` / `removeDelegateForTask:` 三个方法全部用这把 `NSLock` 保护字典。回调分发用了一个并发队列做反序列化、一个 group 做完成通知：

```objc
dispatch_async(url_session_manager_processing_queue(), ^{
    responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];
    ...
    dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(),
                         manager.completionQueue ?: dispatch_get_main_queue(), ^{ ... });
});
```

JSON 解析在后台并发队列，completion 默认回主队列。这个分工今天仍然是对的，也是自己封装时最容易漏的一步。

#### 可达性

`AFNetworkReachabilityManager` 是我认为今天最不该用的那部分。Apple 在弃用说明里直接给了态度：用 `URLSession` 去发请求、让它自己处理网络变化，需要枚举接口再用 `NWPathMonitor`。"先查可达性再决定发不发请求"这个模式本身就是竞态：你查到的是过去某一刻的状态。`URLSessionConfiguration` 有 `waitsForConnectivity`（默认 NO），把它打开配合 `taskIsWaitingForConnectivity` 回调，比自己轮询可达性靠谱。

### 今天新项目还该不该引

我的判断：不该引 AFNetworking。理由不是"它旧了"。是三条具体的：它依赖两组已弃用的系统 API，其中一处还靠 pragma 压着警告；它最有价值的响应校验部分自己写不到一百行；Swift 项目里它还要额外背一个桥接成本。

Alamofire 的情况不同。它在维护，最近一次提交 2026-06，17066 行。`Source/Features` 下 18 个文件把拦截器、重试、认证、重定向、缓存策略、服务端信任、压缩都拆成了独立协议。如果项目已经是 Swift，我会引它，主要为的是 `RequestInterceptor` 这一层抽象和 `RetryPolicy` 那份把 48 个 error code 逐条论证过的清单。

已有的 ObjC 老项目还在用 AFNetworking 就继续用，它没有坏。README 里官方建议的做法是把源码拷进项目直接编，这样至少你能自己修。

---

## 十、我会怎么分

前面九节的每个结论都对应一条边界。把它们摊开，分层就不用画方框了。

接口数在 20 个以内、两三个人的项目，我不建独立的请求层。一个 `APIClient` 单例，一个描述 endpoint 的枚举或者一组常量，直接持有一个 `NSURLSession`。这个规模下多加一层的收益是负的，你会花更多时间在往上传参数上。

超过 20 个接口，请求层要独立出来。但它只该做三件事：把业务参数拼成 `NSURLRequest`；发出去；把 `(NSData, NSHTTPURLResponse, NSError)` 这三个各自可空的东西归一成一个非此即彼的结果类型。第三件是重点。第八节那张表说明了原始三元组有多难用：404 的 error 是 nil，cancel 的 status 是 200。

再往上，我的划法是：

- 重试和取消绑在一起，放同一层，而且是请求层的上边缘。理由是第八节那个 bug。这一层对外暴露的取消句柄必须覆盖所有尝试加所有退避窗口。
- 业务层不该看到 HTTP 状态码，401 除外。我知道有种说法是网络层要完全屏蔽 HTTP 细节，这个说法我不同意。401 必须穿透，因为只有业务层知道"此刻该弹登录页还是静默刷 token 再重放"。429 的 `Retry-After` 也该穿透到重试层。其余的 4xx/5xx 翻译成领域错误就行。
- 业务缓存不要用 `URLCache`。第二节那条实验是硬理由：缓存 key 不含 `Authorization`，账号一切换就串数据。我自己的做法是让服务端对所有带认证的接口发 `no-store`，然后在模型层做自己的缓存，key 里显式带上用户标识。`URLCache` 留给它擅长的场景：不带认证的静态资源、配置文件、CDN 上的东西。这三类反而应该主动让服务端配好 `max-age`，白拿一层缓存。
- 不要给每个业务模块建独立 session。缓存这个维度它隔离不了（第二节），并发上限这个维度它反而会让你悄悄突破（第六节）。要隔离就明确地换掉 `URLCache` 和 `HTTPCookieStorage`。我自己只分两个 session：一个常规的，一个 background 的。

关于超时，我给的具体值是：`timeoutIntervalForRequest` 15 秒，`timeoutIntervalForResource` 按接口分档，普通 API 30 秒、上传 300 秒。默认那个 604800 秒等于没设，第四节 B 那一行说明了只设第一个的后果。

关于 delegate 还是 completionHandler。普通 API 用 completionHandler。需要进度、需要断点续传、需要在取消时保留已收数据的，用 delegate（第七节）。这两个不是风格偏好，是能力差异。

最后一条不是分层，是方法。这篇里几乎每个结论都跑了一遍。跑之前我对其中至少四条的预期是错的：ephemeral 完全不缓存、background 在命令行程序里跑不起来、`timeoutIntervalForRequest` 是总时长、301 会保留 POST body。网络这块的中文资料同质化很严重，验证成本却出奇地低。一个 Python 文件，一句 `clang -framework Foundation`。

---

## 总结

`URLSession` 默认带一层 HTTP 缓存。`sharedURLCache` 在这台机器上是 0.5 MB 内存加 19.1 MB 磁盘，磁盘那半是 `~/Library/Caches/<id>/Cache.db` 这个 SQLite 库。它的缓存 key 不包含 `Authorization`，也不区分 session 实例。所以它不适合存带认证的业务数据。404 会被缓存。304 你永远看不到。

超时有两个。`timeoutIntervalForRequest` 是空闲计时器，只要对端持续吐数据它永远不触发；限制总时长要靠 `timeoutIntervalForResource`，默认 7 天。两者报的都是 -1001。

301/302 会把 POST 降级成 GET 并丢掉 body，307/308 不会。任何一次重定向都会剥掉 `Authorization`，同主机也不例外，自定义头则保留。`HTTPMaximumConnectionsPerHost` 默认 6，是每 session 的。task 一旦创建，回调一定会走一次，cancel 之后 completionHandler 版拿不到已收到的数据，delegate 版能。

重试和取消必须在同一层，否则退避窗口就是漏洞——这条是我自己写的封装里跑出来的 bug，不是推理出来的。按 error code 做重试白名单是必要的，但一个系统代理就能把 DNS 失败变成 HTTP 502，所以真正的兜底是幂等性和次数上限。

AFNetworking 已归档，今天新项目不该引；但它的响应校验（2xx + content-type）补的是 `URLSession` 一个真实的缺口，自己写一份不到一百行。

这一篇也是整个系列正文的最后一篇。收口的那份把前面所有结论落到一个具体 App 上：[[iOS 综合项目设计文档：把这个系列用一遍]]。

---

## 参考资料

### 官方

- [URL Loading System](https://developer.apple.com/documentation/foundation/url-loading-system)：入口
- [URLCache](https://developer.apple.com/documentation/foundation/urlcache)：明确说了它是 in-memory 和 on-disk 的复合缓存，以及 iOS 上磁盘部分可能在 App 未运行时被系统清掉
- [timeoutIntervalForRequest](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/timeoutintervalforrequest)：`The timer associated with this value is reset whenever new data arrives.` 这一句是第四节的依据
- [timeoutIntervalForResource](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/timeoutintervalforresource)：`The default value is 7 days.`
- [ephemeral](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/ephemeral)：`session-related data is stored in RAM`
- [httpMaximumConnectionsPerHost](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/httpmaximumconnectionsperhost)：`This limit is per session`
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)：301/302 与 307/308 对方法的处理差异，以及 Retry-After 的语义
- [Apple Developer News — 证书 pinning 的配置方式](https://developer.apple.com/news/?id=g9ejcf8y)：`NSPinnedDomains` 的完整 Info.plist 写法

### 源码

- [AFNetworking](https://github.com/AFNetworking/AFNetworking)（已归档）：`AFURLResponseSerialization.m` 第 102 行的 `NSMakeRange(200, 100)`、`AFURLRequestSerialization.m` 第 47-72 行的百分号编码、`AFURLSessionManager.m` 第 493-506 行的队列与锁、`AFSecurityPolicy.m` 第 63-128 行的 pragma
- [Alamofire](https://github.com/Alamofire/Alamofire)：`Source/Features/RetryPolicy.swift` 是我见过对"哪些错误可重试"论证最完整的一份清单，48 个 error code 逐条写了启用或禁用的理由

### 本地

- [[iOS 架构模式：MVC 到 MVVM，以及它们各自解决不了的问题]]
- [[iOS SDWebImage：下载、解码与两级缓存的完整链路]]
- [[iOS 持久化选型：沙盒、plist、NSUserDefaults 与 Keychain]]
- [[iOS 序列化：NSCoding、NSSecureCoding 与 JSON 三条路]]

---

实验环境：macOS 26.5.2（arm64，Apple Silicon），Apple clang 21.0.0，客户端全部用 `clang -fobjc-arc -framework Foundation` 编成原生 macOS 二进制直接跑，没有开任何模拟器。服务端是一个自己写的 Python 3.13 `ThreadingTCPServer`，`protocol_version = "HTTP/1.1"`，监听 127.0.0.1:8931。它把每一次真实到达的请求连同 method、path、并发数、`Content-Length` 和几个关心的请求头写进日志。TLS 实验另起一个 server，带自签证书，在 8943 端口。

有一处环境特异性要说明。这台机器开着系统级 HTTP 代理，127.0.0.1:7897，fake-IP DNS。`URLSession` 的 `connectionProxyDictionary` 默认跟随系统设置。所以第八节里 DNS 相关的 error code 结论只对这台机器成立。

> 待真机补测（一）：`URLCache.sharedURLCache` 的默认 `memoryCapacity` / `diskCapacity` 在 iOS 上和 macOS 未必相同。复现方法：真机跑一个空 App，`viewDidLoad` 里打印这两个值，同时打印 `NSSearchPathForDirectoriesInDomains(NSCachesDirectory, ...)` 下的 `Cache.db` 路径。

> 待真机补测（二）：蜂窝/WiFi 切换过程中的表现。`allowsCellularAccess`、`allowsExpensiveNetworkAccess`、`allowsConstrainedNetworkAccess` 三个开关，以及 `waitsForConnectivity = YES` 时 `URLSession:taskIsWaitingForConnectivity:` 的触发时机，在 macOS 有线网络上都测不出来。复现方法：真机开飞行模式发请求看 `waitsForConnectivity` 是否阻塞而不是立刻返回 -1009，再在传输中途从 WiFi 切到蜂窝，观察正在跑的 task 是断开（-1005）还是无缝续上。

> 待真机补测（三）：background session 的完整生命周期。本文只证明了传输由 `nsurlsessiond` 出进程执行、进程退出后仍能完成。App 被系统杀死后靠 `application:handleEventsForBackgroundURLSession:completionHandler:` 唤醒、`discretionary = YES` 时系统实际挑选的时机、以及低电量模式下的行为，都需要真机。复现方法：真机做一个大文件 background download，退到后台后用 Xcode 断开调试器并等待 App 被回收，观察是否被唤醒。
