---
title: 【iOS】持久化选型：沙盒、plist、NSUserDefaults 与 Keychain
published: 2026-07-27
description: setObject:forKey: 返回之后，磁盘上什么都没发生。实测稳态要等十秒，而 synchronize 一毫秒都没提前。从落盘时机、plist 的边界、备份排除标志到 Keychain 的四件套，把选型建立在能测出来的东西上。
tags:
  - iOS
  - Foundation
  - Persistence
  - Keychain
  - plist
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 30
draft: true
---
# 持久化选型：沙盒、plist、NSUserDefaults 与 Keychain

我原本打算把这篇写成一张对比表。写到一半发现，网上那几千张表里没有一张告诉我：`setObject:forKey:` 返回的那一刻，磁盘上到底发生了什么。

答案是什么都没发生。

```text
round 1  sync=0  setObject=3.329ms  落盘=3.4ms
round 2  sync=0  setObject=0.226ms  落盘=2017.5ms
round 3  sync=0  setObject=0.331ms  落盘=9996.8ms
round 4  sync=0  setObject=0.339ms  落盘=10012.5ms
round 5  sync=0  setObject=0.304ms  落盘=9983.4ms
--- 下面几轮显式调 synchronize ---
round 6  sync=1  setObject=0.471ms  落盘=10027.9ms
round 7  sync=1  setObject=0.715ms  落盘=9986.5ms
round 8  sync=1  setObject=0.387ms  落盘=9988.7ms
```

这一篇讲存储介质本身：数据在磁盘上长什么样、什么时候真的写下去、边界在哪里失效。编解码那一层（`NSCoding`、`NSSecureCoding`、JSON）在 [[iOS 序列化：NSCoding、NSSecureCoding 与 JSON 三条路]]。

先说清楚一件事。这一篇比系列里任何一篇都依赖平台。下面所有实验都是 macOS 26.5 / arm64 原生二进制跑的，而沙盒路径、文件保护的执行语义、Keychain 卸载后是否残留这三件事，iOS 和 macOS 有实质差异，我会逐条标出来。凡是标了 `> 待真机补测` 的，就是我在这台机器上确实没法验的。

---

## 一、`setObject:` 之后，你的数据在谁手里

上面那张表的测法很朴素：写一个唯一值进 `NSUserDefaults`，然后每毫秒直接用 `NSData` 读一次 `~/Library/Preferences/<domain>.plist` 的原始字节，用 `NSPropertyListSerialization` 解开看值到没到。不走 `objectForKey:`，因为那条路会命中缓存，测不出磁盘状态。

第一轮我差点写错结论。当时 plist 文件还不存在，`setObject:` 之后 3.4 ms 磁盘上就有了值，我几乎要写下"macOS 上是同步写的"。跑第二轮才发现那是文件首次创建的特例。**稳态是十秒左右，而且 `synchronize` 一毫秒都没提前。**

`synchronize` 的头文件注释把这件事说得很直白：

> `-synchronize` blocks the calling thread until all in-progress set operations have completed. This is no longer necessary. Replacements for previous uses of `-synchronize` depend on what the intent of calling synchronize was. If you synchronized...
> - ...before reading in order to fetch updated values: remove the synchronize call
> - ...for any other reason: remove the synchronize call

它从来不是"flush 到磁盘"，是"等我这个进程发出去的写操作全部完成"。发到哪里？发给 `cfprefsd`。

### 数据其实不在你的进程里

这条能测。写一个值，然后立刻给自己发 `SIGKILL`，不给任何 `atexit` 或者 flush 的机会：

```objc
[ud setObject:@(argv[3]) forKey:key];
kill(getpid(), SIGKILL);
```

```text
已 setObject:，进程即将 被 SIGKILL
退出码 137
立刻读:
磁盘文件里 killed = (不存在)
经 NSUserDefaults 读 killed = KILLVALUE
等 15 秒后再读:
磁盘文件里 killed = KILLVALUE
经 NSUserDefaults 读 killed = KILLVALUE
```

那两次"读"是另一个进程跑的。磁盘上没有，但通过 `NSUserDefaults` 读得到 `KILLVALUE`。值被 `cfprefsd` 接住了，写它的那个进程已经死了十几秒。

这解释了为什么 `synchronize` 会被废弃。你的进程崩掉不会丢数据。数据早就不归你管了。真正会丢的场景只剩一个：整机断电或者内核 panic，且刚好落在那十秒窗口里。

头文件里描述 `NSUserDefaults` 的第一句就点明了这个架构：

> NSUserDefaults is a hierarchical persistent interprocess (optionally distributed) key-value store, optimized for storing user settings.

`interprocess` 这个词是有分量的。App 和它的 Extension 共享同一份 defaults，靠的就是中间这个守护进程。

### 落盘时是原子替换

每一轮的 inode 都变了：

```text
round 2  inode 151482985 -> 151483010  (inode 变了)
round 3  inode 151483010 -> 151483160  (inode 变了)
round 4  inode 151483160 -> 151483252  (inode 变了)
```

`cfprefsd` 写的是临时文件加 rename，不是原地覆盖。这和第六节的 `atomically:YES` 是同一套机制。所以 plist 文件不会出现"写了一半"的中间状态，掉电只会丢掉最后一次写，不会得到一个损坏的文件。

