---
title: 【iOS】Mach-O：结构、符号绑定与 chained fixups
published: 2026-07-27
description: 老文章讲的 __la_symbol_ptr / __stub_helper / dyld_stub_binder 那套惰性绑定，在今天 Xcode 编出的二进制里一个组件都找不到。切换门槛实测是三条线：真机 iOS 13.4、模拟器 15.0、macOS 12.0。
tags:
  - iOS
  - Mach-O
  - 链接
  - dyld
  - 包体积
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 25
draft: true
---
# Mach-O：结构、符号绑定与 chained fixups

我照着一篇写得很细的文章去找 `__la_symbol_ptr`，`otool -l` 翻了两遍没有。以为命令敲错了，换 `size -m`、换 `nm`、换 `dyld_info`，还是没有。那篇文章讲的一整套惰性绑定——`__stubs` 跳进 `__la_symbol_ptr`，首次调用落到 `__stub_helper`，`dyld_stub_binder` 回填地址——在我这台机器上编出来的二进制里，一个组件都不存在。

不是文章写错了，是链接器换了格式。新格式叫 chained fixups。

切换的门槛我实测了一遍，结论和流传的说法对不上：

```text
ios 12.0  :  LC_DYLD_INFO_ONLY
ios 13.0  :  LC_DYLD_INFO_ONLY
ios 13.1  :  LC_DYLD_INFO_ONLY
ios 13.2  :  LC_DYLD_INFO_ONLY
ios 13.3  :  LC_DYLD_INFO_ONLY
ios 13.4  :  LC_DYLD_CHAINED_FIXUPS  LC_DYLD_EXPORTS_TRIE
ios 15.0  :  LC_DYLD_CHAINED_FIXUPS  LC_DYLD_EXPORTS_TRIE
ios 17.0  :  LC_DYLD_CHAINED_FIXUPS  LC_DYLD_EXPORTS_TRIE
```

真机 target 的分界线在部署目标 iOS 13.4。但这组数字有个前提，我第一次跑的时候没意识到：上面编的是 `arm64-apple-ios`，真机。同一份代码换成 `arm64-apple-ios-simulator`，13.4 到 14.9 全是老格式，一直到 15.0 才切。

**所以「iOS 15」这个流传的数字不是记混，它是模拟器 target 的真实阈值。** 三个平台三条线：真机 13.4、模拟器 15.0、macOS 12.0。你在模拟器上跑起来发现还是 `LC_DYLD_INFO_ONLY`，不是构建配置有问题。这条第五节有完整实验。

系列前面 [[iOS 内存：从虚拟地址空间到堆与栈]] 已经讲过 Segment 怎么被映射成 VM Region、ASLR slide 是什么、符号化为什么要减去 slide。那一篇看的是运行时的地址空间，这一篇往下挖一层，看磁盘上那个文件本身长什么样，以及 dyld 拿到它之后要改哪些字节。

---

## 一、三段式，以及 header 自己也在 `__TEXT` 里

Mach-O 的宏观结构只有三块：header、load commands、data。

一个最小的 Objective-C 程序：

```objc
#import <Foundation/Foundation.h>

int   gInitialized = 42;
int   gZero;
static int sInitialized = 7;
const char *gCString = "hello-cstring";
const int  gConstInt = 99;

@interface Greeter : NSObject
- (void)greet;
@end

@implementation Greeter
- (void)greet {
    NSLog(@"greet %d %d %d %s %d", gInitialized, gZero, sInitialized, gCString, gConstInt);
}
@end

int main(int argc, const char *argv[]) {
    @autoreleasepool {
        Greeter *g = [Greeter new];
        [g greet];
        NSString *s = [NSString stringWithFormat:@"%@", @"literal"];
        printf("%s\n", s.UTF8String);
    }
    return 0;
}
```

```shell
clang -fobjc-arc -framework Foundation -o hello hello.m
otool -h -v hello
```

```text
      magic  cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64    ARM64        ALL  0x00     EXECUTE    21       2504   NOUNDEFS DYLDLINK TWOLEVEL PIE
```

这八个字段就是 `mach_header_64` 的全部，SDK 头文件里的原文：

```c
struct mach_header_64 {
	uint32_t	magic;		/* mach magic number identifier */
	int32_t		cputype;	/* cpu specifier */
	int32_t		cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
	uint32_t	reserved;	/* reserved */
};
```

32 字节。`magic` 是 `0xfeedfacf`，倒着读是 `0xcffaedfe`，用来判断字节序和位宽；`ncmds` / `sizeofcmds` 告诉解析器 load command 区有多少条、共多长，之后就是 data。

`flags` 里那几个词值得单说。`PIE` 表示可以被 ASLR 随意搬家，对应上一篇讲的 slide。`TWOLEVEL` 是二级命名空间，符号除了名字还要记住来自哪个库，`nm -m` 输出里 `(from Foundation)` 这个后缀就是它。`NOUNDEFS` 看着奇怪，这个程序明明用了 `NSLog`，怎么会"没有未定义符号"？后面第五节会看到，chained fixups 把导入符号搬进了自己的表，符号表里的确不再需要它们做重定位。

### `filetype` 的取值

`loader.h` 里这一段是全的：

```c
#define	MH_OBJECT	0x1		/* relocatable object file */
#define	MH_EXECUTE	0x2		/* demand paged executable file */
#define	MH_FVMLIB	0x3		/* fixed VM shared library file */
#define	MH_CORE		0x4		/* core file */
#define	MH_PRELOAD	0x5		/* preloaded executable file */
#define	MH_DYLIB	0x6		/* dynamically bound shared library */
#define	MH_DYLINKER	0x7		/* dynamic link editor */
#define	MH_BUNDLE	0x8		/* dynamically bound bundle file */
#define	MH_DYLIB_STUB	0x9		/* shared library stub for static linking only */
#define	MH_DSYM		0xa		/* companion file with only debug sections */
#define	MH_KEXT_BUNDLE	0xb		/* x86_64 kexts */
#define MH_FILESET	0xc		/* a file composed of other Mach-Os */
#define	MH_GPU_EXECUTE	0xd		/* gpu program */
#define	MH_GPU_DYLIB	0xe		/* gpu support functions */
```