### 冷热差一万倍

```text
+standardUserDefaults 首次     0.008 ms
第一次 objectForKey:          3.462 ms
后续 objectForKey: 均值        0.00035 ms
setObject: 均值(1000 次)       0.07588 ms
```

`+standardUserDefaults` 本身几乎不花钱。第一次真正读一个 key 要 3.5 到 8 ms（两次运行分别是 3.462 和 8.009），那是一次 XPC 往返加上把整个域拉进进程内缓存。之后每次读 0.35 微秒，纯内存查表。

这个数字对启动优化有直接意义。启动路径上第一次碰 `UserDefaults` 是毫秒级开销，跟碰多少个 key 无关。所以"把 defaults 读取分散到各个模块"不会更便宜，"启动时少读几个 key"也不会更便宜。要省，只能整条不碰。

---

## 二、搜索域：`registerDefaults:` 的值永远不落盘

`NSUserDefaults` 不是一个字典，是一条搜索链。头文件按优先级从高到低列了十一层，其中和日常开发相关的是这几层：

> - Managed ("forced") preferences, set by a configuration profile or via mcx from a network administrator
> - Commandline arguments
> - Preferences for the current domain, the current user, in any host
> - Preferences added via `-addSuiteNamed:`
> - Preferences global to all apps for the current user, in any host
> - Preferences registered with `-registerDefaults:`

注册域在最后一层。这是它能当兜底值用的原因。

跑一遍看清楚。注册 `theme=dark`、`pageSize=20`，然后只把 `theme` 写进应用域：

```text
theme    = light   （注册域 dark，应用域 light）
pageSize = 20  （只存在于注册域）

volatileDomainNames = (
    NSRegistrationDomain,
    NSArgumentDomain
)

NSRegistrationDomain 内容 = {
    AppleLanguages = ("en-001");
    AppleLocale = "en_001";
    NSInterfaceStyle = macintosh;
    NSLanguages = ("en-001");
    pageSize = 20;
    theme = dark;
}

本应用持久域（persistentDomainForName:ud4）= {
    theme = light;
}

dictionaryRepresentation 共 82 个键；含 theme=light pageSize=20
```

等十二秒之后看磁盘：

```text
$ plutil -p ~/Library/Preferences/ud4.plist
{
  "theme" => "light"
}
```

磁盘上只有 `theme`。`pageSize` 一个字节都没写。头文件的措辞是 "Registered defaults are never stored between runs of an application, and are visible only to the application that registers them."

所以 `registerDefaults:` 必须在每次启动时重新调用，通常放在 `application:didFinishLaunchingWithOptions:` 里。漏调的后果不是崩溃，是 `boolForKey:` 悄悄返回 `NO`。这类 bug 很难查。它只在"用户从没改过这个设置"的路径上出现，而开发机上你早就改过一次了。

注册域里除了我写的两个键，还有系统塞进去的 `AppleLanguages`、`AppleLocale`、`NSInterfaceStyle`。注意它们的值和全局域并不一致：注册域是 `en-001`，`NSGlobalDomain` 是 `zh-Hans-CN`。搜索链先命中全局域，注册域那份只是最后的兜底。

`volatileDomainNames` 只有两项，因为易失域就只有注册域和参数域。`persistentDomainNames` 从 macOS 10.9 / iOS 7 起就标了 `API_DEPRECATED("Not recommended")`，头文件自己的说明是 "returns an incomplete list of domains"，别用它做遍历。

`NSArgumentDomain` 值得知道存在。命令行传 `-key plistvalue` 就能临时覆盖任何 key，优先级仅次于配置描述文件。头文件说 "This can be useful for testing purposes."

---

## 三、plist 装得下什么

`NSUserDefaults` 的类型约束就是 plist 的类型约束，头文件写得很清楚：

> Key-Value Store: NSUserDefaults stores Property List objects (NSString, NSData, NSNumber, NSDate, NSArray, and NSDictionary) identified by NSString keys, similar to an NSMutableDictionary.

六种。挨个试一遍，再试几个常见的非法类型：

```text
=== 六种合法类型 ===
NSString               isValid=1  序列化成功 223 字节
NSNumber               isValid=1  序列化成功 225 字节
NSNumber(BOOL)         isValid=1  序列化成功 211 字节
NSDate                 isValid=1  序列化成功 237 字节
NSData                 isValid=1  序列化成功 225 字节
NSArray                isValid=1  序列化成功 265 字节
NSDictionary           isValid=1  序列化成功 257 字节

=== 非法类型 ===
NSNull                 isValid=0  失败
自定义对象              isValid=0  失败
NSURL                  isValid=0  失败
NSSet                  isValid=0  失败
NSValue(range)         isValid=0  失败

=== 字典 key 必须是 NSString ===
{@1: @"a"} -> The data couldn’t be written because of an error in the destination for the data.

=== NSUserDefaults 存非法类型会怎样 ===
抛异常 NSInvalidArgumentException: Attempt to insert non-property list object <Plain: 0x101324f10> for key bad
```

几个容易踩的点。`NSNull` 不合法，所以 JSON 解析出来的字典里只要有一个 `null`，直接丢进 `NSUserDefaults` 就会抛异常，这个 crash 我见过不止一次。`NSSet` 不合法，得自己转成数组。字典的 key 必须是字符串，`@{@1: @"a"}` 写不出去。

`NSURL` 也不合法，但 `NSUserDefaults` 给它开了后门：`setURL:forKey:` / `URLForKey:` 内部会把 URL 归档成 `NSData` 再存。所以你能存 URL，只是存进去的东西已经不是 URL 了。直接用 `objectForKey:` 读回来会拿到一坨 `NSData`。

`NSPropertyListSerialization` 的报错信息基本没用（"an error in the destination for the data"），定位得靠 `propertyList:isValidForFormat:` 自己先查一遍。

### 两种格式，四点六倍

同一份数据，200 条记录，每条五个字段：

```text
XML    = 49147 字节
Binary = 10623 字节   （4.63x）
```

小数据差距更夸张。`{"a":1,"b":"hello","c":[1,2]}` 三个键，XML 341 字节，binary 74 字节。XML 光是 DOCTYPE 那一行就 100 多字节。

头部一眼能认出来：

```text
small_bin:
00000000: 6270 6c69 7374 3030 d301 0203 0405 0651  bplist00.......Q
00000010: 6151 6251 6310 0155 6865 6c6c 6fa2 0407  aQbQc..Uhello...

small_xml:
00000000: 3c3f 786d 6c20 7665 7273 696f 6e3d 2231  <?xml version="1
```

`bplist00` 那八个字节是二进制 plist 的魔数。读回来时 `format` 出参会告诉你原文件是哪种，`NSPropertyListBinaryFormat_v1_0` 的枚举值是 200，XML 是 100。

现在是这一节真正有用的部分。`[NSDictionary writeToURL:error:]` 写出来的是什么格式？

```text
via_writeToURL.plist: XML 1.0 document text, ASCII text
-rw-r--r--  49147  big_xml.plist
-rw-r--r--  10623  big_bin.plist
-rw-r--r--  49147  via_writeToURL.plist
```

XML。跟手动指定 `NSPropertyListXMLFormat_v1_0` 一字节不差。

而 `cfprefsd` 写的 `Preferences` 目录，我抽了八个文件，头四个字节全是 `62706c69`，也就是 `bplist`：

```text
8DKG4XB37M.group.com.nektony.App-Cleaner-SIII.plist   62706c6973743030
abnerworks.Typora.plist                               62706c6973743030
adobe.com.Adobe-Spaces-Helper.plist                   62706c6973743030
ai.lody.desktop.plist                                 62706c6973743030
```

系统给自己用二进制，给你的便利方法用 XML。所以只要你是拿 `writeToURL:` 直接写 plist 文件的，体积默认就多付四倍多，而且没有任何提示。要二进制得自己绕一圈：

```objc
NSData *d = [NSPropertyListSerialization dataWithPropertyList:root
                    format:NSPropertyListBinaryFormat_v1_0 options:0 error:&e];
[d writeToURL:url atomically:YES];
```

多两行，省 78% 的磁盘和 IO。我自己的做法是：项目里凡是写 plist 的地方一律走这个封装，`writeToURL:` 只在调试时用，因为 XML 能直接 `cat`。

---

## 四、归档产物也是一个 plist

`NSKeyedArchiver` 严格说属于下一篇的范围，但它落到磁盘上的东西属于这一篇。归档一个三字段的对象，然后直接拿 plist 反序列化器去打开它：

```text
归档 298 字节
当成 plist 解析: 成功，format=200
顶层 keys = ($archiver, $objects, $top, $version)
$archiver = NSKeyedArchiver   $version = 100000
$objects 元素个数 = 7
```

`format=200` 就是 `NSPropertyListBinaryFormat_v1_0`。归档文件是一个 bplist，`plutil -p` 直接能打开：

```text
{
  "$archiver" => "NSKeyedArchiver"
  "$objects" => [
    0 => "$null"
    1 => {
      "$class" => <CFKeyedArchiverUID>{value = 6}
      "age" => 27
      "name" => <CFKeyedArchiverUID>{value = 2}
      "tags" => <CFKeyedArchiverUID>{value = 3}
    }
    2 => "tommy"
    3 => {
      "$class" => <CFKeyedArchiverUID>{value = 5}
      "NS.objects" => [
        0 => <CFKeyedArchiverUID>{value = 4}
        1 => <CFKeyedArchiverUID>{value = 2}
      ]
    }
    4 => "ios"
    5 => { "$classes" => ["NSArray", "NSObject"], "$classname" => "NSArray" }
    6 => { "$classes" => ["User", "NSObject"],    "$classname" => "User" }
  ]
  "$top" => { "root" => <CFKeyedArchiverUID>{value = 1} }
  "$version" => 100000
}
```

`$objects` 是一张扁平的对象表，所有引用都变成下标。我给这个对象的 `name` 是 `"tommy"`，`tags` 是 `@[@"ios", @"tommy"]`，字符串 `"tommy"` 故意重复了一次。表里它只出现一次，在下标 2，被 `name` 和 `tags[1]` 各引用一次。归档器做了去重。