日常能碰到的就四个：`.o` 是 `MH_OBJECT`，App 主二进制是 `MH_EXECUTE`，动态库和 Framework 是 `MH_DYLIB`，dSYM 里那个文件是 `MH_DSYM`。`MH_BUNDLE` 在 macOS 上是 `.bundle` 和某些插件，iOS 上基本见不到。`MH_DYLINKER` 只有 `/usr/lib/dyld` 一个。

`MH_GPU_EXECUTE` / `MH_GPU_DYLIB` 这两个我在 Xcode 26.6 的 `loader.h` 里第一次看见，注释只写了 "gpu program" 和 "gpu support functions"，具体是什么产物我没查证，先不瞎猜。

### header 和 load commands 住在 `__TEXT` 里面

这一点我一开始理解错了，以为 header 是独立于所有 Segment 的一段前缀。

```text
segname __TEXT
 vmaddr 0x0000000100000000
fileoff 0
filesize 16384
```

`__TEXT` 的 `fileoff` 是 0。header（32 字节）和 load commands（2504 字节）都落在 `__TEXT` 的映射范围内，第一条指令 `__text` 的文件偏移是 2584，正好在 2536 之后加了点对齐。

所以 `__mh_execute_header` 这个符号的地址就是 `__TEXT` 的起始地址，`nm` 能看到它：

```text
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
```

这个符号很有用。想在运行时自己遍历 load command，起点就是它：拿到它的地址，往后跳 32 字节就是第一条 `load_command`，按 `cmdsize` 一条条走 `ncmds` 次。很多 hook 框架、反调试检测、以及"扫一遍 `__objc_classlist` 拿到本镜像所有类"的代码，都是这么起手的。

---

## 二、同一份代码的四种 filetype

把同一个 `hello.m` 编成四种产物，load command 的差异比想象中大。

| | `.o` (`MH_OBJECT`) | 可执行 (`MH_EXECUTE`) | dylib (`MH_DYLIB`) | dSYM (`MH_DSYM`) |
| --- | --- | --- | --- | --- |
| load command 数 | 4 | 21 | 19 | 9 |
| `LC_SEGMENT_64` | 1（无名） | 5 | 4 | 6 |
| `__PAGEZERO` | 无 | 有 | 无 | 有 |
| 入口 | 无 | `LC_MAIN` | `LC_ID_DYLIB` | 无 |
| 依赖 | 无 `LC_LOAD_DYLIB` | 4 条 | 4 条 | 无 |
| 绑定信息 | 无 | `LC_DYLD_CHAINED_FIXUPS` | `LC_DYLD_CHAINED_FIXUPS` | 无 |
| 签名 | 无 | `LC_CODE_SIGNATURE` | `LC_CODE_SIGNATURE` | 无 |

`.o` 只有四条：`LC_SEGMENT_64`、`LC_BUILD_VERSION`、`LC_SYMTAB`、`LC_DYSYMTAB`。它还没被"装载"过，所以谈不上入口、依赖、绑定，也没有签名。

`.o` 那唯一一个 Segment 是没有名字的，所有 section 挤在里面：

```text
$ size -m hello.o
Segment : 904
	Section (__TEXT, __text): 352
	Section (__DATA, __data): 20
	Section (__TEXT, __cstring): 50
	Section (__DATA, __cfstring): 96
	Section (__DATA, __objc_const): 176
	Section (__DATA, __objc_data): 80
	Section (__DATA, __objc_classrefs): 16
	Section (__LD, __compact_unwind): 64
	...
```

section 自己记着"我想去 `__TEXT`"或者"我想去 `__DATA`"，但目标 Segment 还不存在。链接器读完所有 `.o`，把同名 section 合并，才真正建起 `__TEXT` / `__DATA_CONST` / `__DATA`。`__LD,__compact_unwind` 是给链接器看的中间产物，最后会被翻译成 `__TEXT,__unwind_info`，可执行文件里就没有 `__LD` 了。

还有一处只在 `.o` 里出现：

```text
0000000000000004 (common) (alignment 2^2) external _gZero
```

未初始化的全局变量 `gZero` 在 `.o` 里是 common 符号，不属于任何 section，只记了大小和对齐。到可执行文件里它才落进 `__DATA,__common`。

dylib 少了 `__PAGEZERO`，因为它要被装载到主程序旁边，不该独占低 4 GB。它用 `LC_ID_DYLIB` 代替 `LC_MAIN`，记录自己的安装路径：

```text
cmd LC_ID_DYLIB
name @rpath/libgreet.dylib (offset 24)
```

dSYM 最有意思。它有和主二进制一模一样的 Segment / Section 骨架，连地址和大小都对得上，但：

```text
segname __TEXT
 vmaddr 0x0000000100000000
 vmsize 0x0000000000004000
fileoff 0
filesize 0        ← 零
```

`filesize` 是 0，每个 section 的 `offset` 也是 0。骨架留着是为了地址查询，真实内容一个字节都没带，多出来的是一个 `__DWARF` Segment：

```text
segname __DWARF
  sectname __debug_line
  sectname __debug_aranges
  sectname __debug_addr
  sectname __debug_info
  sectname __debug_abbrev
  sectname __debug_str
  sectname __debug_str_offs
  sectname __debug_line_str
  sectname __debug_names
```

dSYM 和二进制靠 `LC_UUID` 配对，两边必须一致：

```shell
$ otool -l hello_g | grep -A2 LC_UUID | grep uuid
    uuid A91BB752-05CF-3FCB-A571-D3B9BE0E6CC4
$ dwarfdump --uuid hello_g.dSYM
UUID: A91BB752-05CF-3FCB-A571-D3B9BE0E6CC4 (arm64) ...
```

上一篇讲符号化时说"用链接地址去 dSYM 里查函数名和行号"，查的就是 `__DWARF,__debug_info` 和 `__debug_line`。找对哪一份 dSYM，靠的就是这个 UUID。崩溃日志 Binary Images 段里每个镜像后面跟的那串十六进制，也是它。

---

## 三、Segment 与 Section：一层管权限，一层管内容

`otool -l hello` 完整输出太长，把 Segment 骨架抽出来是这样：

| Segment | vmaddr | vmsize | fileoff | filesize | initprot | flags |
| --- | --- | --- | --- | --- | --- | --- |
| `__PAGEZERO` | 0x0 | 0x100000000 | 0 | 0 | `---` | 0 |
| `__TEXT` | 0x100000000 | 0x4000 | 0 | 16384 | `r-x` | 0 |
| `__DATA_CONST` | 0x100004000 | 0x4000 | 16384 | 16384 | `rw-` | 0x10 |
| `__DATA` | 0x100008000 | 0x4000 | 32768 | 16384 | `rw-` | 0 |
| `__LINKEDIT` | 0x10000c000 | 0x4000 | 49152 | 2360 | `r--` | 0 |

Segment 是内核 `mmap` 的单位，所以权限定在这一层，而且必须页对齐。Section 只是 Segment 内部的分区，用来区分内容，不带权限。上一篇的表已经列了各 Section 装什么，这里只补三件当时没展开的。

`__PAGEZERO` 的 `vmsize` 是 0x100000000，整整 4 GB，`filesize` 是 0。 它不占文件、不占物理内存，只是在虚拟地址空间最低端画了一块无任何权限的区域。空指针解引用、以及任何把 32 位整数误当指针用的访问，都会落在这里当场失败。

`__DATA_CONST` 的 `initprot` 是 `rw-`，不是只读的。 我以前想当然以为它就是 `r--`。真正让它只读的是那个 `flags 0x10`：

```c
#define SG_READ_ONLY    0x10 /* This segment is made read-only after fixups */
```

"after fixups"。内核按 `rw-` 映射，dyld 把这一段里所有需要改的指针改完，再自己 `mprotect` 成只读。本地那篇 [[dyld]] 里抄的 dyld4 源码正好有这段：

```cpp
// make __DATA_CONST read-only (kernel maps it r/w)
dyldMA->forEachSegment(^(const MachOAnalyzer::SegmentInfo& segInfo, bool& stop) {
    if ( segInfo.readOnlyData ) {
        sSyscallDelegate.mprotect((void*)start, size, PROT_READ);
    }
});
```

所以 `vmmap` 里看到 `__DATA_CONST` 是 `r--`，和 Mach-O 里写的 `rw-` 都没错，是两个时刻。`__got`、`__cfstring`、`__objc_classlist` 这些"启动后就不该再变"的东西放这里，改一次，然后锁死。ROP 攻击少了一批可写的函数指针。

`__DATA` 里的 `__common` 是 zerofill。 `size -m` 会标出来：

```text
Section __common: 4 (zerofill)
```

它的 section `offset` 字段是 0，文件里不占一个字节，装载时由内核给零填充页。这就是"未初始化全局变量不增加包体积"的机制。

### 一个 Segment 的所有 Section 加起来，远小于 Segment

```text
Segment __TEXT: 16384
	Section __text: 352
	Section __stubs: 96
	Section __objc_stubs: 96
	Section __objc_methlist: 20
	Section __cstring: 50
	Section __const: 4
	Section __objc_classname: 8
	Section __objc_methtype: 8
	Section __objc_methname: 35
	Section __unwind_info: 88
	total 757
```

757 字节的内容，占了 16384 字节的 Segment。因为 Segment 必须按页对齐，而这台机器上 `vm_page_size` 是 16 KB。小程序里对齐浪费能占到 95%，真实 App 里这个比例可以忽略，但看 `size -m` 输出时要分清 Segment 行和 section total 行。

---

## 四、`__LINKEDIT`：一个没有 Section 的 Segment

`__LINKEDIT` 的 `nsects` 是 0。它里面所有东西都不通过 section 描述，而是由各条 load command 用 `(offset, size)` 直接指进去。

把 `hello` 的所有相关 load command 里的偏移抄出来排一排：

```text
chained fixups            49152 ..  49616  size    464
exports trie              49616 ..  49792  size    176
function starts           49792 ..  49800  size      8
data in code              49800 ..  49800  size      0
symtab (30 × 16)          49800 ..  50280  size    480
indirect syms (18 × 4)    50280 ..  50352  size     72
string table              50352 ..  50968  size    616
(pad)                     50968 ..  50976  size      8
code signature            50976 ..  51512  size    536
```

`__LINKEDIT` 的 `fileoff` 是 49152，`filesize` 是 2360，49152 + 2360 = 51512。文件的实际大小：

```shell
$ ls -l hello
-rwxr-xr-x  1 tommywu  wheel  51512 Jul 27 02:44 hello
```

一个字节不多不少，九块内容首尾相接铺满整个 `__LINKEDIT`。这张表是我做这篇实验里最喜欢的一个结果，因为它把"`__LINKEDIT` 里到底有什么"这个问题变成了一道加法题。

九块的分工：

- chained fixups：dyld 要改哪些地址、改成什么，第五节详解。
- exports trie：本镜像导出了哪些符号，前缀树压缩。可执行文件里通常很小，dylib 里是重头。
- function starts：所有函数入口地址的差值序列，ULEB128 编码。崩溃日志里没有符号表也能算出"这个地址落在哪个函数内部"，靠的就是它。
- data in code：`__TEXT` 里混进来的非指令数据（跳转表之类），反汇编器需要知道跳过哪些字节。arm64 上通常是 0。
- symbol table：`nlist_64` 数组，每项 16 字节，只存符号的类型、所在 section、地址，名字是 string table 里的偏移。
- indirect symbol table：`__stubs` 和 `__got` 每一个槽位对应哪个符号，一项 4 字节。
- string table：所有符号名字，`\0` 分隔。
- code signature：必须在最后。