还有一条会影响选型判断：`requiringSecureCoding:YES` 和 `NO` 产出的字节完全一样。

```text
secure=YES 209 字节, secure=NO 209 字节, 字节完全相同=1
非 secure 归档 -> secure 解档: 成功
```

`NSSecureCoding` 是解档时的策略，不影响磁盘上的一个字节。展开留给下一篇。

---

## 五、沙盒目录，以及 macOS 上你看到的不是那回事

这一节我不敢凭印象写，抄 File System Programming Guide 的原文。

> - Put user data in `Documents/`. User data generally includes any files you might want to expose to the user—anything you might want the user to create, import, delete or edit.
> - Put app-created support files in the `Library/Application support/` directory. In general, this directory includes files that the app uses to run but that should remain hidden from the user.
> - Remember that files in `Documents/` and `Application Support/` are backed up by default. You can exclude files from the backup by calling `-[NSURL setResourceValue:forKey:error:]` using the `NSURLIsExcludedFromBackupKey` key. Any file that can be re-created or downloaded must be excluded from the backup.
> - Put temporary data in the `tmp/` directory... The system will periodically purge these files when your app is not running; therefore, you cannot rely on these files persisting after your app terminates.
> - Put data cache files in the `Library/Caches/` directory... Note that the system may delete the `Caches/` directory to free up disk space, so your app must be able to re-create or download these files as needed.

同一份文档里那张 iOS 目录表补齐了备份维度：

| 目录 | 官方原文要点 |
|---|---|
| `AppName.app` | "You cannot write to this directory... The contents of this directory are not backed up by iTunes or iCloud." |
| `Documents/` | "Use this directory to store user-generated content... The contents of this directory are backed up by iTunes and iCloud." |
| `Documents/Inbox` | "Your app can read and delete files in this directory but cannot create new files or write to existing files." 会被备份。 |
| `Library/` | "The contents of the `Library` directory (with the exception of the `Caches` subdirectory) are backed up by iTunes and iCloud." |
| `tmp/` | "the system may purge this directory when your app is not running. The contents of this directory are not backed up by iTunes or iCloud." |

四条判断规则可以从这张表直接读出来。能被用户看到并管理的进 `Documents`；App 自己要用但用户不该看到的进 `Library/Application Support`；丢了能重新下载的进 `Library/Caches`；这次运行完就不要了的进 `tmp`。

`Caches` 和 `tmp` 的区别经常被讲混。`tmp` 是"App 不在运行时系统会清"，`Caches` 是"磁盘紧张时系统可能清"。前者的清理更激进。两者都不进备份。

### macOS 上跑出来是这样的

```text
NSHomeDirectory()        /Users/tommywu
NSTemporaryDirectory()   /var/folders/qw/3___bn4n2vn83fx2gq2ycrn00000gn/T/
Documents                /Users/tommywu/Documents
Library                  /Users/tommywu/Library
Library/Caches           /Users/tommywu/Library/Caches
Application Support      /Users/tommywu/Library/Application Support
mainBundle.bundlePath    /private/tmp/persist
```

一个未沙盒化的命令行工具，`NSHomeDirectory()` 就是真正的用户家目录，`Documents` 是我桌面上那个 Documents。这跟 iOS 完全是两回事。iOS 上这些 API 返回的是 App 数据容器里的子目录。

macOS 上开了 App Sandbox 之后才接近 iOS 的形态。这台机器上现成就有：

```text
$ ls ~/Library/Containers/-1.-------pro/Data
Desktop  Documents  Downloads  Library  Movies  Music  Pictures  SystemData  tmp
```

结构像，但多了 `Desktop`、`Downloads` 这些 iOS 上不存在的目录。

> 待真机补测：在 iPhone 上打印这六个路径，确认 iOS 8 之后 bundle 容器（`.app` 所在）与数据容器（`Documents`/`Library`/`tmp` 所在）确实是两个不同的根目录，且 `NSHomeDirectory()` 指向数据容器。复现方法：把 `dir1.m` 的前七行 `printf` 原样放进一个空工程的 `didFinishLaunching`，跑一次抄下输出。macOS 上的这组路径不能当成 iOS 的结论。

### `NSURLIsExcludedFromBackupKey` 实测

这个键在 macOS 上是可用的（`API_AVAILABLE(macos(10.8), ios(5.1))`），只是后端换成了 Time Machine。

```text
设置前 NSURLIsExcludedFromBackupKey = 0 (err=nil)
setResourceValue: 返回 1, err=nil
设置后读回 = 1
目录标了排除后，目录内新建文件自身的标志 = 1
```

它落在文件系统上是一个扩展属性：

```text
$ xattr -l /tmp/persist/backupflag/cache.bin
com.apple.metadata:com_apple_backup_excludeItem: bplist00_com.apple.backupd
```

xattr 的值本身又是一个 bplist，内容是 `com.apple.backupd`。

最后那行输出值得单独说。`later.bin` 是在目录被标记之后才创建的，我以为它会继承 xattr。`xattr -l` 一看根本没有：