### 为什么在文件末尾

三个原因叠在一起。

`__LINKEDIT` 是唯一一个"长度取决于符号数量而不是代码大小"的 Segment，放在中间的话，改一个符号名就要挪动后面所有 Segment 的文件偏移。

它的内容都是变长编码，链接器只有在其他所有部分都定稿之后才能生成。

代码签名要覆盖它前面的所有字节，自己当然只能排最后。签名的粒度是 4096 字节一页：

```shell
$ codesign -dvvv hello
CodeDirectory v=20400 size=510 flags=0x20002(adhoc,linker-signed) hashes=13+0
Hash type=sha256 size=32
```

13 个哈希。签名数据从 50976 开始，`ceil(50976 / 4096) = 13`，对上了。注意这里的 4096 和上一节的 16384 是两个东西，代码签名用 4 KB 页，虚拟内存用 16 KB 页。

改一个字节试试：

```shell
$ cp hello hello_tampered
$ printf '\x00' | dd of=hello_tampered bs=1 seek=2600 count=1 conv=notrunc
$ ./hello_tampered
$ echo $?
137
```

137 = 128 + 9，SIGKILL。那个偏移落在 `__TEXT,__text` 里，第一页的哈希对不上，进程被内核直接杀掉，连 `main` 都没进。Apple Silicon 上所有可执行代码强制签名，改一个字节就是这个下场。

`__LINKEDIT` 的另一个特点是它运行时会被映射进来（`initprot` 是 `r--`），dyld 全程要读它。上一篇里那句"`__LINKEDIT` 不属于五大分区里的业务数据"是对的，但它确实占着虚拟地址。

---

## 五、符号绑定：老那套和新那套

终于到本文最想说的一节。

### 老那套：`LC_DYLD_INFO_ONLY`

想看它还在的样子，得手动关掉新格式：

```shell
clang -fobjc-arc -framework Foundation -Wl,-no_fixup_chains -o hello_nofc hello.m
```

`LC_DYLD_CHAINED_FIXUPS` 和 `LC_DYLD_EXPORTS_TRIE` 消失，`LC_DYLD_INFO_ONLY` 回来了：

```text
cmd LC_DYLD_INFO_ONLY
 rebase_off 49152     rebase_size 32
   bind_off 49184       bind_size 200
lazy_bind_off 49384  lazy_bind_size 208
 export_off 49592     export_size 176
```

四段独立的 opcode 流。rebase 处理"本镜像内部的指针要加上 slide"，bind 处理"这个槽位要填别的库里某个符号的地址"，lazy_bind 是延后到第一次调用才做的 bind。这些 opcode 长这样：

```text
rebase opcodes:
  0x0000 REBASE_OPCODE_SET_TYPE_IMM(1)
  0x0001 REBASE_OPCODE_SET_SEGMENT_AND_OFFSET_ULEB(2, 0x00000028)
  0x0003 REBASE_OPCODE_DO_REBASE_ADD_ADDR_ULEB(32)

lazy bind opcodes:
  0x0000 BIND_OPCODE_SET_SEGMENT_AND_OFFSET_ULEB(0x03, 0x00000000)
  0x0002 BIND_OPCODE_SET_DYLIB_ORDINAL_IMM(1)
  0x0003 BIND_OPCODE_SET_SYMBOL_TRAILING_FLAGS_IMM(0x00, _NSLog)
  0x000B BIND_OPCODE_DO_BIND()
  0x000C BIND_OPCODE_DONE()
```

一台小型虚拟机，dyld 逐条解释执行。

配合它的是那套熟悉的三级跳。`hello_nofc` 里同时存在 `__TEXT,__stubs`、`__TEXT,__stub_helper`、`__DATA,__la_symbol_ptr`：

```text
Contents of (__TEXT,__stubs) section
0000000100000c38	adrp	x16, 8 ; 0x100008000     ← __la_symbol_ptr 的地址
0000000100000c3c	ldr	x16, [x16]
0000000100000c40	br	x16

Contents of (__DATA,__la_symbol_ptr) section
0000000100008000	00000cb0 00000001 ...              ← 初值 0x100000cb0

Contents of (__TEXT,__stub_helper) section
0000000100000c98	adrp	x17, 8
0000000100000c9c	add	x17, x17, #0x150
0000000100000ca0	stp	x16, x17, [sp, #-0x10]!
0000000100000ca4	adrp	x16, 4
0000000100000ca8	ldr	x16, [x16, #0x10]           ← __got 里的 dyld_stub_binder
0000000100000cac	br	x16
0000000100000cb0	ldr	w16, 0x100000cb8            ← 本符号在 lazy_bind 流里的偏移
0000000100000cb4	b	0x100000c98
```

`__la_symbol_ptr` 的初值 `0x100000cb0` 正落在 `__stub_helper` 里。第一次调用 `NSLog`，跳进 stub，读 `__la_symbol_ptr`，跳到 `__stub_helper` 的对应条目，压入一个偏移，转到公共入口，最后调 `dyld_stub_binder`。它解析完符号，把真地址回填进 `__la_symbol_ptr`，之后每次调用就只剩两条指令。

`nm` 也能佐证：

```shell
$ nm -m hello_nofc | grep stub_binder
                 (undefined) external dyld_stub_binder (from libSystem)
```

### 新那套：`LC_DYLD_CHAINED_FIXUPS`

默认参数编出来的 `hello`，上面这三样东西一样都没有。

```text
$ otool -l hello | grep sectname
 sectname __text
 sectname __stubs
 sectname __objc_stubs
 ...
 sectname __got
 ...
$ nm -m hello | grep stub_binder
（无输出）
```

没有 `__stub_helper`，没有 `__la_symbol_ptr`，没有 `dyld_stub_binder`。stub 直接穿 `__got`：

```text
Contents of (__TEXT,__stubs) section
0000000100000b78	adrp	x16, 4 ; 0x100004000     ← __DATA_CONST,__got
0000000100000b7c	ldr	x16, [x16]
0000000100000b80	br	x16
```

**惰性绑定在 chained fixups 下不存在了，所有导入符号在启动时一次绑完。** 这是我这一节最想让人记住的一句。老文章里"第一次调用才解析符号"的图，画的是一个今天已经不存在的机制。

新格式为什么能省事，看它的数据布局就懂了。`LC_DYLD_CHAINED_FIXUPS` 指向的那 464 字节里是这么组织的：

```text
dyld_chained_fixups_header:
    fixups_version  0x00000000
    starts_offset   0x00000020
    imports_offset  0x00000068
    symbols_offset  0x000000A0
    imports_count   0x0000000E     ← 14 个导入符号
    imports_format  0x00000001
dyld_chained_starts_in_segment:
    page_size           0x00004000
    pointer_format      0x00000006  ← DYLD_CHAINED_PTR_64_OFFSET
    segment_offset      0x00004000  ← __DATA_CONST
    page_count          0x00000001
       start[ 0]:  0x0000          ← 本页第一个 fixup 的页内偏移
dyld_chained_starts_in_segment:
    ...
    segment_offset      0x00008000  ← __DATA
       start[ 0]:  0x0018
```

它只记了"每一页的第一个 fixup 在哪"。剩下的位置信息藏在被修改的指针自己身上。

原始字节：

```text
  0x00004000:  raw: 0x8010000000000000    bind: (next: 002, bindOrdinal: 0x000000)
  0x00004008:  raw: 0x8010000000000001    bind: (next: 002, bindOrdinal: 0x000001)
  ...
  0x00004050:  raw: 0x802000000000000A    bind: (next: 004, bindOrdinal: 0x00000A)
  0x00004060:  raw: 0x0020000000000C62  rebase: (next: 004, target: 0x00000000C62)
```

对着头文件的位域定义拆一遍：

```c
struct dyld_chained_ptr_64_rebase {
    uint64_t    target    : 36,   // 64GB max image size
                high8     :  8,
                reserved  :  7,
                next      : 12,   // 4-byte stride
                bind      :  1;   // == 0
};
struct dyld_chained_ptr_64_bind {
    uint64_t    ordinal   : 24,
                addend    :  8,
                reserved  : 19,
                next      : 12,   // 4-byte stride
                bind      :  1;   // == 1
};
```

`0x8010000000000000`：bit63 = 1 表示 bind；`next` 字段（bit 51-62）值是 2，步长 4 字节，所以下一个 fixup 在 8 字节之后；低 24 位是导入符号序号 0，查 imports 表得到 `_NSLog`。

`0x0020000000000C62`：bit63 = 0 表示 rebase；`next` = 4，下一个在 16 字节之后；低 36 位 `0xC62` 是目标在本镜像内的偏移，dyld 加上 slide 就是最终地址。

一条链就这么串下去，`next` 为 0 表示本页结束。整个 `__DATA_CONST` + `__DATA` 里所有需要改的指针，被穿成两条链，元数据只有"每页起点"这么点。dyld 不再解释 opcode 流，只是顺着链走，读一个、算一个、写回去。

这也解释了 header 里那个反直觉的 `NOUNDEFS`：导入符号的名字存在 chained fixups 自己的 symbols 区，不再依赖符号表做重定位。

### 实测的切换阈值

开头那组数据的完整命令：

```shell
SDK=$(xcrun --sdk iphoneos --show-sdk-path)
for v in 12.0 13.0 13.1 13.2 13.3 13.4 15.0 17.0; do
  clang -fobjc-arc -isysroot "$SDK" -target arm64-apple-ios$v \
        -framework Foundation -o ios_$v hello.m
  otool -l ios_$v | grep -E "LC_DYLD_CHAINED_FIXUPS|LC_DYLD_INFO_ONLY"
done
```

分界线卡在 13.3 和 13.4 之间。macOS 侧同样测法，11.0 还是老格式，12.0 起是新格式。

到这里我本来准备收工，写下"iOS 15 那个说法是记混了"。写之前顺手把 `-isysroot` 换成模拟器 SDK、target 后面加个 `-simulator` 重跑了一遍，结果打脸：

```text
ios 13.4-simulator :  LC_DYLD_INFO_ONLY
ios 14.0-simulator :  LC_DYLD_INFO_ONLY
ios 14.5-simulator :  LC_DYLD_INFO_ONLY
ios 14.9-simulator :  LC_DYLD_INFO_ONLY
ios 15.0-simulator :  LC_DYLD_CHAINED_FIXUPS
```

模拟器的阈值是 15.0。iOS 15 这个数字在网上流传得那么广，很可能就是因为大多数人是在模拟器上验的。三条线并排放：

| 平台 | 切到 chained fixups 的部署目标 |
| --- | --- |
| iOS 真机（`arm64-apple-ios`） | 13.4 |
| iOS 模拟器（`-simulator`） | 15.0 |
| macOS（`arm64-apple-macos`） | 12.0 |

决定用哪套的是部署目标（deployment target）加平台这一对，不是 Xcode 版本，也不是运行的系统版本。 这就是为什么老项目升级了 Xcode，二进制里还是 `LC_DYLD_INFO_ONLY`。我在本机的 `/Applications` 里随手扫了四个第三方 App，两个新格式两个老格式，这两种东西今天是真的共存在同一台机器上。

### arm64e 上多一层：指针认证

同样的代码换成 `arm64e`：

```text
pointer_format:  12 (DYLD_CHAINED_PTR_ARM64E_USERLAND24)(authenticated arm64e, 8-byte stride, target vmoffset)

0x00008000:  raw: 0xC009000000000000  auth-bind: (next: 001, key: IA, addrDiv: 1, diversity: 0x0000, bindOrdinal: 0x000000)
0x00008048:  raw: 0xC0156AE100000009  auth-bind: (next: 002, key: DA, addrDiv: 1, diversity: 0x6AE1, bindOrdinal: 0x000009)
```