```text
$ xattr -l backupflag/later.bin
com.apple.provenance:
```

但换一个全新进程去查资源值，它照样报 1：

```text
backupflag                               -> 1
backupflag/cache.bin                     -> 1
backupflag/later.bin                     -> 1
big_bin.plist                            -> 0        ← 无关文件对照
```

**标记落在目录上，查询时会向上继承，所以给缓存目录标一次就够了，不用每写一个文件标一次。** 这条我以前一直是逐文件标的，纯属白写。

`NSURL.h` 里那条注释是我这次才注意到的，很关键：

> This property is only useful for excluding cache and other application support files which are not needed in a backup. Some operations commonly made to user documents will cause this property to be reset to false and so this property should not be used on user documents.

对用户文档设这个标志不可靠，系统会在某些操作后把它重置。

> 待真机补测：以上是 Time Machine 后端的行为。iOS 上同一个键走的是 iCloud / iTunes 备份，"目录标记向上继承"是否同样成立需要在设备上验。复现方法：在 `Library/Caches` 下建一个目录，标记后往里放文件，跑一次加密备份，用 iMazing 之类的工具查看备份内是否包含该文件。

---

## 六、`atomically:YES` 到底做了什么

很短的一节，但它是上面好几处的共同底座。

```text
atomically:YES inode 151485677 -> 151485679 (变了)   size 16 -> 4096
atomically:NO  inode 151485678 -> 151485678 (不变)   size 16 -> 4096
```

`YES` 换了 inode，`NO` 没换。写临时文件再 `rename` 覆盖，`rename` 在同一个卷上是原子的，所以读者要么看到旧文件的完整内容，要么看到新文件的完整内容，不存在中间态。

inode 变了意味着一件容易忽略的事。做个硬链接实验：

```text
硬链接测试（atomically:YES）: 源=NEW  链接=OLD
```

`hl_link.bin` 还指着老的 inode，内容停留在 `OLD`。所以 `atomically:YES` 会切断硬链接，也会让已经打开的文件描述符和别的路径引用继续指向旧数据。用 `mmap` 映射过这个文件的代码同样看不到新内容。

权限位倒是保住了：

```text
权限位: 写前 600  atomically:YES 写后 600
```

Foundation 会把原文件的属性复制到临时文件再 rename。

第一节 `cfprefsd` 每轮都换 inode，走的就是同一套。

---

## 七、文件保护等级：macOS 上并不是不能测

我本来打算在这里写"iOS 专属，未测"。动笔前先问了一句这个卷支不支持，结果是这样的：

```text
NSURLVolumeSupportsFileProtectionKey = 1
当前 NSURLFileProtectionKey = NSURLFileProtectionCompleteUntilFirstUserAuthentication
设 NSURLFileProtectionComplete -> 1, err=nil
NSFileManager setAttributes: -> 1, err=nil
attributesOfItemAtPath 里有 NSFileProtectionKey 吗？ NSFileProtectionComplete
根卷 supportsFileProtection = 1
```

**macOS 26 / Apple Silicon 上这套 API 是通的，而且每个文件默认就带着 `CompleteUntilFirstUserAuthentication`。** 头文件也印证了这一点：`NSFileProtection*` 系列标的是 `API_AVAILABLE(macos(10.6), ios(4.0))`，`NSURLFileProtection*` 是 `macos(11.0), ios(9.0)`。整组常量里只有一个 iOS 专属：

```objc
FOUNDATION_EXPORT NSFileProtectionType const NSFileProtectionCompleteWhenUserInactive
    API_AVAILABLE(ios(17.0), watchos(10.0), tvos(17.0)) API_UNAVAILABLE(macos);
```

所以"文件保护是 iOS 专属"这个说法我不同意，至少在今天的 macOS 上不成立。

跨进程验一遍设置是否真的生效：

```text
set complete -> 1        当前 NSURLFileProtectionComplete
（新进程）               当前 NSURLFileProtectionComplete
set none -> 1            当前 NSURLFileProtectionCompleteUntilFirstUserAuthentication
（新进程）               当前 NSURLFileProtectionCompleteUntilFirstUserAuthentication
set unlessopen -> 1      当前 NSURLFileProtectionCompleteUnlessOpen
（新进程）               当前 NSURLFileProtectionCompleteUnlessOpen
```

`Complete` 和 `CompleteUnlessOpen` 都写进去了。`None` 是个陷阱。`setResourceValue:` 返回 `1`、`error` 是 `nil`，读回来却还是 `CompleteUntilFirstUserAuthentication`。降级到卷的默认等级以下这件事被静默忽略了，返回值骗人。

四个等级的语义抄 `NSURL.h` 原文，因为中文转述这几条经常把 `CompleteUnlessOpen` 讲反：

> NSURLFileProtectionNone: The file has no special protections associated with it. It can be read from or written to at any time.
>
> NSURLFileProtectionComplete: The file is stored in an encrypted format on disk and cannot be read from or written to while the device is locked or booting.
>
> NSURLFileProtectionCompleteUnlessOpen: Files can be created while the device is locked, but once closed, cannot be opened again until the device is unlocked. If the file is opened when unlocked, you may continue to access the file normally, even if the user locks the device. There is a small performance penalty when the file is created and opened, though not when being written to or read from.
>
> NSURLFileProtectionCompleteUntilFirstUserAuthentication: The file is stored in an encrypted format on disk and cannot be accessed until after the device has booted. After the user unlocks the device for the first time, your app can access the file and continue to access it even if the user subsequently locks the device.