格式号从 6 变成 12，步长从 4 字节变成 8 字节，每个条目多带了签名密钥（IA 用于代码指针，DA 用于数据指针）和 diversity。section 名字也变了，`__got` 变成 `__auth_got`：

```text
__DATA_CONST  __auth_got  0x100008000  auth-bind  Foundation/_NSLog (div=0x0000 ad=1 key=IA)
```

dyld 写进去的不是裸地址，是带 PAC 签名的指针。真机上 A12 起就是这套。同一件事在 [[iOS 内存管理：从 MRC、ARC 到属性关键字#引用计数存在哪里|MRC 的所有权规则]] 里也出现过：指针认证占掉 isa 的位，把 `extra_rc` 从 19 位压到 8 位。arm64e 这个架构对底层细节的影响，比大多数人以为的深。

### 另一个眼生的 section：`__objc_stubs`

`nm` 输出里有三个眼生的符号：

```text
0000000100000be0 (__TEXT,__objc_stubs) non-external _objc_msgSend$UTF8String
0000000100000c00 (__TEXT,__objc_stubs) non-external _objc_msgSend$greet
0000000100000c20 (__TEXT,__objc_stubs) non-external _objc_msgSend$stringWithFormat:
```

每个被调用的 selector 生成一段小 stub，调用点直接 `bl _objc_msgSend$greet`，不用在调用处加载 selref 再传参。它让 `__objc_selrefs` 的引用集中到这些 stub 里，代码体积换调用点体积。我把部署目标从 iOS 13.0 一路试到 17.0，`__objc_stubs` 都在，说明它和部署目标无关，是编译器的默认行为。看反汇编时别把它当成什么新的消息发送机制，最后还是 `objc_msgSend`。

### 一条我没能在本机验证的

Apple 在 WWDC22 讲过 page-in linking：iOS 16 / macOS 13 起，chained fixups 可以由内核在页面缺页调入时才应用，而不是启动时一次性做完。这样启动时不用把整个 `__DATA` 走一遍，而且被修改的页可以保持 clean，不计入 dirty memory。

> 待核实：page-in linking 在本机没有直接的观察手段（需要看内核的 `vm_map` 行为或 dyld 的内部计数）。上面这段来自 WWDC22 Session 110362 的说法，我没有独立验证过。链式格式本身让这件事成为可能，因为 fixup 的位置信息是按页组织的，这一点从 `dyld_chained_starts_in_segment` 的 `page_start[]` 数组能直接看出来。

---

## 六、通用二进制

```shell
clang -fobjc-arc -target x86_64-apple-macos13 -framework Foundation -o hello_x86 hello.m
clang -fobjc-arc -target arm64-apple-macos13  -framework Foundation -o hello_arm hello.m
lipo -create hello_x86 hello_arm -output hello_fat
```

```text
$ lipo -detailed_info hello_fat
fat_magic 0xcafebabe
nfat_arch 2
architecture x86_64
    offset 4096      size 13984    align 2^12 (4096)
architecture arm64
    offset 32768     size 51512    align 2^14 (16384)
```

fat 文件不是一种 Mach-O，它是一个容器：开头一个大端的 `fat_header`（`0xcafebabe` + 架构数），后面每个架构一条 `fat_arch` 记录偏移、大小、对齐，再往后就是若干个完整的、独立的 Mach-O。

```text
$ xxd -l 48 hello_fat
00000000: cafe babe 0000 0002 0100 0007 0000 0003
00000010: 0000 1000 0000 36a0 0000 000c 0100 000c
00000020: 0000 0000 0000 8000 0000 c938 0000 000e
```

大端是历史包袱，Mach-O 从 PowerPC 时代继承下来的，主机字节序变了但 fat header 没改。

算一下大小：

```text
13984 + 51512 = 65496
实际 fat 文件    84280
```

多出 18784 字节全是对齐填充。x86_64 slice 结束在 18080，而 arm64 slice 必须从 16 KB 边界开始，所以要垫到 32768。通用二进制的体积不是各架构之和，还要加上对齐空洞。

日常用得上的三条：

```shell
lipo -info App.framework/App          # 有哪些架构
lipo -thin arm64 in -output out       # 抽一个架构出来
lipo -remove x86_64 in -output out    # 去掉一个架构
```

App Store 分发的时候 Apple 会做 app thinning，用户只下到自己设备那一个 slice，所以主二进制里带着模拟器架构，影响的是本地包和 CI 传输，不是最终下载体积。但 `Frameworks/` 里手动塞进去的胖二进制，如果没走 thinning（比如通过企业签名分发），就实打实地留在包里。

---

## 七、包体积：`size` 和 `strip` 能看出什么

`size -m` 是分析包体积最快的入口，因为它把 Segment 和 Section 两层一起给出来了。拿抖音里一个真实的 ObjC 框架看：

```shell
$ lipo -thin arm64 ByteDanceKit -output bdk_arm64
$ size -m bdk_arm64
Segment __TEXT: 196608          （只摘大项）
	Section __text: 142944
	Section __objc_methlist: 7664
	Section __objc_methname: 16312
	Section __objc_methtype: 2022
	Section __unwind_info: 4836
	Section __cstring: 1772
Segment __DATA_CONST: 16384
Segment __DATA: 16384
Segment __LINKEDIT: 245760
```

`__LINKEDIT` 比 `__TEXT` 还大（这里 `size -m` 给的是页对齐后的 vmsize，`__LINKEDIT` 实际 filesize 是 244912）。拆开看：

```text
LC_SYMTAB   nsyms 5544     → 5544 × 16 = 88704 字节
            strsize 128024
```

符号表加字符串表 216728 字节，占 `__LINKEDIT` 的 88%，占整个 arm64 slice（474288 字节）的 46%。

这个框架用的还是老格式（`LC_DYLD_INFO_ONLY`），四段 opcode 加起来只有 6464 字节。所以撑大 `__LINKEDIT` 的从来不是绑定信息，是符号名。

```shell
$ strip -x bdk_arm64 -o bdk_stripped
$ ls -l bdk_arm64 bdk_stripped
474288  bdk_arm64
265712  bdk_stripped
```

**`strip -x` 把这一个框架从 474 KB 压到 266 KB，砍掉 44%。** `nsyms` 从 5544 降到 209，`strsize` 从 128024 降到 4816，`__LINKEDIT` 从 244912 降到 36336。

Xcode 里对应的是 `Strip Linked Product` / `Strip Style` / `Deployment Postprocessing`。前提是 dSYM 已经产出并归档，否则线上崩溃就没法符号化了。

### ObjC 元数据到底多贵

想知道一个方法值多少字节，最直接的办法是编两个只差 100 个方法的程序。

```objc
- (int)someReasonablyLongMethodName0000:(int)a with:(int)b { return a+b+0; }
// ... 共 100 个
```

| | 0 个方法 | 100 个方法 | 差值 |
| --- | --- | --- | --- |
| 文件大小 | 50456 | 56824 | +6368 |
| `__text` | 68 | 4468 | +4400 |
| `__objc_methlist` | 0 | 1208 | +1208 |
| `__objc_methname` | 0 | 3900 | +3900 |
| `__objc_methtype` | 0 | 14 | +14 |

两个数字能被精确解释。

`__objc_methlist` 的 1208 = 8 + 100 × 12。头 8 字节是 entsize 和 count，实际内容：

```text
0000000100001958	8000000c 00000064 ...
```

`0x8000000c`：低位 12 是每项大小，最高位 `0x80000000` 是 relative method list 标志。每项 12 字节，三个 int32 相对偏移（name / types / imp）。

这个格式本身也有部署目标门槛，我编了两份对照：

| 部署目标 | `__objc_methlist` | `__objc_const` | 方法列表合计 |
| --- | --- | --- | --- |
| iOS 13.0 | 无此 section | 2552 | 2408 |
| iOS 14.0 | 1208 | 144 | 1208 |

iOS 13.0 上没有 `__objc_methlist`，方法列表待在 `__DATA,__objc_const` 里，2552 − 144 = 2408 = 8 + 100 × 24，每项三个 8 字节绝对指针。iOS 14.0 起换成相对偏移，每项 12 字节，正好省一半。而且这些字节从 `__DATA` 挪进了 `__TEXT`：绝对指针要 rebase，相对偏移不用，所以它既省体积又省一批启动时的 fixup。

`__objc_methname` 的 3900 = 100 × 39。`someReasonablyLongMethodName0000:with:` 长 38 字符，加一个 `\0` 是 39。selector 的名字，在二进制里一个字节不少地存着。

所以一个 ObjC 方法的固定成本大约是：12 字节方法列表项 + selector 长度 + 1，再加上方法体本身的机器码。类型编码字符串是共享的，100 个签名相同的方法只花了 14 字节。

这给包体积优化定了个量级：删掉一个几乎不用的 category，省的是它所有方法名的字符串长度加方法体；而把 selector 起短一点这种"优化"，一百个方法也就省几 KB，不值得牺牲可读性。

### `__DATA_CONST` 的另一半意义

前面说 `__DATA_CONST` 是"fixup 完就锁死"。它对内存也有影响：这些页被 dyld 改过，属于 dirty memory，一旦有了 page-in linking 就可能变回 clean。老二进制里所有这些数据都在 `__DATA`，运行时全是 dirty。

ByteDanceKit 那个 fat 文件正好给了对照。它的 x86_64 slice 压根没有 `__DATA_CONST` 这个 Segment，`__objc_classlist` / `__objc_catlist` / `__objc_const` 全挤在 `__DATA` 里；arm64 slice 有 `__DATA_CONST`，`__objc_classlist` 和 `__objc_catlist` 被挪了过去，`__objc_const` 留在原地。

我一开始以为这是架构差异，试了一下不是：自己编一个 `x86_64-apple-macos13` 的可执行文件，`__DATA_CONST` 好好地在。回头看这两个 slice 的版本命令，答案就清楚了：

```text
x86_64 slice:  LC_VERSION_MIN_MACOSX   version 10.10
arm64  slice:  LC_BUILD_VERSION        minos 11.0
```

同一份源码、同一次构建，两个 slice 的部署目标不一样，链接器就给了不同的分区。这一节和第五节的 iOS 13.4 是同一个道理：影响 Mach-O 长相的主要开关是部署目标。

这一段和上一篇 [[iOS 内存：从虚拟地址空间到堆与栈]] 里的 clean / dirty 讨论是同一件事的两头：Mach-O 决定了哪些页有机会保持 clean。

---

## 八、几个已经不准的说法

- "惰性绑定：第一次调用才通过 `dyld_stub_binder` 解析符号。" 部署目标 iOS 13.4 / macOS 12.0 以上，`__la_symbol_ptr`、`__stub_helper`、`dyld_stub_binder` 三样都不再生成，所有导入符号启动时一次绑完。
- "chained fixups 是 iOS 15 引入的。" 半对。真机 target 部署目标 13.4 就切了，模拟器确实要到 15.0。说这话的人多半是在模拟器上验的，而读的人拿去套真机。
- "`__DATA_CONST` 是只读段。" 它的 `initprot` 是 `rw-`，靠 `SG_READ_ONLY`（0x10）标志让 dyld 在 fixup 完成后 `mprotect` 成只读。
- "Mach-O = header + load commands + segments，header 独立于所有段。" header 和 load commands 本身就落在 `__TEXT` 的映射范围内，`__TEXT` 的 `fileoff` 是 0。
- "`__LINKEDIT` 是符号表。" 它至少装九类东西，符号表只是其中之一。在一个未 strip 的大框架里，符号表 + 字符串表能占到 `__LINKEDIT` 的 88%，但换成一个小可执行文件，chained fixups 和代码签名的占比反而更大。
- "通用二进制的大小等于各架构之和。" 每个 slice 要按各自架构的页大小对齐，中间的空洞可以很大（本文的例子是 18784 字节）。
- "`.o` 里已经有 `__TEXT` 段了。" `.o` 只有一个无名 Segment，section 记着自己想去哪个 Segment，真正的 Segment 由链接器建立。
- "`nm` 看到 `NOUNDEFS` 说明没链接外部符号。" 在 chained fixups 二进制里，导入符号存在 fixups 自己的表里，符号表确实"没有未定义项"。

---

## 总结

Mach-O 的结构可以浓缩成一句话：header 说这是什么，load commands 说怎么装，剩下的是内容。真正需要动手才能建立直觉的是两处。

一处是 `__LINKEDIT`。它没有 Section，全靠各条 load command 用偏移指进去，九块内容首尾相接铺满整个 Segment，最后一块必须是代码签名。把 `otool -l` 里的偏移抄下来做一次加法，能加到文件大小一字节不差，这比读十遍"`__LINKEDIT` 存放链接信息"这句话有用。

另一处是符号绑定。老的 `LC_DYLD_INFO_ONLY` 是四段 opcode 流加一套三级跳的惰性绑定；新的 chained fixups 把要修改的指针自己串成链，只在元数据里记每页的起点，dyld 顺着链走一遍就完事，惰性绑定整个消失。切换的门槛是部署目标加平台：真机 13.4、模拟器 15.0、macOS 12.0。两种格式今天在同一台机器上共存。

包体积方面，最容易被忽略的是符号表。一个真实框架 `strip -x` 之后小了 44%，全来自 `__LINKEDIT`。ObjC 侧的固定成本是每个方法 12 字节的相对方法列表项加上 selector 名字的完整长度，实测精确到字节。

最后是方法论，和 [[iOS 内存管理：从 MRC、ARC 到属性关键字#引用计数存在哪里|MRC 的所有权规则]] 里那条一样：这一篇里所有"网上说的不对"的结论，没有一条是靠读更多文章得到的。开头那个阈值，是一个 for 循环编八次的事；`__LINKEDIT` 的九块内容，是把偏移抄进 Python 加一遍的事。遇到格式、阈值、体积这类问题，`otool -l` 打一遍比读十篇文章可靠。

但也别把这句话当成免死金牌。我编八次拿到 13.4，差点就写下"网上说 iOS 15 的都是抄的"，而 iOS 15 在模拟器上是对的——我只是恰好没往那个方向编。**跑了实验只能保证你测的那个组合是对的，保证不了你测的组合覆盖了读者会遇到的组合。** 下判断之前先问一句：还有哪个维度我一直没动过。

下一篇讲这些字节被 dyld 拿到之后发生了什么：[[dyld]]。编译器和链接器怎么把源码变成上面这些 section，见 [[iOS 从源码到可执行文件：四个阶段与符号]]。

## 参考资料

### 官方

- [Apple — Overview of the Mach-O Executable Format](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html)：概念层面的权威说明，但成文很早，chained fixups 之类的内容没有
- SDK 头文件 `mach-o/loader.h`：`mach_header_64`、`segment_command_64`、所有 `MH_*` 和 `SG_*` 常量的最终裁判
- SDK 头文件 `mach-o/fixup-chains.h`：chained fixups 的全部结构体和 `DYLD_CHAINED_PTR_*` 枚举，本文第五节的位域拆解全部依据这个文件
- SDK 头文件 `mach-o/fat.h`、`mach-o/nlist.h`
- [WWDC22 — Link fast: Improve build and launch times](https://developer.apple.com/videos/play/wwdc2022/110362/)：chained fixups 与 page-in linking 的官方介绍
- [WWDC21 — Symbolication: Beyond the basics](https://developer.apple.com/videos/play/wwdc2021/10211/)：UUID、dSYM 与地址还原
- [apple-oss-distributions/dyld](https://github.com/apple-oss-distributions/dyld)：`MachOLoaded`、`MachOAnalyzer` 里是 chained fixups 的实际消费方

### 工具

- `otool -h / -l / -s / -Iv / -L`：最基础，输出啰嗦但完整
- `dyld_info`：Xcode 自带，比 otool 好用得多。`-fixups`、`-fixup_chain_details`、`-fixup_chain_header`、`-opcodes`、`-exports`、`-objc` 这几个开关本文全用到了。它还能直接读 dyld shared cache 里的库，而 otool 读不了（系统 dylib 在磁盘上已经没有独立文件）
- `size -m`、`nm -m`、`lipo`、`strip`、`vtool`、`dwarfdump`、`codesign -dvvv`

### 本地

- [[iOS 内存：从虚拟地址空间到堆与栈]]：Segment 如何映射成 VM Region、ASLR slide、符号化
- [[dyld]]：装载与初始化流程，`__DATA_CONST` 的 `mprotect` 源码出自这里
- [[iOS 内存管理：从 MRC、ARC 到属性关键字#引用计数存在哪里|MRC 的所有权规则]]：arm64e 指针认证对 isa 位域的影响

---

实验环境：Xcode 26.6（clang 21.0.0 / ld 1267.0），macOS 26.5.2，Apple Silicon（arm64），`vm_page_size = 16384`。

全部实验在 macOS 上原生完成，**没有启动任何模拟器**。Mach-O 是文件格式，`otool` / `nm` / `size` / `lipo` / `strip` / `dyld_info` / `dwarfdump` 都是静态解析工具，跑一次 UIKit 也解释不了多一个字节。针对 iOS 的对照（`-target arm64-apple-ios17.0`、`arm64e-apple-ios17.0`）只需要编译，不需要运行。

> 待真机补测：page-in linking 的实际效果（`__DATA_CONST` 在 iOS 16+ 真机上是否计入 dirty memory），需要在设备上用 `vmmap` 或 Xcode Memory Report 对照，本机无法观察。