`CompleteUnlessOpen` 是为后台下载设计的：锁屏状态下能新建并往里写，但一旦关闭就打不开了。后台任务要在锁屏时继续读已有文件，得用 `CompleteUntilFirstUserAuthentication`。

我能测的到此为止：API 通、默认值是什么、哪些等级写得进去。执行语义测不了。macOS 的"锁定"和 iOS 的"锁屏"不是一回事，我没法在这台机器上制造出"设备已锁定、密钥已从内存清除"的状态。

> 待真机补测：iOS 上把一个文件设成 `NSFileProtectionComplete`，用后台任务（`beginBackgroundTaskWithExpirationHandler:` 或者 `BGProcessingTask`）在锁屏后去读它，确认拿到的是 `NSFileReadNoPermissionError`（errno `EPERM`）。同一文件改成 `CompleteUntilFirstUserAuthentication` 重跑，确认锁屏后能读。还要单独验一条：iOS 上 App 的默认保护等级是不是也是 `CompleteUntilFirstUserAuthentication`，用本节 `fp2.m` 那段代码打印 `Documents` / `Library/Caches` / `tmp` 下新建文件的 `NSURLFileProtectionKey`。macOS 上的默认值不能当成 iOS 的默认值。

---

## 八、Keychain：四件套跑通了，但只跑通了一半

先说没跑通的那一半，因为它本身是个信息。

标准写法里应该带 `kSecUseDataProtectionKeychain: @YES`，这样 macOS 上用的才是和 iOS 同一套数据保护钥匙串。跑出来是这样：

```text
SecItemAdd                   status=-34018 (A required entitlement isn't present.)
SecItemCopyMatching          status=-25300 (The specified item could not be found in the keychain.)
SecItemUpdate                status=-34018 (A required entitlement isn't present.)
```

`-34018` 是 `errSecMissingEntitlement`。数据保护钥匙串要求进程带 `keychain-access-groups` entitlement，一个裸编的命令行二进制没有。

我试了两条路补救，都失败了。ad-hoc 签名不带 entitlement，还是 `-34018`。ad-hoc 签名带上 `keychain-access-groups`：

```text
codesign -d --entitlements - kc1
    [Key] keychain-access-groups
    [Value] [Array] [String] com.tommywu.persist-demo
=== 运行 ===
（无输出，退出码 137）
```

退出码 137 是 SIGKILL。系统直接把进程杀了。这个 entitlement 需要真实的开发者证书和 provisioning profile 背书，ad-hoc 签名声明它等于伪造。要真跑数据保护钥匙串，得建一个带 Team ID 的 Xcode 工程。

退回到 macOS 传统的文件式钥匙串（去掉 `kSecUseDataProtectionKeychain`），四件套全通：

```text
SecItemAdd                   status=0 (No error.)
SecItemAdd 第二次             status=-25299 (The specified item already exists in the keychain.)
SecItemCopyMatching          status=0 (No error.)
  值 = hunter2
  属性 keys = acct, cdat, class, labl, mdat, svce, v_Data
  accessible = (null)  cdat=2026-07-26 23:12:26 +0000
SecItemUpdate                status=0 (No error.)
  update 后的值 = hunter3
换 account 再 Add             status=0 (No error.)
只改 label 再 Add             status=-25299 (The specified item already exists in the keychain.)
SecItemDelete(整个 service)   status=0 (No error.)
再 CopyMatching              status=-25300 (The specified item could not be found in the keychain.)
```

三条能直接用的结论。

第一，Keychain 没有 upsert。重复 `SecItemAdd` 返回 `errSecDuplicateItem`（-25299），不会覆盖。正确写法是先 `SecItemUpdate`，拿到 `errSecItemNotFound`（-25300）再 `SecItemAdd`；或者先无脑 `SecItemDelete` 再 `Add`。我见过的封装十有八九是后者，简单，但会丢掉创建时间这类属性。

第二，主键由哪些属性组成，这个实验直接给出答案。同样的 service、换一个 `kSecAttrAccount`，添加成功，是两条独立记录；同样的 service 和 account、只改 `kSecAttrLabel`，报重复。**`kSecClassGenericPassword` 的主键里有 service 和 account，没有 label。** 这解释了一个常见现象：同一个 App 里两处代码用了同样的 service 和 account，后写的那次静默失败，读出来的永远是第一次存的值。

第三，`kSecAttrAccessible` 在传统钥匙串上读回来是 `(null)`。它是数据保护钥匙串的概念，传统钥匙串的访问控制走的是另一套（ACL 加上钥匙串本身是否解锁）。

`SecItemDelete` 那个查询字典里我只给了 `kSecClass` 和 `kSecAttrService`，没给 account，两条记录一次删干净。批量删按属性匹配，不需要遍历。

### `kSecAttrAccessible` 的六个等级

抄 `SecItem.h` 原文。这一组的差别全在"什么时候能读到"和"会不会跟着备份走"两个维度上：

> kSecAttrAccessibleWhenUnlocked: Item data can only be accessed while the device is unlocked. This is recommended for items that only need be accesible while the application is in the foreground. Items with this attribute will migrate to a new device when using encrypted backups.
>
> kSecAttrAccessibleAfterFirstUnlock: Item data can only be accessed once the device has been unlocked after a restart. This is recommended for items that need to be accesible by background applications. Items with this attribute will migrate to a new device when using encrypted backups.
>
> kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly: ... requires a passcode to be set on the device. Items with this attribute will never migrate to a new device... This attribute will not be available on devices without a passcode. Disabling the device passcode will cause all previously protected items to be deleted.
>
> kSecAttrAccessibleWhenUnlockedThisDeviceOnly / AfterFirstUnlockThisDeviceOnly: 语义同上面两条，但 "Items with this attribute will never migrate to a new device, so after a backup is restored to a new device, these items will be missing."

`kSecAttrAccessibleAlways` 和 `AlwaysThisDeviceOnly` 从 iOS 12 / macOS 10.14 起已经废弃，头文件给的替代建议是 `kSecAttrAccessibleAfterFirstUnlock`：

```objc
extern const CFStringRef kSecAttrAccessibleAlways
    API_DEPRECATED("Use an accessibility level that provides some user protection, such as kSecAttrAccessibleAfterFirstUnlock", macos(10.9, 10.14), ios(4.0, 12.0));
```

`ThisDeviceOnly` 那一组的取舍很实在。安全性更高，代价是用户换手机之后这条记录不存在。登录 token 用 `ThisDeviceOnly` 意味着换机必须重新登录，这未必是你想要的产品行为。`WhenPasscodeSetThisDeviceOnly` 最激进，用户一旦关掉锁屏密码，之前存的全部被删除。

还有一条头文件里的坑：

> When asking SecItemCopyMatching to return the item's data, the error `errSecInteractionNotAllowed` will be returned if the item's data is not available until a device unlock occurs.

后台任务读 `WhenUnlocked` 的项目会拿到 `errSecInteractionNotAllowed`（-25308），而不是数据。这是"我本地怎么测都好好的，线上一堆用户登录失败"的经典成因。

### 卸载之后还在不在

这条我没法在 macOS 上验。传统钥匙串和 App 生命周期根本没有绑定关系。

> 待真机补测：iOS 上 Keychain 数据在 App 卸载后是否残留。复现方法：真机装 App A，用 `kSecClassGenericPassword` 存一条 service = `com.x.test` 的记录，确认能读到；删除 App，重启设备（排掉缓存干扰）；重新安装同一个 Bundle ID、同一个 Team ID 的 App A，`SecItemCopyMatching` 同一条查询，看返回 `errSecSuccess` 还是 `errSecItemNotFound`。同时记录 iOS 版本号，因为这个行为历史上变过（iOS 10.3 的某个 beta 改成过卸载即清，正式版又回退了），任何一篇文章给的答案都必须绑定版本才有意义。
>
> 附带验第二条：把 App A 换成同 Team ID、不同 Bundle ID 的 App B，配上同一个 `keychain-access-groups`，看 B 能否读到 A 存的记录。这是"Keychain 跨 App 共享"的实际边界。

关于这一条，我现在能给的判断只到这里：不要把"卸载后 Keychain 数据保留"当成一个可以依赖的特性来设计功能。就算它今天是真的，它也是一个从未被 Apple 写进文档的行为。我见过用它做"免费试用只能用一次"的，用户重装绕不过，看起来很聪明，但这个功能的正确性完全押在一个未文档化的实现细节上。

---

## 九、我的阈值

前面的数据够支撑几条具体的线了。

**`NSUserDefaults`：单个值不超过 1 KB，整个域不超过 100 KB。** 理由是这个：

```text
落盘后 plist: 4194373 字节, inode 151542995
改一个 NSNumber 后: 4194390 字节, inode 151543065, 距 setObject 14683 ms, 文件被整体重写
```

域里躺着一个 4 MB 的 blob 时，改一个 `NSNumber` 会导致整个 4.19 MB 的文件被重写一遍（inode 变了）。plist 是一个整体，没有部分更新这回事。域里键的数量倒是无所谓，10 个键和 5000 个键，写一个小键的耗时都是 0.12 ms 上下。代价全在文件体积上。

另外，`removePersistentDomainForName:` 之后 plist 文件不会消失，只会缩到 42 字节的空壳。

能重新下载的东西一律进 `Library/Caches`，并且给那个目录标一次 `NSURLIsExcludedFromBackupKey`。官方原文是 "Any file that can be re-created or downloaded must be excluded from the backup"，用的是 must。图片缓存放 `Documents` 是我见过最常见的错误。后果是用户的 iCloud 备份被几百 MB 可重下的图片撑爆。

密码、token、加密密钥进 Keychain，其他一切不进。这里我想反驳一个流传很广的理由。常见说法是"UserDefaults 是明文 plist，谁都能看到"。在一台没越狱的 iPhone 上这句话不成立，别的 App 读不到你沙盒里的文件。真正的理由是另外三条：defaults 的 plist 跟着备份走，一份未加密备份就把 token 明文交出去了；`NSUserDefaults` 没有 accessibility 分级，你没法表达"锁屏时不可读"；卸载即消失，而 Keychain 的行为不同。

选等级的话，我的默认选择是 `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`。后台任务能读，换机不带走。如果产品明确要求换机免登录，降到 `AfterFirstUnlock`。`WhenUnlocked` 只给纯前台使用的秘密。

只整读整写、不需要查询的中等规模数据，用 plist 文件，格式选 binary。4.63 倍的体积差是白给的。一旦出现"按某个字段筛选""只更新一条记录"这类需求，就该换 SQLite 或 Core Data 了，因为 plist 的每次更新都是全量重写。我给自己划的线是几百 KB。超过这个量还在整体重写，说明数据结构选错了。

`Library/Application Support` 是最容易被漏掉的那个目录。不该给用户看、又不能丢的东西该放这里：数据库文件、下载的配置、生成的索引。很多人在 `Documents` 和 `Caches` 之间二选一，其实要的是第三个。

最后一条不算阈值，算习惯。写完持久化代码之后去看一眼磁盘上真实的文件。`plutil -p` 打开它，`xxd` 看头八个字节，`stat` 看 inode 变没变。这一篇里所有推翻我原有印象的结论，都是这么撞出来的。

---

## 总结

`setObject:forKey:` 返回时磁盘上什么都没有，稳态要等十秒，`synchronize` 一毫秒都没提前。数据在 `cfprefsd` 手里，所以你的进程被 SIGKILL 也不会丢。落盘时是临时文件加 rename，每次都换 inode。

`registerDefaults:` 的值永远不进 plist，每次启动都得重新调。plist 只认六种类型加字符串 key，`NSNull` 和 `NSSet` 都会让你崩在 `NSInvalidArgumentException` 上。`writeToURL:` 默认写 XML，比二进制大 4.63 倍，而系统给自己写的 `Preferences` 全是 `bplist00`。

沙盒四个目录的分界线在官方文档里写得很死：可重建的必须排除备份，`Caches` 会被系统清，`tmp` 清得更狠。`NSURLIsExcludedFromBackupKey` 标在目录上就够了，查询时会向上继承。

文件保护在 macOS 26 上是可用的，默认等级 `CompleteUntilFirstUserAuthentication`，往下降会被静默忽略。Keychain 的数据保护钥匙串需要真实 entitlement，裸二进制拿不到；传统钥匙串上四件套跑通了，主键含 service 和 account 但不含 label，重复 add 报 `errSecDuplicateItem`。

这一篇 macOS 和 iOS 的差距比系列里任何一篇都大。凡是涉及沙盒路径、保护等级执行语义、Keychain 卸载残留的结论，我都标了待补测。没有一条是从 macOS 的观察直接推到 iOS 的。

下一篇 [[iOS 序列化：NSCoding、NSSecureCoding 与 JSON 三条路]]。

## 参考资料

### 官方

- [File System Programming Guide — File System Basics](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html)：第五节两段引文的出处，包括 "Where You Should Put Your App's Files" 和 iOS 目录表
- [Using the File System Effectively](https://developer.apple.com/documentation/foundation/using-the-file-system-effectively)：计划里指定的入口，内容是上面那篇的现代版
- [Keychain Services](https://developer.apple.com/documentation/security/keychain-services)
- `NSUserDefaults.h`（MacOSX26.5.sdk）：搜索链的十一层、`synchronize` 的废弃说明、`registerDefaults:` 不持久化，全部抄自这个头文件
- `NSURL.h`（MacOSX26.5.sdk）：`NSURLFileProtection*` 四个等级的原文、`NSURLIsExcludedFromBackupKey` 那条"会被重置"的警告
- `NSFileManager.h`（MacOSX26.5.sdk）：`NSFileProtection*` 的可用性标注
- `SecItem.h`（MacOSX26.5.sdk）：`kSecAttrAccessible` 六个等级的原文与废弃提示

### 本地

- [[iOS 序列化：NSCoding、NSSecureCoding 与 JSON 三条路]]
- [[iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint]]
- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]

---

实验环境：macOS 26.5.2（build 25F84），Apple Silicon / arm64，Apple clang 21.0.0，SDK MacOSX26.5。全部是 macOS 原生二进制，没有开模拟器：

```shell
clang -fobjc-arc -framework Foundation -o out prog.m && ./out
clang -fobjc-arc -framework Foundation -framework Security -o kc prog.m && ./kc
```

落盘时机那组数据在同一台机器上跑了两轮八次，稳态的 10 秒是可复现的。这个数值属于 `cfprefsd` 的实现细节，别当成契约，iOS 上和不同系统版本上都可能不同。我能确定并且愿意背书的只有一条：它明确不是同步的，`synchronize` 也改变不了这一点。

Keychain 那一节只覆盖了 macOS 传统钥匙串。数据保护钥匙串（也就是 iOS 上唯一存在的那套）在本次实验中完全没跑起来，`kSecAttrAccessible` 的行为一条都没实测，只有头文件原文。
