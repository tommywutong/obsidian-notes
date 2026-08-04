---
title: "【iOS】Runtime - Part 1 && 对象与类的本质"
published: 2026-05-31
description: "从 objc_object、isa_t、Tagged Pointer、对象内存布局一路拆到类与对象的本质，梳理 Objective-C Runtime 中对象如何关联到类。"
tags: ["iOS", "Objective-C", "Runtime", "isa", "objc4"]
category: "iOS"
series: "iOS Runtime 系列"
seriesSlug: "ios-runtime"
seriesOrder: 1
draft: false
---

# Runtime 简介

写 Objective-C 的人,每天都在敲这样的代码:

```objc
[person sayHello];
```

我们几乎是条件反射地把它读成「调用 person 的 sayHello 方法」。但这其实是一个被其他语言的思维带偏的误读。在 Objective-C 里,方括号语法真正的含义是:**向 person 这个对象,发送一条名为 `sayHello` 的消息。** 编译器最终会把这一行翻译成一次再普通不过的 C 函数调用:

```objc
objc_msgSend(person, @selector(sayHello));
```

到这里,一个平时被我们彻底忽略的问题就浮现了 : **到底是在哪个时刻、由谁来决定 person 真正去执行哪一段代码?**

如果是 C++,这个问题的答案大多在编译期就尘埃落定了——一次普通的成员函数调用,编译完地址就基本刻死在了二进制里。但 Objective-C 偏偏选了另一条路:它把「这个对象到底是什么类」「这条消息对应哪一份方法实现」这些决定,统统推迟到程序跑起来的那一刻,交给 `objc_msgSend` 现场查找、现场拍板。这种「推迟到运行时再决定」的能力,正是我们口中「动态」二字的真正含义。

而支撑起这套动态机制的,是一个始终活在你程序里的库——运行时(Runtime)。

Runtime 是一套用 C（和少量汇编）编写的 API 库，它是 OC「动态性」与「消息发送」机制得以成立的底座。程序启动时，正是它负责把 OC 的类结构注册起来、把分类(Category)的方法整合进宿主类——这些「编译期没做完、留到运行期才落定」的活儿，都由它接手。

我们平时用到的很多能力，底层其实都是 runtime 在支撑：

1. 消息发送与转发机制
2. 动态获取类的属性列表、方法列表等
3. 关联对象（给分类「加」属性）
4. 方法交换 Method Swizzling（俗称 iOS 黑魔法）
5. 分类(Category)的实现
6. KVO 的底层原理

这系列博客也会在后期对每个部分进行深入讲解

# 对象的本质：objc_object

我们可以先通过这篇文章来简单了解Objecti-C：[About Objective-C](https://github.com/XiyouMobile3G-iOS/apple-developer-archive-vault/blob/main/documentation/Cocoa/Programming%20with%20Objective-C/About%20Objective-C.md)

## objc_object：对象的骨架

我们平时写的 `NSObject *obj`、`id`、`Class`，在公开头文件 `objc.h` 里其实是一组**裸指针 typedef**：

```objc
// objc.h
typedef struct objc_class    *Class;   // Class = 指向 objc_class 的指针
typedef struct objc_object   *id;      // id    = 指向 objc_object 的指针
typedef struct objc_selector *SEL;     // SEL   = 指向 objc_selector 的指针
```

也就是说，一个 OC 对象的指针，本质上就是指向一块堆内存的指针，这块内存的开头是一个 `objc_object` 结构体。在新版本 runtime（objc4）里，`objc_object` 长这样：

```
struct objc_object {
private:
    isa_t isa;
public:
    // 一堆操作 isa 的方法：getIsa()、initIsa()、ISA() 等等
};
```


我们首先打开源码工程
这里是入口：他说明OC对象底层至少有一个isa，isa用来找到对象对应的类
```objective-C
/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
```

```objc
struct objc_object {
private:
    char isa_storage[sizeof(isa_t)];
    isa_t &isa() { return *reinterpret_cast<isa_t *>(isa_storage); }
    const isa_t &isa() const { return *reinterpret_cast<const isa_t *>(isa_storage); }
public:
    // ISA() assumes this is NOT a tagged pointer object
    Class ISA(bool authenticated = false) const;
    // rawISA() assumes this is NOT a tagged pointer object or a non pointer ISA
    Class rawISA() const;
    // getIsa() allows this to be a tagged pointer object
    Class getIsa() const;
    uintptr_t isaBits() const;
    // initIsa() should be used to init the isa of new objects only.
    // If this object already has an isa, use changeIsa() for correctness.
    // initInstanceIsa(): objects with no custom RR/AWZ
    // initClassIsa(): class objects
    // initProtocolIsa(): protocol objects
    // initIsa(): other objects
    void initIsa(Class cls /*nonpointer=false*/);
    void initClassIsa(Class cls /*nonpointer=maybe*/);
    void initProtocolIsa(Class cls /*nonpointer=maybe*/);
    void initInstanceIsa(Class cls, bool hasCxxDtor);
    // changeIsa() should be used to change the isa of existing objects.
    // If this is a new object, use initIsa() for performance.
    Class changeIsa(Class newCls);
    bool hasNonpointerIsa() const;
    bool isTaggedPointer() const;
    bool isBasicTaggedPointer() const;
    bool isExtTaggedPointer() const;
    bool isClass() const;
    bool hasCxxDtor() const;        // 类/父类有无 C++ 析构（dealloc 时要不要走 .cxx_destruct）
    // —— 以下省略一大簇方法 ——
    // hasAssociatedObjects / isWeaklyReferenced / retain / release /
    // rootRetain / rootRelease / sidetable_* …… 全是引用计数、关联对象、
    // 弱引用、SideTable 的内容，与「对象 = isa」主线无关，
    // 留到 Part 3（关联对象）、候选 Part 5（weak 与 SideTable）再展开。
};
```

![isa_storage_memory_layout.svg](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/isa_storage_memory_layout.svg)

![CleanShot 2026-05-30 at 13.17.22@2x.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/CleanShot%202026-05-30%20at%2013.17.22%402x.png)

在看新版之前，先把旧版长什么样摆出来对比。这个结构其实经历了三个时代：

```cpp
// ① 古早版（objc4-750 及更早，sunnyxx / draveness 那批老博客里的样子）
struct objc_object {
    isa_t isa;        // public：外部能直接 obj->isa 摸到
};
```

```cpp
// ② objc4-818
struct objc_object {
private:
    isa_t isa;        // 已经 private，但仍是一个「有类型的 isa_t 成员」

public:
    // —— 取 / 初始化 / 修改 isa ——
    Class ISA(bool authenticated = false);   // 非 tagged 对象取类（强制走方法）
    Class rawISA();                          // 取原始 isa（非 nonpointer）
    Class getIsa();                          // 允许 tagged pointer 对象
    uintptr_t isaBits() const;
    void initIsa(Class cls);                 // 新对象初始化 isa
    void initInstanceIsa(Class cls, bool hasCxxDtor);
    void initClassIsa(Class cls);
    void initProtocolIsa(Class cls);
    Class changeIsa(Class newCls);           // 改已存在对象的 isa

    // —— 一批基于 isa 的判定 ——
    bool hasNonpointerIsa();
    bool isTaggedPointer();
    bool isClass();
    bool hasCxxDtor();

    // —— 以下省略一大簇 ——
    // hasAssociatedObjects / isWeaklyReferenced / retain / release /
    // rootRetain / rootRelease / rootDealloc / sidetable_* …… 引用计数、
    // 关联对象、weak、dealloc，留到 Part 3 / 候选 Part 5 再展开。
};
```

```cpp
// ③ objc4-951.1（本文这版）
struct objc_object {
private:
    char  isa_storage[sizeof(isa_t)];        // 退化成「一坨裸字节」
    isa_t &isa() { return *reinterpret_cast<isa_t *>(isa_storage); }  // 只能借方法戴上 isa_t 这副眼镜
public:
    Class ISA(bool authenticated = false) const;
};
```

我们可以看到， isa_storage是真正存isa的地方。旧版本的写法是直接暴露 `isa_t isa` 作为 public 成员，任何人都能直接读写。新版本改成：`char isa_storage[sizeof(isa_t)];` 使用一个 char 数组来占位。arm64e上，isa里面那根指针是被签名过的，如果有一个公开有类型的 isa_t  isa，外部代码就能直接obj->isa.cls直接摸到那根带签名的指针
——读出来是“看着像乱码”的值，甚至可能绕过验证。修改后，想要接触这8个字节都要经过isa()访问器，isa_t里面的 cls又被设置为private，因此你必须`getClass()/setClass()`

- `char` 在 C++ 标准里是"字节类型"，用它做原始存储是合法的 type-punning，不触发 UB（Undefined Behavior）。直接在 union 成员之间互相访问在 C++ 里有严格限制，但通过 `char[]` 中转是标准允许的。

- `char[]` 不会触发任何构造函数或析构函数。`isa_t` 是一个 union，如果直接作为成员，某些编译器版本可能对 union 成员的初始化有额外的限制，而 `char[]` 完全透明、惰性，什么都不做。

- 给对象内部开一块内存，大小刚好等于isa_t，所以实际的内存布局也没有变化

总的来说，这是一次**封装重构**，内存布局和 isa_t 的位域语义完全没变，只是把裸字段换成了私有字节数组 + 私有访问器，防止外部绕过 Runtime 直接操作 isa，同时避免 C++ union 直接访问的潜在 UB 问题。

再回头看这两行，其实它们都指向同一个地方：

```cpp
isa_t &isa() { return *reinterpret_cast<isa_t *>(isa_storage); }  // 内部访问
Class ISA(bool authenticated = false) const;                     // 对外取类
```

不管是内部用的 `isa()`，还是对外的 `ISA()`，它们去读那 8 个裸字节时，都是指的同一个——`isa_t`。换句话说，`objc_object` 这个壳本身没几两肉，它把「对象到底属于哪个类、引用计数是多少、有没有关联对象、是否被弱引用」这些信息，**全都打包压进了 `isa_t` 这 8 个字节里**。

![[素材/isa_storage_to_isa_t_steps.html]]

所以「对象的本质是什么」这个问题，到这里就收敛成了：`isa_t` 里到底装了什么？
## isa_t

```objc
union isa_t {
    isa_t() { } //默认构造函数
    isa_t(uintptr_t value) : bits(value) { }  // 带参数的构造函数
    uintptr_t bits;   //isa_t 里面真正存的是一个和指针一样大的无符号整数(64)。
private:
    // Accessing the class requires custom ptrauth operations, so
    // force clients to go through setClass/getClass by making this
    // private.
    Class cls;
    // 这段放在 private 中，让你不能从外部通过isa.cls直接访问，注释的意思是说：访问 `Class` 指针时，可能需要做 **ptrauth 指针认证**。在 Apple 的 arm64e 架构上，指针可能带有签名，不能像普通地址一样随便读出来用。Runtime 需要通过专门逻辑去认证、解码、还原。
public:
#if defined(ISA_BITFIELD)
// 如果当前平台定义了 ISA_BITFIELD，就启用 isa 位域结构。
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
// 当前平台的 isa 是否支持把一部分引用计数直接存在 isa 里面。
#if ISA_HAS_INLINE_RC
    bool isDeallocating() const {
        return extra_rc == 0 && has_sidetable_rc == 0;
	//extra_rc：存在 isa 里的额外引用计数
	//has_sidetable_rc：是否还有引用计数存在 SideTable 里
    }
    void setDeallocating() {
        extra_rc = 0;
        has_sidetable_rc = 0;
    }
#endif // ISA_HAS_INLINE_RC
#endif // defined(ISA_BITFIELD)
    void setClass(Class cls, objc_object *obj);
    Class getClass(bool authenticated) const;
    Class getDecodedClass(bool authenticated) const;
};
```

如上代码为 isa_t 联合体本体，**union 里所有成员，起始地址相同，共享同一块内存。bits、cls、ISA_BITFIELD struct 都是这块内存的**成员。

bits ：整块 8 字节当作一个 64 位无符号整数。

cls ：同一块 8 字节当作一个 Class 指针来解读。之所以 private，是因为 arm64e 上指针带 PAC 签名，不能直接读，必须走 `setClass` / `getClass` 做签名和验证。

ISA_BITFIELD struct ：同一块 8 字节按「位」拆开，摊成 `nonpointer` / `shiftcls`(arm64e 为 `shiftcls_and_sig`) / `extra_rc` / `has_assoc` / `weakly_referenced` … 一串命名字段。当这 8 字节是 **nonpointer isa**（`nonpointer == 1`，绝大多数普通对象）时，类指针、内联引用计数、各种标志全压在这几段里，直接按名字 `isa.extra_rc`、`isa.has_assoc`、`isa.shiftcls_and_sig` 就能读写，不用自己算位移。

>用一个例子来解释这个union： 假设这块 8 字节内存里存的是 0x011d800100000001
>用bits 读：isa.bits == 0x011d800100000001 就是一个普通的64位整数，没有任何结构 拿来做位运算；用cls读 isa.cls == 0x011d800100000001 把同一个数字当成一个内存地址，认为它指向某个 Class 对象。用 匿名 struct 解读：把同一个数字按 bit 切开，每段单独看；

> 也就是说，内存本身没有类型，字节本身从来没有变过，变的只是你怎么解读他

![[素材/isa_t_three_views.html]]

### ISA_BITFIELD：isa 的位布局

我们接下来看看`ISA_BITFIELD`：
```objc
/*
 * isa 的位布局随架构而不同，共 4 套。下面把每个架构的 ISA_BITFIELD
 * 直接展开成位域列表（省去 #define 和续行符 \，方便阅读），并标注每段
 * 的 bit 范围。挑你的目标架构看即可：真机 A12+ 与模拟器看 ①。
 */

// ===== ① arm64e（真机 A12+ / 模拟器，__has_feature(ptrauth_calls)）=====
//   ISA_MASK = 0x007ffffffffffff8ULL    ·    无独立 has_cxx_dtor 位（移到 cache flags）
uintptr_t nonpointer        : 1;    // bit0     nonpointer isa 开关（0=纯类指针）
uintptr_t has_assoc         : 1;    // bit1     有无关联对象
uintptr_t weakly_referenced : 1;    // bit2     有无被 __weak 弱引用
uintptr_t shiftcls_and_sig  : 52;   // bit3-54  类指针 + PAC 签名（合并）
uintptr_t has_sidetable_rc  : 1;    // bit55    RC 是否溢出到 SideTable
uintptr_t extra_rc          : 8;    // bit56-63 内联引用计数（retainCount-1）

// ===== ② arm64（非 e，不开 PAC）=====
//   ISA_MASK = 0x0000000ffffffff8ULL    ·    保留 has_cxx_dtor / magic 位
uintptr_t nonpointer        : 1;    // bit0
uintptr_t has_assoc         : 1;    // bit1     有无关联对象
uintptr_t has_cxx_dtor      : 1;    // bit2     类/父类有无 C++ 析构（arm64e 已移除）
uintptr_t shiftcls          : 33;   // bit3-35  类指针（无签名，故名 shiftcls）
uintptr_t magic             : 6;    // bit36-41 调试期识别 isa 的魔数（arm64e 已移除）
uintptr_t weakly_referenced : 1;    // bit42    有无被 __weak 弱引用
uintptr_t unused            : 1;    // bit43    保留位
uintptr_t has_sidetable_rc  : 1;    // bit44    RC 是否溢出到 SideTable
uintptr_t extra_rc          : 19;   // bit45-63 内联引用计数（位数比 arm64e 多）

// ===== ③ x86_64（Intel Mac / 旧模拟器）=====
//   ISA_MASK = 0x00007ffffffffff8ULL
uintptr_t nonpointer        : 1;    // bit0
uintptr_t has_assoc         : 1;    // bit1
uintptr_t has_cxx_dtor      : 1;    // bit2
uintptr_t shiftcls          : 44;   // bit3-46  类指针
uintptr_t magic             : 6;    // bit47-52 魔数
uintptr_t weakly_referenced : 1;    // bit53
uintptr_t unused            : 1;    // bit54
uintptr_t has_sidetable_rc  : 1;    // bit55
uintptr_t extra_rc          : 8;    // bit56-63 内联引用计数

// ===== ④ armv7k / arm64_32（Apple Watch 等 32 位，索引式 isa）=====
//   存「类表索引 indexcls」而非类指针；32 位地址空间小，用索引更省位
uintptr_t nonpointer        : 1;    // bit0
uintptr_t has_assoc         : 1;    // bit1
uintptr_t indexcls          : 15;   // bit2-16  类表索引（不是指针！）
uintptr_t magic             : 4;    // bit17-20
uintptr_t has_cxx_dtor      : 1;    // bit21
uintptr_t weakly_referenced : 1;    // bit22
uintptr_t unused            : 1;    // bit23
uintptr_t has_sidetable_rc  : 1;    // bit24
uintptr_t extra_rc          : 7;    // bit25-31 内联引用计数
```

![[素材/isa_t-四套架构位布局对照（可切换）.html]]

这里顺手补一个老版 Runtime 里很经典的问题：为什么字段叫 `shiftcls`，而且旧源码里经常能看到类似 `isa.shiftcls = (uintptr_t)cls >> 3` 的写法？

原因是 **Class 指针按 8 字节对齐**。8 字节就是 `2^3`，所以合法类对象地址的最低 3 位恒为 0。既然这 3 位没有携带有效地址信息，就可以在存进位域前右移 3 位，把高位地址压进更短的 `shiftcls` 字段；取出来时再左移 3 位还原。换句话说，`shiftcls` 不是随便截断指针，而是利用了内存对齐带来的低位空洞。

现代 arm64e 上这一段变成了 `shiftcls_and_sig`：它不再只是“右移后的类指针”，还要和 PAC 签名合在同一段位域里。所以老博客里 `shiftcls >> 3` 的解释适合帮助理解 nonpointer isa 的压缩思路，但不能直接套到 arm64e 的最新布局上。

### arm64e 的 PAC 指针签名（Pointer Authentication）

上面 ① arm64e 那段 `shiftcls_and_sig : 52` 里的 **`_and_sig`**，就是这一节的主角——**PAC 签名**。它是理解「为什么 arm64e 的 isa 和别的架构长得不一样」的最后一块拼图。

**它要解决什么**：攻击者拿到内存写权限后，最爱**篡改指针**（把 isa、函数指针、返回地址改成自己布置的地址）来劫持控制流。CPU 解引用时分不清「这指针是程序写的，还是被人改的」。PAC 是 **ARMv8.3-A 的硬件特性**（苹果 A12 / arm64e 起启用），思路一句话：**给指针盖一个"防伪钢印"，用之前先验章，章不对就崩。**

**原理**：64 位指针其实没用满（用户态地址只用低 ~40 位），PAC 就把空闲的高位拿来存一段签名：

```
未签名:  0x0000_0001_0008_00e8   高位全 0（浪费）
已签名:  0x8a3f_0001_0008_00e8   高位塞进 PAC 签名
         └签名┘└────真实地址────┘
```

这段签名 = `f(密钥 key, 修饰子 modifier, 指针本身的值)`，过一遍硬件加密算法（QARMA）算出。三者任一不同，签名就不同。配套三个硬件动作：
**两个**——密钥和修饰子，objc 给 isa 用的是（`objc-config.h`）：

```objc
#define ISA_SIGNING_KEY            ptrauth_key_process_independent_data  // :239 数据密钥
#define ISA_SIGNING_DISCRIMINATOR  0x6AE1   // :232  = ptrauth_string_discriminator("isa")
```

密钥存在特殊寄存器、用户态读不到；**修饰子**相当于"加盐"，让同一地址在不同用途下签出不同的章。关键在于签 isa 时这样混盐（`objc-object.h` `setClass`）：

```objc
ptrauth_blend_discriminator(obj, ISA_SIGNING_DISCRIMINATOR)  // 把 0x6AE1 和「对象自己的地址 obj」混在一起
```

→ **签名同时绑定了"isa 这个用途"和"这个对象的地址"**。所以攻击者**没法把 A 对象的合法签名 isa 整段拷到 B 对象头上**——地址变了，验签必败。

PAC 不只用在 isa，Runtime 把同一套机制盖在了一批关键指针上，每处用**不同的盐**做用途隔离：

| 被签名的指针 | 修饰子（用途盐） | 在本文哪出现过 |
|---|---|---|
| 对象的 `isa`（类指针） | `"isa"` → `0x6AE1` | 本节 / `shiftcls_and_sig` |
| 类的 `superclass` | `"objc_class:superclass"` → `0xB5AB`（objc-config.h:233） | 「四大件」superclass 那段也带验签 |
| 方法 `IMP` / 缓存 `bucket_t` | 方法/缓存各自的盐 | `cache_t` / `bucket_t::encodeImp` |
| 类数据 `class_rw_t *`（`bits` 里） | `CLASS_DATA_BITS_RW_DISCRIMINATOR` | `## bits` 的 `data()` 验签 |

> PAC 正好把前面那三个bits、cls、ISA_BITFIELD struct 的取舍讲透了——
> ① **`cls` 为什么是 `private`**：因为 arm64e 上 isa 里的类指针**带签名**，直接 `isa.cls` 读出来是"带章的乱码"，不能当地址用，只能走 `getClass`/`setClass` 让访问器去验签/签名。
> ② **签名存在哪**：就压在**位域**的 `shiftcls_and_sig` 高位里（类指针在低位、签名在高位，合成一段 52 位）。这也是 818 起 `shiftcls` 必须扩成 52 位、`ISA_MASK` 加宽到 `0x007ffffffffffff8` 的根因。
> ③ **不验签时怎么取类**：用 **`bits` ** `& ISA_MASK`（或 `objc_debug_isa_class_mask`）把签名位直接 strip 掉拿地址——快，但不防篡改。

而**下一节的 `getClass`它那个 `if (authenticated) { …auth… } else { …strip… }` 分叉，正是在"安全验签"和"图快 strip"之间做选择。

### isa 位域的历史演进（2015 → 至今）

上面那张 arm64e 的图，和在很多老博客里看到的「`shiftcls:33` + `magic` + `deallocating`」并不一样。这不是谁画错了，而是 isa 的位布局本身改过——而且关键的一次改动就发生在 iOS 14→15 之间。把几个节点版本的 objc4 源码摆在一起看就清楚了（以下均为各版本 `isa.h` / `objc-private.h` 实际源码）：

```objc
// ===== objc4-680（2015，iOS 9）：位域还内联在 objc-private.h，没有 isa.h =====
uintptr_t indexed           : 1;    // ← 当年首位字段叫 indexed，不是 nonpointer
uintptr_t has_assoc         : 1;
uintptr_t has_cxx_dtor      : 1;
uintptr_t shiftcls          : 33;
uintptr_t magic             : 6;
uintptr_t weakly_referenced : 1;
uintptr_t deallocating      : 1;    // ← 独立的「正在析构」位
uintptr_t has_sidetable_rc  : 1;
uintptr_t extra_rc          : 19;
// ISA_MASK 0x0000000ffffffff8

// ===== objc4-750.1（2018，iOS 12）：位域搬进 isa.h（ISA_BITFIELD 宏），仅改名 =====
uintptr_t nonpointer        : 1;    // ← indexed 改名 nonpointer，其余一字未动
uintptr_t has_assoc:1; has_cxx_dtor:1; shiftcls:33; magic:6;
uintptr_t weakly_referenced:1; deallocating:1; has_sidetable_rc:1; extra_rc:19;

// ===== objc4-781.2（2020，iOS 14）：和 750 逐字相同，arm64 仍无 arm64e 分支 =====
// （略，与上一段完全一致）

// ===== objc4-818.2（2021，iOS 15）：arm64e 大改，即本文开头那张图 =====
uintptr_t nonpointer:1; has_assoc:1; weakly_referenced:1;
uintptr_t shiftcls_and_sig  : 52;   // 🆕 类指针 + PAC 签名合并
uintptr_t has_sidetable_rc:1; extra_rc:8;
// ISA_MASK 0x007ffffffffffff8；magic / deallocating / has_cxx_dtor 三个老字段全部消失
```

把演进归一下，真正的拐点只有三个：

| 拐点 | 版本 | 变了什么 |
|---|---|---|
| 命名 + 拆文件 | 680 → 750 | 首位 `indexed` 改名 `nonpointer`；位域从 objc-private.h 抽到独立的 `isa.h`（`ISA_BITFIELD` 宏） |
| **arm64e 签名合并** | **781 → 818** | `shiftcls:33` → `shiftcls_and_sig:52`（类指针并入 PAC 签名）；**砍掉 `magic`**、**砍掉 `deallocating`**（改由 `extra_rc==0 && has_sidetable_rc==0` 算出）、`has_cxx_dtor` 移出位域到 cache flags；`ISA_MASK` 由 `0x...0ffffffff8` 加宽到 `0x007ffffffffffff8`；`extra_rc` 从 19 位缩到 8 位（位都让给 52 位签名了） |
| 封装收口 | 818 → 951 | `objc_object` 不再直接放 `isa_t`，改成 `char isa_storage[]` + `isa()` 访问器；成员方法批量加 `const`；RC 宏改为从 `RC_HAS_SIDETABLE_BIT` 派生；新增分平台的 `ISA_MASK_NOSIG`（位域本身不变） |

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260603170233955.png)
老isa图⬆️

一句话记忆：**老博客那张 isa 图（`shiftcls:33`/`magic`/`deallocating`）一直用到 781；分水岭是 781↔818，arm64e 上为了塞进 PAC 签名，一口气合并了 `shiftcls` 并删掉了 `magic`、`deallocating` 两个字段。** 而 781 之前五年（680→781）arm64 布局几乎没动过

![[素材/isa-evolution.html]]

> 注意：上面的对照只针对 **arm64e**。同一份 818/951 源码里，arm64（非 e）、x86_64 分支仍保留着 `magic` / `has_cxx_dtor`（见前一节的 ②③），所以「老字段消失」只发生在开了指针签名的 arm64e 上。

### getClass：从 isa 里取出 Class

刚才我们了解了，isa_t 长什么样、位怎么分布，接下来我们看看 Runtime是怎么从isa_t 里拿到 Class 的？

```objc
// 从 isa 里把 Class 指针抠出来。authenticated 决定要不要做 PAC 指针认证：
// 多数调用方对安全不敏感，默认 false 跳过认证换性能；只有 msgSend / 填缓存
// 这类安全攸关的路径才会传 true。
inline Class
isa_t::getClass(MAYBE_UNUSED_AUTHENTICATED_PARAM bool authenticated) const {
#if SUPPORT_INDEXED_ISA
    // 索引式 isa（armv7k / arm64_32）：cls 本身就是裸类指针，原样返回
    return cls;
#else
    uintptr_t clsbits = bits;          // 先拿到完整的 64 位 isa

#   if __has_feature(ptrauth_calls)    // ===== arm64e：类指针被 PAC 签过名 =====
#       if ISA_SIGNING_AUTH_MODE == ISA_SIGNING_AUTH
    if (authenticated) {
        // 要认证：先用 ISA_MASK 保留「类指针 + 签名」那 52 位（shiftcls_and_sig）
        clsbits &= ISA_MASK;
        if (clsbits == 0)
            return Nil;
        // 再用 ptrauth_auth_data 验签 + 还原出真正能用的指针；鉴别子由
        // this(本 isa 的地址) 和固定常量混合而成，地址不对就还原失败
        clsbits = (uintptr_t)ptrauth_auth_data((void *)clsbits, ISA_SIGNING_KEY,
                      ptrauth_blend_discriminator(this, ISA_SIGNING_DISCRIMINATOR));
    } else {
        // 不认证：直接用运行期算好的 objc_debug_isa_class_mask 把签名/标志/RC 抹掉
        clsbits &= objc_debug_isa_class_mask;
    }
#       else
    clsbits &= objc_debug_isa_class_mask;   // 编译配置不要求认证，同样走快路
#       endif

#   else                               // ===== arm64(非e) / x86_64：指针无签名 =====
    clsbits &= ISA_MASK;               // 一把 ISA_MASK 抹掉低位标志 + 高位 RC 即可
#   endif

    return (Class)clsbits;             // 剩下的就是一根干净的 Class 指针
#endif
}
```

## Tagged Pointer 优化

到这里我们讲的「对象」，都是堆上一块内存、开头一根 `isa` 的普通对象。但其实还有一类「对象」根本没有 `isa`——它就是 **Tagged Pointer（标记指针）**。

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260803003008744.png)

这个指针中，其实只使用了中间高亮部分来表示一个真实的对象指针，由于字节对齐的原因，低位总是0；由于我们不会真正用到所有64进行寻址，所以高位也有一部分总是0。

这个设计不是凭空出现的。2013 年，苹果在 iPhone 5s / A7 把 iOS 带进 64 位时代后，指针从 4 字节变成 8 字节，`NSNumber`、`NSDate` 这类小对象如果仍然都走堆分配，内存压力会被放大。WWDC 2013 Session 404《Advances in Objective-C》里就专门讲过 Tagged Pointer：把小值直接塞进指针后，典型收益可以概括成三点：相关对象内存占用下降、访问更快、创建销毁成本大幅降低。老资料里常引用的量级是：内存约省一半、访问约快 3 倍、创建销毁约快 100 倍。


对一个 `NSNumber *n = @5` 这种**又小又高频**的值类型来说，为了一个 `5` 去 `malloc` 一块堆内存、维护 `isa`、再管引用计数，实在太奢侈。Tagged Pointer 的思路很直接：**不再把这 8 字节只当成普通对象地址，而是拿其中一些位保存类型，剩余位置保存 Payload。** 对最常见的小数值来说，Payload 就是数据本身；在后来引入的特定格式中，Payload 也可能承载一个指向常量数据的指针。无论哪一种，它都不能被当成“指向普通 Objective-C 对象本体的地址”直接解引用。

![Snapzy_2026-08-03_00-31-48_022.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-08-03_00-31-48_022.png)
![Snapzy_2026-08-03_00-32-28_628.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-08-03_00-32-28_628.png)
![Snapzy_2026-08-03_00-32-45_054.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-08-03_00-32-45_054.png)

### Payload、Tag 与 Extended Tag

这三张图都是把一个 64 位的值从 `bit63` 画到 `bit0`：**左边是最高位，右边是最低位**。图中字段的位置会随着架构和系统版本变化，但每一版都在回答相同的三个问题：
#### Payload：这个值真正携带的内容

`Payload` 通常翻译为**载荷**。它是扣除标志位、基础 Tag 和可能存在的 Extended Tag 后，剩下用来表达对象内容的那些位。
例如一个足够小的 `NSNumber`，概念上可以理解为：

```text
[ Tagged Pointer 标志 ][ NSNumber 的 Tag ][ 数字 5 的 Payload ]
```

Runtime 先根据 Tag 找到 `NSNumber` 对应的 Class；之后由这个类的实现解释 Payload，最终让它表现得像一个能够响应 `integerValue`、`description` 等消息的对象。也就是说，Runtime 只提供“识别 Tagged Pointer、找到 Class、取出 Payload”的通用机制，**Payload 的具体编码含义由对应类型决定**：数字可以保存数值位，日期可以保存时间信息，字符串则可能采用自己的字符编码方案。

基础格式通常可以提供 **60 位 Payload**。这不等于所有类型都能无条件使用完整的 60 位：具体类型还可能把其中一些位拿去表达符号、长度或编码方式。某个值装不下时，也不会强行截断，而是退回普通对象表示。

在当前 arm64 的 Split Tagged Pointer 方案中，还有一部分特殊的扩展类型允许 Payload 承载一个对齐后的普通指针，用来引用常量字符串或其他常量数据。这里保存的是“常量数据的地址”，但整个 Tagged Pointer 仍然不是普通 Objective-C 对象本体的地址，不能跳过 Runtime 直接把它当成对象解引用。

#### Tag：告诉 Runtime“我应该表现成哪一类对象”

基础 `Tag` 是一个较小的类型编号。三位一共能表达 `0...7` 八种状态，但其中 `7` 通常被保留为“这是扩展格式”的入口，因此基础格式实际使用 `0...6` 表示普通 Tag。

```text
基础 Tag 不是 Class 指针
        ↓
它只是一个很小的整数索引
        ↓
Runtime 用它查询 Tagged Pointer 类表
        ↓
得到真正的 Class
```

可以把它近似理解成：

```text
tag = 3
    ↓
objc_tag_classes[3]
    ↓
NSNumber 对应的 Class
```

因此，“Tagged Pointer 没有 `isa`”不等于它没有类型。普通对象通过对象本体开头的 `isa` 找到 Class；Tagged Pointer 则从自身位模式中取出 Tag，再去 Runtime 的类表中找到 Class。

Tag 的具体编号属于私有实现，可能随着平台和系统版本变化。源码中的枚举可以帮助理解当前实现，但业务代码不能因为某一版中 `3` 对应 `NSNumber`，就自行截取位并把 `3` 永久写死。

#### Extended Tag：基础 Tag 不够用时的二级类型编号

三位基础 Tag 能容纳的类型非常有限。为了支持更多种类，Runtime 把基础 Tag 的一个值——通常是 `7`，也就是二进制 `111`——定义为**转义标记**：它不直接代表某个 Foundation 类型，而是告诉 Runtime“继续读取旁边额外的 8 位”。这 8 位就是 `Extended Tag`。

```text
基础格式：
基础 Tag != 111
→ 直接使用基础 Tag 查类表
→ 通常保留 60 位 Payload

扩展格式：
基础 Tag == 111
→ 再读取 8 位 Extended Tag
→ 使用扩展 Tag 查另一张类表
→ 通常剩下 52 位 Payload
```

所以 `Extended` 的含义是“**扩展了可表示的类型数量**”，不是“扩展了 Payload”。它恰好相反：为了多保存 8 位类型编号，Payload 会从基础格式的 60 位缩小到 52 位。这是一笔很直观的交换：

```text
基础 Tag：类型少，Payload 大
扩展 Tag：类型多，Payload 小
```

8 位 Extended Tag 理论上提供 256 个槽位，但这不代表系统一定注册或公开使用了全部槽位。哪些类型能采用 Tagged Pointer、使用基础格式还是扩展格式，都由 Apple 的 Runtime 和 Foundation 实现决定。

#### 三张图为什么把这些字段画在不同位置

三张图展示的是同一套思想在不同时代的排列方式，而不是三种同时使用的对象类型：

```text
Intel 旧方案（从 bit63 到 bit0）：
[ Payload 60 位 ][ 基础 Tag 3 位 ][ 标志位 1 位 ]

arm64、iOS 14 以前：
[ 标志位 1 位 ][ 基础 Tag 3 位 ][ Payload 60 位 ]

arm64、iOS 14 起的 Split Tagged Pointer：
[ 标志位 1 位 ][ Payload 60 位 ][ 基础 Tag 3 位 ]
```

当基础 Tag 为 `111`、进入扩展格式后，对应关系是：

```text
Intel 旧方案：
[ Payload 52 位 ][ Extended Tag 8 位 ][ 111 ][ 标志位 ]

arm64、iOS 14 以前：
[ 标志位 ][ 111 ][ Extended Tag 8 位 ][ Payload 52 位 ]

arm64、iOS 14 起：
[ 标志位 ][ Extended Tag 8 位 ][ Payload 52 位 ][ 111 ]
```

arm64 把 Tagged Pointer 标志留在 `bit63`，是为了让 `objc_msgSend` 用一次有符号比较同时筛出两种少见情况：`nil` 等于零，而最高位为 1 的 Tagged Pointer 看作有符号整数时小于零；正常对象地址则大于零。这样最常见的普通对象可以尽快进入读取 `isa` 的路径。

iOS 14 又把三位基础 Tag 移到低位，是因为按 8 字节对齐的普通指针低三位天然为零，正好可以利用；扩展 Tag 则放进 arm64 的高位区域，配合 Top Byte Ignore 等平台能力。这样既保留了 `bit63` 对消息发送的快速判断，又让部分扩展格式的 52 位 Payload 能容纳一个对齐后的常量数据指针。

```text
先看标志位，识别 Tagged Pointer
        ↓
读取基础 Tag
        ↓
若基础 Tag 为 111，再读取 Extended Tag
        ↓
根据 Tag 查询 Class
        ↓
由对应类型解释 Payload
```



### 判定一个指针是不是 Tagged Pointer

判定逻辑只有一行（`objc-internal.h`）：

```objc
static inline bool
_objc_isTaggedPointer(const void * _Nullable ptr)
{
    // 标记位被置 1 的，就不是真指针，而是 Tagged Pointer
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
```

关键就在 `_OBJC_TAG_MASK` 这个「标记位」放在哪，随平台不同：

```objc
#if OBJC_SPLIT_TAGGED_POINTERS        // arm64（iOS 真机 / Apple Silicon）
#   define _OBJC_TAG_MASK (1UL<<63)   // 看最高位 bit63
#elif OBJC_MSB_TAGGED_POINTERS        // 非 arm64、非 x86 Mac 的其它配置
#   define _OBJC_TAG_MASK (1UL<<63)
#else                                 // x86_64 Mac
#   define _OBJC_TAG_MASK 1UL         // 看最低位 bit0
#endif
```

这就解释了一个常见现象：真机上打印一个 Tagged 的 `NSNumber`，地址常是 `0xb000...` / `0x8000...` 这种「最高位为 1 的怪地址」——因为那根本不是地址，是被置了标记位的数据。

### 它标记的是哪些类

指针里除了标记位，还存了一个 `tag`，用来标识「这是哪个类的对象」。`tag` 的取值来自一张枚举表（`objc-internal.h`，这里节选）：

```objc
enum objc_tag_index_t : uint16_t
{
    // tag 0..6：60 位载荷
    OBJC_TAG_NSAtom            = 0,
    OBJC_TAG_NSString          = 2,
    OBJC_TAG_NSNumber          = 3,
    OBJC_TAG_NSIndexPath       = 4,
    OBJC_TAG_NSManagedObjectID = 5,
    OBJC_TAG_NSDate            = 6,
    OBJC_TAG_RESERVED_7        = 7,   // 保留

    // tag 8..263：52 位扩展载荷（NSColor / UIColor / NSIndexSet ...）
    OBJC_TAG_Photos_1          = 8,
    OBJC_TAG_NSColor           = 16,
    OBJC_TAG_UIColor           = 17,
    // ...
};
```

所以 arm64 上那 8 个字节大致是这样分的：

```
bit63        标记位（=1 表示这是 Tagged Pointer）
低位若干 bit   tag —— 它是哪个类（NSNumber? NSString? NSDate?）
中间 bit      payload —— 真正的数据
```

tag 0–6 有 60 位载荷，扩展 tag 有 52 位载荷。**装不下就退回普通堆对象**——比如一个很大的整数、一个很长的字符串，就不会做成 Tagged Pointer。

### 没有 isa，它怎么找到类

这一点正好和前面 `getClass` 那节对照着看。普通对象靠 `isa.bits & mask` 拿到 `Class`；而 Tagged Pointer 根本没有 `isa`，于是 `getIsa()` 走了另一条路（`objc-object.h`）：

```objc
inline Class
objc_object::getIsa() const
{
    // 普通对象：还是走 isa
    if (fastpath(!isTaggedPointer())) return ISA(/*authenticated*/true);

    // Tagged Pointer：从指针里抠出 tag 当下标，去全局表里查类
    uintptr_t slot = ((uintptr_t)this >> _OBJC_TAG_SLOT_SHIFT) & _OBJC_TAG_SLOT_MASK;
    Class cls = objc_tag_classes[slot];   // 即 objc_debug_taggedpointer_classes[]
    // ...（扩展 tag 再查 ext 表）
    return cls;
}
```

它把指针里的 `tag` 当作下标，去 `objc_debug_taggedpointer_classes[]` 这张「tag → Class」的全局表里查出类。

这也顺带解释了前面 `objc_object` 源码里那一堆 `ASSERT(!isTaggedPointer())`——像 `ISA()` 开头就断言「我不是 tagged」，因为 `ISA()` 只处理真有 `isa` 的对象，**Tagged Pointer 必须改走 `getIsa()`**。

### 为什么内存里看到的值像「乱码」

iOS 8.3 之后，Tagged Pointer 的真实布局在存进指针前，会先和一个全局混淆值 `objc_debug_taggedpointer_obfuscator` 做一次**异或加扰**：

```objc
uintptr_t value = (obfuscator ^ ptr);   // 编码/解码都要异或这个值
```

目的是**防止开发者去硬编码、依赖它的内部位布局**（Apple 保留随时改布局的权利）。所以你直接在内存里看到的 Tagged 指针位是被混淆过的，要 decode 才能还原出真正的 tag 和 payload。

### 它带来的好处

- **零分配**：不 `malloc`、不进堆，创建和销毁几乎零成本；
- **引用计数免费**：Tagged 对象的 `retain / release` 直接是 no-op——源码里 `rootRetain` 开头就是 `if (isTaggedPointer()) return (id)this;` 原样返回；
- **访问快**：数据就在指针里，无需解引用去堆上读。

> **几点边界**
> 1. 不是所有 `NSNumber / NSString` 都是 Tagged——超出载荷容量（大数、长字符串）会退回普通堆对象。
> 2. Tagged 对象 `object_getClass` 能拿到类（如 `__NSCFNumber` / `NSTaggedPointerString`），但它**没有 isa 字段**，行为和普通对象不完全一样（retain 免费、布局被混淆）。
> 3. 布局随架构 / 系统版本会变（arm64 看 bit63、x86 Mac 看 bit0），别把某个具体 bit 位当成「永远如此」写死。



下面用一张图总结一下：isa_t 
![[素材/isa_t_layout 1.html]]


# 对象的内存布局
## 实例里装了什么：isa + 成员变量

一个普通实例对象在堆上的内存，结构非常朴素：

```
偏移 +0     isa（8 字节）
偏移 +8     第 1 个成员变量(ivar)
偏移 ...     第 2 个成员变量
            ...（按声明顺序、各自对齐规则排布）
```

`isa` 永远在 `+0`，紧接着是这个类（含父类）的所有成员变量，按声明顺序、各自的对齐要求依次排下去。换句话说——

> **实例对象 = 一根 isa + 一串成员变量。** 它身上没有方法，方法都存在「类」里（下一章讲）。

这里有个**源码和内存对不上的地方**：你去翻 `objc_object` 源码，会发现它**只有 isa**（`char isa_storage[sizeof(isa_t)]`），根本没声明任何成员变量——那上图里 `+8` 之后的 ivar 是哪来的？

答案是：**ivar 不是 `objc_object` 的 C 结构体字段，而是 `alloc` 时拼在 isa 之后的尾巴**，它的位置由元数据描述、靠偏移量寻址。源码里 `objc_object` 只写了 `isa_t isa`，是因为成员变量并非写死在结构体里，而是编译器根据类的 `ivar_list` 在 `isa` 之后动态布局并通过偏移量访问的。`objc_object` 只盖住「所有对象都有的公共头」（isa），而每个类各不相同的 ivar，记在类的只读数据里（`objc-runtime-new.h`）：

```objc
struct ivar_t {
    int32_t *offset;          // :1214  这个 ivar 距对象开头多少字节
    const char *name;
    const char *type;
    // ...
};
struct class_ro_t {
    // ...
    const ivar_list_t * ivars;   // :1614  本类所有 ivar 的清单（含各自 offset）
};
```

取值时不是 `obj->field`，而是 **「对象首地址 + offset」** 强转类型再读写（`objc-class.mm:583` / `ivar_getOffset` `objc-runtime-new.mm:4817`）：

```objc
(char *)object + *ivar.offset          // 基址 + 偏移
ivar_getOffset(Ivar v){ return *v->offset; }
```

你写的 `person->_age`，编译器其实编成 `*(int *)((char *)person + OBJC_IVAR_$_Person._age)`——这正是上图 `+8` 的由来（也是后面 LLDB 实测里 `_age` 落在对象 `+8` 处的原因）。

> 一句话：源码里 `objc_object` 只有 isa，是因为它只是「对象的公共头」；ivar 是 alloc 时拼在 isa 后面、由 `class_ro_t.ivars` 里每个 `ivar_t.offset` 记录位置、靠 `(char*)obj + offset` 访问的变长部分。这也是经典的「变长对象」套路——结构体只固定头 8 字节，后面是按偏移寻址的裸内存。

这也正是为什么 `NSObject` 的实例「最小」：它一个自定义成员都没有，身上就只有那 8 字节的 isa。

## 对象多大：从「需求」到「实分」的三个数

「一个对象多大」这个问题，其实有**三个互不相同**的答案。一个对象从「源码声明」走到「躺在堆上」，大小要过三道关：


第①步的 `instanceSize` 是编译器在编译期就算好、存进类的只读数据 `class_ro_t` 里的（`isa` 加上所有成员变量的字节数）。第②步把它按字长（64 位下是 8 字节）向上对齐，这就是 `class_getInstanceSize` 返回的值：

```objc
// objc-class.mm:817 —— class_getInstanceSize 只到「对齐后的 ivar 需求」，没有 16 下限
size_t class_getInstanceSize(Class cls) {
    if (!cls) return 0;
    cls->realizeIfNeeded();
    return cls->alignedInstanceSize();   // = word_align(ro->instanceSize)，按 8 字节对齐
}
```

但**真正 alloc 时用的不是它**，而是 `instanceSize()`——这里才加上了「最小 16 字节」的下限：

```objc
// objc-runtime-new.h:3144 —— runtime 真正分配时用的大小
inline size_t instanceSize(size_t extraBytes) const {
    if (fastpath(cache.hasFastInstanceSize(extraBytes)))   // 缓存里有算好的就走快路
        return cache.fastInstanceSize(extraBytes);

    size_t size = alignedInstanceSize() + extraBytes;
    if (size < 16) size = 16;   // CF requires all objects be at least 16 bytes.
    return size;
}
```

最后在创建实例的核心函数里，把这个 `size` 交给 `malloc` 去要内存：

```objc
// objc-runtime-new.mm:9309 —— _class_createInstance_realized 节选
size = cls->instanceSize(extraBytes);     // 拿到「抬到 ≥16」之后的大小
id obj = objc::malloc_instance(size, cls); // 向堆申请
// ...
obj->initInstanceIsa(cls, hasCxxDtor);     // 把 isa 写进对象开头那 8 字节
```

而 `malloc` 自己还有**分桶粒度**（在 64 位上以 16 字节为单位），所以最终 `malloc_size()` 拿到的实际块，往往会再被向上取整到 16 的倍数。

## 实战：NSObject 为什么是「8 需求 / 16 实分」

把上面三个数套到具体的类上，就一目了然了：

| 类 | ① isa + ivar | ② `class_getInstanceSize`（对齐后） | ③ `malloc_size`（堆实分） |
|---|---|---|---|
| `NSObject`（只有 isa） | 8 | **8** | **16** |
| `Person { int age }` | 8 + 4 = 12 | 16（对齐到 8 的倍数） | 16 |
| `Person { NSString *name; int age }` | 8 + 8 + 4 = 20 | **24** | **32** |

- **`NSObject` 的「8 需求 / 16 实分」**：`class_getInstanceSize` 只返回 8（刚够装一根 isa），但真分配时 `instanceSize()` 把它抬到 16，`malloc` 也按 16 给——所以 `malloc_size` 是 16。这就是那句经典面试题「NSObject 对象占多少内存」的完整答案：**理论需求 8，实际占用 16。**
- **第三个例子最能说明「三个数互不相等」**：成员变量需求 20，对齐成 24，`malloc` 按 16 分桶再抬到 32。

到这里，「对象的本质」就讲完整了：**一根 isa + 一串成员变量，在堆上占着一块向上对齐到 16 倍数的内存。** 那 isa 指向的、成员变量大小所记录在的那个「类」，本身其实也是一个对象。

# 类的本质：objc_class

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260617160338329.png)

## 类本身也是对象

打开 `objc_class` 的定义，第一行就说明了一切：

```objc
// objc-runtime-new.h:2635
struct objc_class : objc_object {
    // Class ISA;               // 继承自 objc_object，指向元类
    Class superclass;
    cache_t cache;              // formerly cache pointer and vtable
    class_data_bits_t bits;     // class_rw_t * plus custom rr/alloc flags

    Class getSuperclass() const;               // 读父类（arm64e 下验签，见下文 superclass 小节）
    void  setSuperclass(Class newSuperclass);  // 写父类（arm64e 下签名）
    class_rw_t *data() const { return bits.data(); }
    void setData(class_rw_t *newData) { bits.setData(newData); }
    const class_ro_t *safe_ro() const { return bits.safe_ro(); }
    // ... isMetaClass()/getMeta()/instanceSize() 等大量方法（元类相关留 §4）
};
```

`objc_class` 继承自 `objc_object`，这意味着**类也是一个对象**，它同样从 `objc_object` 那里继承了一根 `isa`。所以前面讲对象时的那套（`isa`、`isa_t`、`getClass`）对类一样成立——只不过类的 `isa` 指向的不是普通类，而是**元类（metaclass）**，这个留到下一章。

## 类的四大件：isa / superclass / cache / bits

从上面的定义可以看出，一个类身上就四样东西：

1. **`isa`**（继承自 `objc_object`）→ 指向它的**元类**
2. **`superclass`** → 它的父类（`NSObject` 的 superclass 为 `nil`，这条链是「继承」的物理体现）
3. **`cache`** → 方法缓存。一条消息查到对应的实现后，会缓存在这里，下次同样的消息直接命中、跳过慢速查找。
4. **`bits`** → 类数据的「入口」。它不是直接把方法、属性等内容塞在一个整数里，而是用一根“指针 + 标志位”带我们找到后面的 `class_ro_t`、`class_rw_t`，必要时再找到 `class_rw_ext_t`。

### superclass：父类指针

`superclass` 就是一根指向父类的 `Class` 指针。但在 arm64e 上，它和 `isa` 一样会被 PAC 签名（具体见前文），所以读写都得走专门的访问器，不能直接当地址用：

```objc
// objc-runtime-new.h:2641 / 2645 / 2671
Class superclass;

Class getSuperclass() const {
#if __has_feature(ptrauth_calls)
#   if ISA_SIGNING_AUTH_MODE == ISA_SIGNING_AUTH
    if (superclass == Nil) return Nil;
#if SUPERCLASS_SIGNING_TREAT_UNSIGNED_AS_NIL
    void *stripped = ptrauth_strip((void *)superclass, ISA_SIGNING_KEY);
    if ((void *)superclass == stripped) {
        void *resigned = ptrauth_sign_unauthenticated(stripped, ISA_SIGNING_KEY,
            ptrauth_blend_discriminator(&superclass, ISA_SIGNING_DISCRIMINATOR_CLASS_SUPERCLASS));
        if ((void *)superclass != resigned) return Nil;
    }
#endif
    void *result = ptrauth_auth_data((void *)superclass, ISA_SIGNING_KEY,
        ptrauth_blend_discriminator(&superclass, ISA_SIGNING_DISCRIMINATOR_CLASS_SUPERCLASS));
    return (Class)result;
#   else
    return (Class)ptrauth_strip((void *)superclass, ISA_SIGNING_KEY);
#   endif
#else
    return superclass;
#endif
}

void setSuperclass(Class newSuperclass) {
#if ISA_SIGNING_SIGN_MODE == ISA_SIGNING_SIGN_ALL
    superclass = (Class)ptrauth_sign_unauthenticated((void *)newSuperclass, ISA_SIGNING_KEY,
        ptrauth_blend_discriminator(&superclass, ISA_SIGNING_DISCRIMINATOR_CLASS_SUPERCLASS));
#else
    superclass = newSuperclass;
#endif
}
```

可以看到，这和前面 `isa` 的 `getClass` 是同一套路数：arm64e 上凡是存在结构体里的关键指针（isa、superclass）都被签了名，读写必须经访问器验签 / 签名。`NSObject` 的 `superclass` 为 `nil`，整条 `superclass` 链就是「继承关系」的物理体现。

### cache：方法缓存

为什么类对象里要专门放一块 `cache_t`？原因很朴素：消息发送太频繁，而方法调用分布又极不均匀。一个类可能有很多方法，但真实运行时经常被反复调用的只是一小部分。老文章常用“二八原则”来解释：大约 20% 的方法贡献了 80% 的调用次数。既然热点 selector 会被一遍遍发送，Runtime 就没必要每次都从方法列表、父类链重新扫一遍。

完整查找链大致是这样：

1. receiver 通过 `isa` 找到类对象。
2. 先查这个类的 `cache_t`，命中就直接拿到 `IMP` 执行。
3. 未命中再去本类方法列表查。
4. 本类没有，就沿 `superclass` 到父类继续查。
5. 找到后把 `SEL -> IMP` 写回最初接收者所属类的缓存。
6. 继承链都找不到，才进入动态方法解析、消息转发，或者最终抛出 unrecognized selector。

这段流程 Part 2 会完整展开；在 Part 1 这里只要记住：`cache_t` 是类对象上的热路径加速器，它服务的是“同一个 selector 反复发给同一类对象”这种最高频场景。

```objc
// objc-runtime-new.h:337
struct cache_t {
private:
	// 缓存的真实存储，可以理解为：_bucketsAndMaybeMask -> bucket_t数组 —> 每个 bucket 存一组 SEL -> IMP
    explicit_atomic<uintptr_t> _bucketsAndMaybeMask;   // 桶数组指针（可能拼着 mask）
    union {
    //  普通情况下，当成struct看
        struct {
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED && !__LP64__
            explicit_atomic<mask_t>    _mask; // 缓存容量掩码，用来算 bucket 下标
            uint16_t                   _occupied;  //  当前用了多少个 bucket
            
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED && __LP64__
            explicit_atomic<mask_t>    _mask;
            uint16_t                   _occupied;
            uint16_t                   _flags;
#elif __LP64__   // 内联 mask，64 位，这里没有_mask，因为mask 被内联编码进 _bucketsAndMaybeMask 里面了。也就是说前面的_bucketsAndMaybeMask，不只是保存 buckets 指针，还顺便藏了 mask。所以第二个 word 就空出来一部分，可以放：
            uint32_t                   _disguisedPreoptCacheSignature;// 主要是区分是普通cache 还是伪装过的 preopt cache，这里放了一个用于 preopt cache 识别的 signature
            uint16_t                   _occupied;
            uint16_t                   _flags;
#else            // 内联 mask，32 位
            uint16_t                   _occupied;
            uint16_t                   _flags;
#endif
        };
        // 这是union的另一个成员
        explicit_atomic<preopt_cache_t *> _originalPreoptCache;   // dyld shared cache 里的预优化方法缓存，这个主要给系统类用，一些方法缓存可以在系统共享缓存里提前计算好，App运行不一定从空缓存开始
    };

    cache_t() : _bucketsAndMaybeMask(0) {}

    // —— 各 CACHE_MASK_STORAGE 方案下的 bucketsMask/maskShift 等 constexpr（337–453，存储方案内部常量，从略）——
	//是有工具函数
    bool isConstantEmptyCache() const;
    bool canBeFreed() const;
    mask_t mask() const;
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);
    void reallocate(mask_t oldCapacity, mask_t newCapacity, bool freeOld);
    void collect_free(bucket_t *oldBuckets, mask_t oldCapacity);
    
    // 创建各种buckets
    static bucket_t *emptyBuckets();
    static bucket_t *mallocBuckets(mask_t newCapacity);
    static bucket_t *allocateBuckets(mask_t newCapacity);
    static bucket_t *emptyBucketsForCapacity(mask_t capacity, bool allocate = true);
    static struct bucket_t *endMarker(struct bucket_t *b, uint32_t cap);
    void bad_cache(id receiver, SEL sel) __attribute__((noreturn, cold));

public:
    unsigned capacity() const;
    struct bucket_t *buckets() const;
    Class cls() const;
    mask_t occupied() const;
    void initializeToEmpty();
	// 预优化缓存相关函数
    bool isConstantOptimizedCache(bool strict = false, uintptr_t empty_addr = 0) const;
    bool shouldFlush(SEL sel, IMP imp) const;
    bool isConstantOptimizedCacheWithInlinedSels() const;
    void initializeToEmptyOrPreoptimizedInDisguise();

    void insert(SEL sel, IMP imp, id receiver);     // 写缓存（Part2 主角）
    void copyCacheNolock(objc_imp_cache_entry *buffer, int len);
    void destroy();
    void eraseNolock(const char *func);
    static void init();
    static void collectNolock(bool collectALot);
    static size_t bytesForCapacity(uint32_t cap);

    // ===== FAST_CACHE_* 标志（存在 _flags 里）=====
#if CACHE_T_HAS_FLAGS  // 当前平台 cache_t 支持 _flags 字段时才启用这些快速标志
#   if __arm64__
#       define FAST_CACHE_HAS_CXX_DTOR       (1<<0)   // 第 0 位，便于 bfi 进 isa_t::has_cxx_dtor
#       define FAST_CACHE_HAS_CXX_CTOR       (1<<1)
#       define FAST_CACHE_META               (1<<2)
#   else
#       define FAST_CACHE_META               (1<<0)
#       define FAST_CACHE_HAS_CXX_CTOR       (1<<1)
#       define FAST_CACHE_HAS_CXX_DTOR       (1<<2)
#   endif
#   if __LP64__
#       define FAST_CACHE_ALLOC_MASK         0x0ff8
#       define FAST_CACHE_ALLOC_MASK16       0x0ff0
#       define FAST_CACHE_ALLOC_DELTA16      0x0008
#       define FAST_CACHE_FLAGS_MASK         0xf000
#   endif
#   define FAST_CACHE_HAS_CUSTOM_DEALLOC_INITIATION (1<<12)
#   define FAST_CACHE_REQUIRES_RAW_ISA   (1<<13)   // 实例必须用 raw isa
#   define FAST_CACHE_HAS_DEFAULT_AWZ    (1<<14)   // 有默认 alloc/allocWithZone（存元类里）
#   define FAST_CACHE_HAS_DEFAULT_CORE   (1<<15)   // 有默认 new/self/class/...

    bool getBit(uint16_t flags) const { return _flags & flags; }
    void setBit(uint16_t set)   { __c11_atomic_fetch_or ((_Atomic(uint16_t)*)&_flags,  set, __ATOMIC_RELAXED); }
    void clearBit(uint16_t clr) { __c11_atomic_fetch_and((_Atomic(uint16_t)*)&_flags, ~clr, __ATOMIC_RELAXED); }
#endif

    // ===== 快速实例大小（§2 instanceSize 走的快路）=====
#if FAST_CACHE_ALLOC_MASK
    bool hasFastInstanceSize(size_t extra) const {
        if (__builtin_constant_p(extra) && extra == 0) return _flags & FAST_CACHE_ALLOC_MASK16;
        return _flags & FAST_CACHE_ALLOC_MASK;
    }
    size_t fastInstanceSize(size_t extra) const {
        ASSERT(hasFastInstanceSize(extra));
        if (__builtin_constant_p(extra) && extra == 0) return _flags & FAST_CACHE_ALLOC_MASK16;
        size_t size = _flags & FAST_CACHE_ALLOC_MASK;
        return align16(size + extra - FAST_CACHE_ALLOC_DELTA16);   // 去掉 setFastInstanceSize 加的 DELTA16
    }
    void setFastInstanceSize(size_t newSize) {
        uint16_t newBits = _flags & ~FAST_CACHE_ALLOC_MASK;
        uint16_t sizeBits = word_align(newSize) + FAST_CACHE_ALLOC_DELTA16;
        sizeBits &= FAST_CACHE_ALLOC_MASK;
        if (newSize <= sizeBits) newBits |= sizeBits;
        _flags = newBits;
    }
#endif
};
static_assert(sizeof(cache_t) == 2 * sizeof(void *), "cache_t must be two words");
```

缓存的桶 `bucket_t`（一个 `SEL → IMP` 槽）：

```objc
// objc-runtime-new.h:214
struct bucket_t {
private:
#if __arm64__                          // arm64：IMP 在前（利于 ptrauth）
    explicit_atomic<uintptr_t> _imp;
    explicit_atomic<SEL> _sel;
#else                                  // 其它：SEL 在前
    explicit_atomic<SEL> _sel;
    explicit_atomic<uintptr_t> _imp;
#endif
    uintptr_t modifierForSEL(bucket_t *base, SEL newSel, Class cls) const {
        return (uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;   // ptrauth 签名修饰子
    }
    uintptr_t encodeImp(bucket_t *base, IMP newImp, SEL newSel, Class cls) const;  // IMP 按方案编码/签名
public:
    inline SEL sel() const { return _sel.load(memory_order_relaxed); }
    inline IMP rawImp(objc_class *cls) const;   // 解码取出 IMP
    // ... set()/imp() 等
};
```

一句话：`cache` 把「最近用过的 `SEL` → `IMP`」缓存起来，让重复的消息发送走极快的命中路径。还记得 §2 讲 `instanceSize` 时提到的 `FAST_CACHE_*` 吗？它存的就是这里的 `_flags`——苹果把一批高频标志和实例大小塞进了 `cache`，正是为了在发消息的路径上少绕一层。

```objc
// 旧版（objc4-750 及之前）
struct cache_t {
    struct bucket_t *_buckets;   // 桶数组指针
    mask_t           _mask;      // 桶数 - 1
    mask_t           _occupied;  // 已占用桶数
};
```

![[素材/cache_t_layout.html]]

### bits：类数据的入口（ro → rw → rw_ext）

`bits` 本身是一个 `class_data_bits_t`，内部核心仍是一个机器字大小的整数。这个整数同时承载**类数据指针**和少量**快速标志位**，方法、属性、协议并不直接存放在这 8 字节里。

而且 `bits` 后面的结构不是始终不变的。类刚从 Mach-O 映射进来、尚未被 realize 时，它直接指向编译期生成的 `class_ro_t`；类第一次真正被 Runtime 使用后，Runtime 才分配 `class_rw_t` 并让 `bits` 改为指向它。绝大多数类到这里就够了，只有确实需要合并 Category 或动态修改方法、属性、协议时，才继续按需分配 `class_rw_ext_t`：

```text
编译产物 / 尚未 realize：
objc_class.bits ──────────────→ class_ro_t

realize 之后：
objc_class.bits ─data()───────→ class_rw_t
                                      │
                                      ▼
                               ro_or_rw_ext（二选一）
                                  ├─通常────→ class_ro_t
                                  └─需要扩展→ class_rw_ext_t
                                                ├─ ro ─→ class_ro_t
                                                ├─ methods
                                                ├─ properties
                                                └─ protocols
```

> 这里直接阅读 `objc_class`、`class_rw_t` 等私有结构，是为了理解 Runtime 的实现，不代表业务代码应该依赖它们的字段和偏移。Apple 在 WWDC20 特别提醒：这些布局会随系统版本改变。正式代码应使用 `class_getName`、`class_getSuperclass`、`class_copyMethodList` 等公开 Runtime API。

#### 先分清两个同名的 bits

> 前面讲对象时，`isa_t` 里也有个 `uintptr_t bits`（isa 那 8 字节本身）。这里类对象的 `bits` 是 `class_data_bits_t`，和它**同名、套路相同（都是"指针+标志位塞进一个字"）、但管的事完全不同**。一路从 isa 看下来到这儿容易卡住，单独拎出来对比一下：

| 维度   | `isa_t.bits`（每个对象都有）                                               | `class_data_bits_t.bits`（只有类对象有）                                                                                           |
| ---- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| 位置   | 对象头那 8 字节                                                          | `objc_class` 第 4 个字段；在本文 objc4-951.1 的 64 位布局里位于 `isa/super/cache` 之后，即偏移 `0x20`（这是私有实现，不是稳定 ABI）                          |
| 指针载荷 | **类指针**（arm64e 下还涉及 PAC 签名）                                        | realize 前为 **`class_ro_t *`**，realize 后为 **`class_rw_t *`**                                                                |
| 附加信息 | 引用计数 `extra_rc` + `nonpointer`/`has_assoc`/`weakly_referenced` 等标志 | 低 3 位是 `FAST_IS_SWIFT_LEGACY`、`FAST_IS_SWIFT_STABLE`、`FAST_HAS_DEFAULT_RR`；64 位下另用最高位 `FAST_IS_RW_POINTER` 标记当前是否已经指向 `rw` |
| 取chu | 按当前平台的 `ISA_MASK` / PAC 流程取得 **Class / 元类**                        | 按 `FAST_DATA_MASK` / PAC 流程取得 **`class_ro_t` 或 `class_rw_t`**                                                              |


`isa_t.bits` 解决的是“**对象属于哪个类**”，`class_data_bits_t.bits` 解决的是“**这个类的数据在哪里**”。

#### 从旧版到新版

下面先看旧版（以 objc4-779.1 及更早版本为例），再看本文使用的新版。放旧版不是为了让你背两套源码，而是因为旧版把 `ro`、方法数组和运行期字段直接铺在 `class_rw_t` 中，更容易看清各自职责；新版主要是在此基础上增加原子访问、PAC，并把少用的可变数据拆进 `class_rw_ext_t`。

第一次读这些代码只追踪一件事：**类名、实例大小和原始方法列表在 `ro`；运行期状态在 `rw`；需要合并或修改的方法、属性、协议才进入 `rw_ext`。**

##### 旧版：ro 和可变数组直接挂在 rw 上

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802220821840.png)


```objc
// 旧版 class_data_bits_t：bits 是裸 uintptr_t，无 atomic / 无 ptrauth
struct class_data_bits_t {
    // Values are the FAST_ flags above.
    uintptr_t bits;
};

// 旧版 class_rw_t：ro / methods / properties / protocols 四个「直接内嵌」，还带 version
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t   methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
};

// 旧版 class_ro_t：name 是裸 const char*；方法表叫 baseMethodList（+baseMethods() 访问器）；
//                 baseProtocols/baseProperties 是普通指针；ivarLayout 是独立字段
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t *ivarLayout;

    const char      *name;
    method_list_t   *baseMethodList;
    protocol_list_t *baseProtocols;
    const ivar_list_t *ivars;

    const uint8_t   *weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```

##### 新版入口：class_data_bits_t

看完旧版的直观关系，再看新版 `bits` 怎样保存 `ro` 或 `rw` 指针。下面的 `FAST_*` 掩码负责从同一个机器字里区分真正的地址和附加标志；只需看懂 `has_rw_pointer()`、`data()`、`safe_ro()` 三个入口。

`bits` 的真身 `class_data_bits_t`（它用到的 `FAST_*` 掩码先列出）：
```objc
// objc-runtime-new.h:125（__LP64__） 下面三行是低三位标志
#define FAST_IS_SWIFT_LEGACY    (1UL<<0)
#define FAST_IS_SWIFT_STABLE    (1UL<<1)
#define FAST_HAS_DEFAULT_RR     (1UL<<2)

#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR  // 下面这条掩码用来扣出真正的指针地址
#define FAST_DATA_MASK          0x0f00007ffffffff8UL
#else
#define FAST_DATA_MASK          0x0f007ffffffffff8UL
#endif
#define FAST_FLAGS_MASK         0x0000000000000007UL   // 取低三位：把最低三位标志抠出来
#define FAST_IS_RW_POINTER      0x8000000000000000UL   // 快速判断这是 rw 指针而非 ro：realize 前 bits 指向 ro，realize 后指向包装了 ro 的 rw
```
```objc
// objc-runtime-new.h:2364
struct class_data_bits_t {
    friend objc_class;
    explicit_atomic<uintptr_t> bits;   // 存的是 class_ro_t* 或 class_rw_t*（低位塞 FAST_ 标志）
private:
    bool getBit(uintptr_t bit) const { return bits.load(std::memory_order_relaxed) & bit; }
    void setAndClearBits(uintptr_t set, uintptr_t clear);   // 改标志位时带 ptrauth 验签+重签
    void setBits(uintptr_t set)   { setAndClearBits(set, 0); }
    void clearBits(uintptr_t clr) { setAndClearBits(0, clr); }
public:
    void copyRWFrom(const class_data_bits_t &other);
    void copyROFrom(const class_data_bits_t &other, bool authenticate);

	// 判断当前bits里是不是 class_rw_t 
    bool has_rw_pointer() const { return has_rw_pointer(bits.load(std::memory_order_relaxed)); }
    static bool has_rw_pointer(uintptr_t bits) {
#if FAST_IS_RW_POINTER
        return (bool)(bits & FAST_IS_RW_POINTER);
#else
        return bits != 0 && (flags(bits) & RW_REALIZED);
#endif
    }    // 最高位是 1，说明是rw，如果是 0，存的是ro

    class_rw_t *data() const {       // 取 rw：arm64e 下验签 + FAST_DATA_MASK 抠地址
        ASSERT(has_rw_pointer());
        uintptr_t localBits = bits.load(std::memory_order_relaxed);
        return (class_rw_t *)((uintptr_t)ptrauth_auth_data((class_rw_t *)localBits,
            CLASS_DATA_BITS_RW_SIGNING_KEY,
            ptrauth_blend_discriminator(&bits, CLASS_DATA_BITS_RW_DISCRIMINATOR)) & FAST_DATA_MASK);
    }
    void setData(class_rw_t *newData);   // 存 rw：FAST_FLAGS_MASK | 新指针 | FAST_IS_RW_POINTER，再签名

    const class_ro_t *safe_ro() const {  // 并发 realize 期间也能安全取 ro
        uintptr_t bitsValue = bits.load(std::memory_order_relaxed);
        if (has_rw_pointer(bitsValue)) return data()->ro();
        return (const class_ro_t *)(/* 验签后 */ bitsValue & FAST_DATA_MASK);
    }

    static uint32_t flags(uintptr_t bits) {   // flags 在 ro/rw 起始处，直接 strip+mask 读
        return *(const uint32_t *)((uintptr_t)ptrauth_strip((const uint32_t *)bits,
                 CLASS_DATA_BITS_RO_SIGNING_KEY) & FAST_DATA_MASK);
    }
    uint32_t flags() const { return flags(bits.load(std::memory_order_relaxed)); }
};
```

##### class_ro_t：编译期已经确定的内容

先看底下那层 `class_ro_t`——它就是编译期写死、运行期不可变的部分：

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260602110437370.png)


```objc
// objc-runtime-new.h:1598 —— class_ro_t（编译期只读）
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;       // §2 实例大小源头
#ifdef __LP64__
    uint32_t reserved;
#endif
    union {
        const uint8_t *ivarLayout;
        Class nonMetaclass;
        // 对实例类而言,这里存 `ivarLayout`——描述哪些 ivar 是 strong 引用的位图,供 ARC 在拷贝/释放对象时遍历强引用。对**元类**而言,ivar 布局无意义,于是这块空间被复用为 `nonMetaclass`,反向指回它所对应的实例类。
    };
    explicit_atomic<const char *> name;
    // baseMethods/baseProtocols 用 PointerUnion 包了相对/绝对 + ptrauth 多种表示
    objc::PointerUnion<method_list_t,   relative_list_list_t<method_list_t>,   ...> baseMethods;
    objc::PointerUnion<protocol_list_t, relative_list_list_t<protocol_list_t>, ...> baseProtocols;
    const ivar_list_t *ivars;
    const uint8_t *weakIvarLayout;
    objc::PointerUnion<property_list_t, relative_list_list_t<property_list_t>, ...> baseProperties;

    // RO_HAS_SWIFT_INITIALIZER 时才存在
    _objc_swiftMetadataInitializer _swiftMetadataInitializer_NEVER_USE[0];
    _objc_swiftMetadataInitializer swiftMetadataInitializer() const;
    const char *getName() const { return name.load(std::memory_order_acquire); }
    class_ro_t *duplicate() const;
};
```

`§2` 里反复提到的 `instanceSize`、`instanceStart`，源头就在这里。
`class_rw_t` 由 `objc_class::bits.data()` 取得,它内部再指向 `class_ro_t`。但这里有个关键时间点问题: 类在编译产物里**最初只有 `ro`**,`bits` 里一开始存的其实是 `ro` 指针而非 `rw`（注意：此时还**不能**调用 `data()`——它内部 `ASSERT(has_rw_pointer())` 会失败；要取只读数据得走 `safe_ro()`）。只有当这个类第一次被使用(消息发送、`+alloc` 等)触发 `realizeClassWithoutSwift` 时,runtime 才会分配一个 `class_rw_t`,把 `ro` 塞进去,并回写 `bits`。所以 `class_rw_t` 是"类被实现(realize)"的产物,而 `class_ro_t` 是"类被编译"的产物。

##### class_rw_t：realize 后出现的运行期外壳

因此，下面的 `class_rw_t` 不是和 `ro` 二选一地保存同一套数据。它更像 Runtime 为类加上的一层可写外壳：保留运行期状态和子类链，再通过 `ro_or_rw_ext` 接回原来的 `ro`，或者接到按需创建的 `rw_ext`。

```objc
// objc-runtime-new.h:2212 —— class_rw_t（运行期可写）
struct class_rw_t {
    uint32_t flags;
    uint16_t witness;
#if SUPPORT_INDEXED_ISA
    uint16_t index;
#endif
    explicit_atomic<uintptr_t> ro_or_rw_ext;   // 二选一：const class_ro_t* 或 class_rw_ext_t*
    Class firstSubclass;
    Class nextSiblingClass;

private:
    using ro_or_rw_ext_t = objc::PointerUnion<const class_ro_t, class_rw_ext_t,
                            PTRAUTH_STR("class_ro_t"), PTRAUTH_STR("class_rw_ext_t")>;
    const ro_or_rw_ext_t get_ro_or_rwe() const { return ro_or_rw_ext_t{ro_or_rw_ext}; }
    void set_ro_or_rwe(const class_ro_t *ro);
    void set_ro_or_rwe(class_rw_ext_t *rwe, const class_ro_t *ro);
    class_rw_ext_t *extAlloc(const class_ro_t *ro, bool deep = false);   // 真要改时才分配 rwe

public:
    void setFlags(uint32_t set);
    void clearFlags(uint32_t clear);
    void changeFlags(uint32_t set, uint32_t clear);

    class_rw_ext_t *ext() const;
    class_rw_ext_t *extAllocIfNeeded();
    class_rw_ext_t *deepCopy(const class_ro_t *ro);

    const class_ro_t *ro() const {        // rwe 在就从 rwe 取 ro，否则 ro_or_rw_ext 本身就是 ro
        auto v = get_ro_or_rwe();
        if (slowpath(v.is<class_rw_ext_t *>()))  //如果已经升级，那么从rwe里取ro
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->ro;
        return v.get<const class_ro_t *>(&ro_or_rw_ext);// 不然自己就是ro
    }
    void set_ro(const class_ro_t *ro);

    const method_array_t   methods() const;     // rwe 在取 rwe->methods；否则取 ro->baseMethods
    const property_array_t properties() const;
    const protocol_array_t protocols() const;
};
```

#### realizeClass：ro/rw 

前面讲 `bits` 时说过：类在编译产物里**最初只有 `ro`**，`bits` 里一开始存的就是 `ro` 指针；只有第一次被使用（消息发送、`+alloc` 等）才会被 **realize**，把这套数据真正「点亮」、并生出可写的 `rw`。这件事由 `realizeClassWithoutSwift` 完成（`objc-runtime-new.mm:2961`）：

```objc
// objc-runtime-new.mm:2978 / 2986-2992 / 3117 —— 正常类分支节选
auto ro = cls->safe_ro();                        // ① 取只读 ro（不是 data()）
rw = objc::zalloc<class_rw_t>();                 // ② 分配可写 rw
rw->set_ro(ro);                                  //    把 ro 收进 rw
rw->flags = RW_REALIZED | RW_REALIZING | isMeta; //    打上「已实现」标志
cls->setData(rw);                                //    ③ bits 从「指 ro」翻面成「指 rw」
...
supercls = realizeClassWithoutSwift(...);        //    递归 realize 父类 / 元类
cls->setSuperclass(supercls);                    //    回写 superclass
cls->initClassIsa(metacls);                      //    回写 isa（详见下文 isa 走位）
...
methodizeClass(cls, previously);                 // ④ 装方法/属性/协议、合并 category
```

拆成四步看：

1. **取 `ro`：走 `cls->safe_ro()`，不是 `data()`。** 此刻 `bits` 里压根还没有 `rw`，直接 `data()` 会撞上它内部的 `ASSERT(has_rw_pointer())`（这正是前面 §bits 留的伏笔）。有些老博客把第一步写成「`data()` 把 `rw` 强转成 `ro`」——那是更早版本的说法，951 取只读数据走的是 `safe_ro()`。
2. **分配 `rw` 并翻面。** `zalloc` 出一个 `class_rw_t`，`set_ro(ro)` 把只读数据收进去，打上 `RW_REALIZED` 标志，再 `setData(rw)` 回写——`bits` 从此**由「指向 `ro`」变成「指向 `rw`」**（低位仍留着那些 `FAST_` 标志）。
3. **接上继承链。** 递归把父类、元类也 realize，然后回写 `superclass` 和 `isa`。这一步就是「isa 走位图」的接环时刻，留到后面「接环就发生在 realize」那节细看，这里不展开。
4. **装方法：`methodizeClass`（`:3117`）。** 把 `ro` 里的 `baseMethods` / `baseProperties` / `baseProtocols` 安装好，并把外部 category 合并进来。具体到单个方法长什么样（`method_t` 的 `big` / `small` / `bigSigned` 三种表示，以及 `SEL` / `types` / `IMP`）、消息发送又怎么靠它找到 `IMP`，是 **Part 2** 的主线，这里按下不表。

这里先给 `method_t` 留一个小尾巴，本文暂时不展开这部分细节，Part 2 的方法查找会回到 `method_t`。


![new_class_layout_objc4_818.svg](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/new_class_layout_objc4_818.svg)


#### rw 的分离：clean memory vs dirty memory

- **Clean Memory** ：被加载后就不会再变化的内存。例如，`class_ro_t`就是 **Clean Memory** ，因为它是只读的。
 
- **Dirty Memory** ：在进程运行时会发生变化的内存。类结构体一旦被使用就是 **Dirty Memory** ，因为运行时会写入新的数据，例如它的方法缓存部分。

- **`class_ro_t` 通常位于 clean、只读的文件映射页**：编译期就定死，系统有内存压力时可以直接丢弃，之后再从文件重新加载；`instanceSize`、`name`、原始方法列表都在这里。

- **`class_rw_t` / `class_rw_ext_t` 是运行期分配并会被修改的数据**：类被 realize 后要维护子类链、合并 Category，动态修改方法、属性、协议时还会用到扩展结构。这些页会计入 dirty memory，不能像 clean 文件页那样直接丢弃后从文件恢复；iOS 还可能对部分匿名页进行内存压缩。

**Dirty Memory** 比 **Clean Memory** 代价更昂贵，因为在进程运行的整个过程中，都需要被保留； **Clean Memory** 则可以为其他事情腾出空间，因为当我们需要时，系统总是可以很容易地从磁盘中重新加载它。

macOS可以通过内存交换来解决内存不足的问题，但iOS不支持这个技术，所以 **Dirty Memory** 的代价会更昂贵。 **Dirty Memory** 就是为什么类结构被分为了这两个部分的原因。当然，如果我们可以拥有更多的 **Clean Memory** ，当然是更好的。把不会改变的数据分离出来，我们就可以让大部分的类数据保持为 **Clean Memory** 。

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802210804082.png)
一旦类被使用，运行时会分配额外的空间来存储这部分数据，即`class_rw_t`，其中 **rw表示read write** 。这个结构体中，我们只存储运行时产生的数据。

苹果在 WWDC2020 给出的设备测量中发现：系统里 `class_rw_t` 一度合计约占 30 MB，但真正需要修改方法等扩展信息的类只有约 10%。把少用字段拆进按需分配的 `class_rw_ext_t` 后，约 90% 的类不需要这份扩展数据，估算可在整个系统节省约 14 MB；现场对 Mail 的演示里，大约 9000 个 `class_rw_t` 中只有 900 多个需要扩展结构，单个进程约省 250 KB。

- **First Subclass** 和 **Next Sibling Class** 指针让运行时可以遍历当前使用的所有类。
- **Methods** 、 **Properties** 、 **Protocols** ，这部分也是可以在运行时进行修改的。在实践中发现，其实只有大约10%类的方法会发生变化，所以这部分内存可以得到优化，滕出一些空间。
- **Demangled Name** 只会被Swift类所使用，而且除非有需要获取它们的Objective-C名称，甚至都不会用到。

所以后两个不常用的部分，我们又可以拆分出来：

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802210822789.png)

这项拆分从 Apple 公开的 objc4-781 已经出现；本文使用的 objc4-951.1 延续并演进了这套设计，把可变方法、属性、协议数组等少用字段放进**懒分配**的 `class_rw_ext_t`：
![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802211647241.png)


```objc
// objc-runtime-new.h:2202 —— 真要动态改时才分配的扩展
struct class_rw_ext_t {
    class_ro_t_authed_ptr<const class_ro_t> ro;   // 指回只读 ro（带 ptrauth）
    method_array_t   methods;      // 可写方法数组（base + category + 动态添加）
    property_array_t properties;
    protocol_array_t protocols;
    const char *demangledName;
    uint32_t version;
};
```
如果原来的代码直接访问`class_rw_t`结构，由于结构内存布局发生了变化，可能产生崩溃。苹果推荐使用运行时API，这样底层的细节会由他们处理。

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802211815208.png)



上面的正文负责解释每一步为什么存在；下面这张交互图只用来**读完后复盘**三种状态，不需要再从头逐字段学习一次：

![[素材/rw_ro_ext_layout.html]]

#### 还有两块「不弄脏内存」的优化（同属 WWDC2020）

##### 相对方法列表（Relative Method Lists）

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802211918227.png)

每个方法包含3个部分的信息。

- 名称，或者选择器，例如`init`。
- 方法参数类型的编码，例如`@16@0:8`。
- 方法的IMP，Objective-C方法最终会编译为一个C函数。

这些信息都是指针，在64位的系统上会占用24字节。

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802211950582.png)
我们的方法列表是存在于镜像中的，而镜像的加载位置可能在内存的任何地方，这取决于动态链接器的选择。也就是说，链接器需要解析镜像中的指针，修复它们指向内存真实的的位置。**这部分会产生额外的消耗。
又由于镜像中的方法都是固定的，不会跑到其他镜像中去。其实我们不需要64位寻址的指针，只需要32位即可。
![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802212005354.png)

新版引入 `method_t::small`（`objc-runtime-new.h:975`），换成三个 **32 位相对偏移**（共 12 字节），体积直接减半。

更关键的是：相对偏移按「自身地址 + 偏移」现算，**镜像加载时不需要 rebase（重写指针）**，所以方法列表能继续待在 **clean memory**、随 dyld 共享缓存被多进程共享，更加安全，同时也不会因为 ASLR 重定位而被「弄脏」——和 `ro` 是同一招。承载这种表示的列表类型是 `relative_list_list_t`（`objc-runtime-new.h:1380`）。

> 代价：`int32` 偏移要求方法和它引用的目标落在 ±2GB 内；越界（极少数情况）才回退到绝对指针。

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802212849783.png)

如果这部分数据使用了 **Method Swizzling** 呢？

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802212938482.png)
苹果会在一个全局表中映射交换的实现。由于交换并不是非常常见的操作，所以这个全局表也不会特别大。

此外，在以前的实现中，进行方法交换会导致整个分页`Page`变成 **Dirty Memory** 。即仅仅一个交换，就可能造成数千字节的 **Dirty Memory** ，这是很不划算的。

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260802212956162.png)
如果我们的代码中直接处理了这些底层细节，但没有处理好的话，可能会造成1个64位的指针去读取2个32位的指针值。这是没有意义的，会造成崩溃。同样，苹果推荐使用运行时API，这样底层的细节会由他们处理。


##### 预优化缓存（回到 cache_t）

这一项已经不是 `bits → ro/rw/rw_ext` 的继续，而是回到前面的 `cache_t`。普通方法缓存要等消息第一次走慢速查找后，才把 `SEL → IMP` 写进当前进程的缓存；对于 dyld shared cache 中符合条件的系统类，shared cache builder 可以提前生成一份 `preopt_cache_t`，其中保存部分预构建的 IMP 缓存项。

类刚被 realize 时还必须让消息进入慢速路径，以完成 `+initialize` 的同步。因此 `initializeToEmptyOrPreoptimizedInDisguise()` 并不是直接在“空缓存”和“立即可用的预优化缓存”之间切换：没有可用预优化缓存时它初始化为空；有预优化缓存时，它先把原缓存地址**伪装保存**起来，使其暂时不能正常命中。等类初始化完成后，`maybeConvertToPreoptimized()` 才根据有效性决定正式启用它，或者丢弃并回到空缓存。

# 元类 metaclass

## 元类也是 `objc_class`

元类**没有**自己的结构体定义，它和普通类共用 `objc_class`，仅靠**标志位**区分自己是不是元类

```objc
// objc-runtime-new.h:3043 —— realize 之后判定
bool isMetaClass() const {
    ASSERT(isRealized());
#if FAST_CACHE_META
    return cache.getBit(FAST_CACHE_META);    // 真机：直接读 cache 的 META 标志位（快）
#else
    return data()->flags & RW_META;
#endif
}

// objc-runtime-new.h:3054 —— 未 realize 也能判定
bool isMetaClassMaybeUnrealized() const {
    // flags 在 class_ro_t 和 class_rw_t 的同一偏移（§bits 讲过），且 RO_META==RW_META，
    // 所以不管 bits 里现在是 ro 还是 rw，直接读 flags 都能拿到 META 位
    static_assert(offsetof(class_rw_t, flags) == offsetof(class_ro_t, flags), "flags alias");
    static_assert(RO_META == RW_META, "flags alias");
    if (isStubClass()) return false;
    return bits.flags() & RW_META;
}
```

这正好回收了 bits 里那个 `flags()` 的伏笔：**`flags` 之所以放在 `ro`/`rw` 的起始处、能直接 strip+mask 读出，就是为了在类还没 realize 时也能判断它是不是元类。**

## 类方法，其实就是元类的「实例方法」

这是元类存在的根本原因，源码给得非常直白：
```objc
// objc-class.mm:598 —— 取类方法 = 去「元类」里取同名实例方法
Method class_getClassMethod(Class cls, SEL sel)
{
    if (!cls || !sel) return nil;
    return class_getInstanceMethod(cls->getMeta(), sel);   // getMeta() 就是 cls->ISA()
}

// objc-runtime-new.h:3063 —— 类 ↔ 元类互导
Class getMeta() const {
    if (isMetaClassMaybeUnrealized()) return (Class)this;  // 自己是元类 → 返回自己
    else return this->ISA();                               // 普通类 → isa 指向的元类
}
```

也就是说：**「类对象」之于「元类」，等同于「实例对象」之于「类」**。实例的方法存在类里、类的方法（类方法）存在元类里——同一套机制套了两层。

# isa 走位与继承链

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260602162105369.png)


类的 `isa` 指向元类，元类的 `isa` 又指向谁？元类有没有父类？把这两条链走完，就是经典的「isa 走位图」。

## 接环就发生在 realize

类加载时，`isa`（指向元类）和 `superclass`（指向父类）这两根指针，是在 `realizeClassWithoutSwift` 里回写的：

```objc
// objc-runtime-new.mm:3019-3084 —— 节选
// 先把父类、元类也 realize（cls->ISA() 此刻就是编译期写好的元类）
supercls = realizeClassWithoutSwift(remapClass(cls->getSuperclass()), nil);
metacls  = realizeClassWithoutSwift(remapClass(cls->ISA()), nil);
...
cls->setSuperclass(supercls);   // superclass → 父类
cls->initClassIsa(metacls);     // isa        → 元类
```

而「根」的两条判定，把整张图的两个特殊点钉死了：

```objc
// objc-runtime-new.h:3077 / 3080
bool isRootClass()     const { return getSuperclass() == nil; }   // 根类：superclass 为 nil（NSObject）
bool isRootMetaclass() const { return ISA() == (Class)this; }     // 根元类：isa 指向自己（闭环）
```


### 实测：真实地址把这张图跑通

> 同一个 `isa_walk.m`，`Dog : Animal : NSObject` 三层继承，用 `object_getClass` / `class_getSuperclass` 逐层打印（地址为某次运行实测值）：

```
==== isa 链（object_getClass 逐层）====
instance d                  0x100d6f430
d.isa   = Dog 类            0x100d381a0
Dog.isa = Dog 元类          0x100d38178
Dog 元类.isa                0x1f6e35bb0   ← 跳到根元类
Animal 元类                 0x100d38128
NSObject 类                 0x1f6e35bd8
NSObject 元类(根元类)        0x1f6e35bb0
根元类.isa  →               0x1f6e35bb0   ← 等于自己，闭环 ✓

==== superclass 链 ====
Dog.superclass              0x100d38150   → Animal 类
Animal.superclass           0x1f6e35bd8   → NSObject 类
NSObject.superclass         0x0           → nil
Dog 元类.superclass         0x100d38128   → Animal 元类
Animal 元类.superclass      0x1f6e35bb0   → 根元类
根元类.superclass  →        0x1f6e35bd8   ← 等于 NSObject 类 ✓
```

对着地址核对两个闭环点：
1. **`根元类.isa (0x1f6e35bb0) == 根元类自身 (0x1f6e35bb0)`** —— isa 链到根元类就咬住自己，不再往上。
2. **`根元类.superclass (0x1f6e35bd8) == NSObject 类 (0x1f6e35bd8)`** —— 元类的 superclass 链不是断在根元类，而是拐回 `NSObject` 类，再由它 `superclass=nil` 收尾。

# Runtime 入院考试：四道题串起对象、类与元类

下面四道题出自 sunnyxx 的[《神经病院 objc runtime 入院考试》](https://blog.sunnyxx.com/2014/11/06/runtime-nuts/)。原文顺序依次是 `[self class] / [super class]`、`isKindOfClass / isMemberOfClass`、Category 中的 `+foo / -foo`、栈上伪造对象。本文仍按前面的知识递进来讲，所以保留“先类型判断、再 self/super”的顺序，但标题中的编号对应原题号。

这四道题不是四个孤立的技巧：第一、二、三题都依赖前面那张 **类—元类—根类** 关系图，第四题则回到文章开头的 **对象 = isa + ivar**。

## 入院题二：isKindOfClass / isMemberOfClass

`isKindOfClass:` / `isMemberOfClass:`，底层就是顺着 `superclass` 链、isa 和元类链在走。源码只有几行（`NSObject.mm:2450`）：

先把原题放在这里。假设 `Sark : NSObject`，下面四个结果分别是什么？

```objc
BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
```

答案是：`YES / NO / NO / NO`。这里特意强转成 `id`，是为了让编译器按实例消息的语法接受调用；但 receiver 实际仍是类对象，所以消息最终会落到下面的类方法版实现中。先记住答案，再沿源码逐个拆开。

```objc
+ (BOOL)isMemberOfClass:(Class)cls { return self->ISA() == cls; }   // 类方法版：比 isa（元类）
- (BOOL)isMemberOfClass:(Class)cls { return [self class] == cls; }  // 实例版：比 class

+ (BOOL)isKindOfClass:(Class)cls {                                  // 类方法版
    for (Class tcls = self->ISA(); tcls; tcls = tcls->getSuperclass())
        if (tcls == cls) return YES;
    return NO;
}
- (BOOL)isKindOfClass:(Class)cls {                                  // 实例版
    for (Class tcls = [self class]; tcls; tcls = tcls->getSuperclass())
        if (tcls == cls) return YES;
    return NO;
}
```

- `[self class]`：`self` 是**实例**时，拿到它的类。`Person *p` 的 `[p class]` 就是 `Person` 类。
- `self->ISA()`：类方法里的 `self` 是**类对象**，它的 `isa` 指向**元类**。所以 `Person` 调 `self->ISA()` 拿到的是 `Person 元类`，不是 `Person` 类——这就是 `+` 版和 `-` 版结果分叉的根。
- `getSuperclass()`：往上走一层父类。注意元类也有自己的继承链，和类的继承链平行：`Person 元类 → NSObject 元类 → NSObject 类`（根元类的 superclass 拐回根类）。

四个方法逐个看。

**① `- isMemberOfClass:`（实例版）** 就一次全等 `[self class] == cls`，不找父类。问的是「你是不是*正好*这个类的实例」。

- `[p isMemberOfClass:[Person class]]` → YES
- `[p isMemberOfClass:[NSObject class]]` → NO（p 的类是 Person，虽然继承自 NSObject，但不"正好"是它）

**② `- isKindOfClass:`（实例版）** 从 `[self class]` 起步，沿 `superclass` 往上爬（`Person → NSObject → nil`），任一层命中就 YES。问的是「你是不是这个类、或它子类的实例」。

- `[p isKindOfClass:[Person class]]` → YES（第一轮命中）
- `[p isKindOfClass:[NSObject class]]` → YES（爬到 NSObject 命中）

这两个实例版符合直觉，日常就用它们，没坑。坑在下面两个类方法版。

**③ `+ isMemberOfClass:`（类方法版）** `self->ISA() == cls`，比的是**元类**。`[Person isMemberOfClass:[Person class]]` → NO，因为左边是 Person 元类、右边是 Person 类。想让它 YES 得把元类传进去，可元类平时既拿不到也不会去传，所以这方法实际上几乎永远返回 NO。

**④ `+ isKindOfClass:`（类方法版）** 从 `self->ISA()`（元类）起步，沿**元类的 superclass 链**往上爬。对 Person 来说是：`Person 元类 → NSObject 元类 → NSObject 类 → nil`。注意倒数第二站是 **NSObject 类**——根元类的 superclass 指回根类。就是这条特殊连线，埋下了下面那个"灵异"结论。

最后补一句运行时的实情：`[obj isKindOfClass:]` 多半根本走不到上面这个方法。编译器有个快路径 `objc_opt_isKindOfClass`（`NSObject.mm:2185`）：

```objc
BOOL objc_opt_isKindOfClass(id obj, Class otherClass) {
    if (slowpath(!obj)) return NO;
    Class cls = obj->getIsa();
    if (fastpath(!cls->hasCustomCore())) {        // 没重写过 isKindOfClass 等 NSObject 核心方法
        for (Class tcls = cls; tcls; tcls = tcls->getSuperclass())
            if (tcls == otherClass) return YES;   // 直接 C 循环走 superclass 链，绕开 objc_msgSend
        return NO;
    }
    return ((BOOL(*)(id, SEL, Class))objc_msgSend)(obj, @selector(isKindOfClass:), otherClass);
}
```

只要这个类没重写过 NSObject 的核心方法（`hasCustomCore()` 为假），判定就被内联成一段裸 C 循环直接爬完 `superclass` 链，连消息都不发——和前面 cache 那套「热路径少绕一层」是一个意思。

## 入院题一：self 与 super——为什么 `[self class]` 和 `[super class]` 都打印 Son

再看一道老题。`Son` 继承 `Father`，在 `Son` 的 `init` 里打印两行：

```objc
@implementation Son : Father
- (id)init {
    self = [super init];
    if (self) {
        NSLog(@"%@", NSStringFromClass([self class]));   // 输出？
        NSLog(@"%@", NSStringFromClass([super class]));  // 输出？
    }
    return self;
}
@end
```

直觉容易答「Son、Father」。其实两行都是 `Son`。要讲清，得先分清 `self` 和 `super`。

- **`self` 是方法的隐藏参数。** 每个方法被调用时，编译器都偷偷塞了两个参数：`self`（消息接收者）和 `_cmd`（SEL）。`[self class]` 编译成 `objc_msgSend(self, @selector(class))`——`self` 既是接收者，也是方法查找的起点（从 `self` 的类开始找）。`objc_msgSend` 的完整机制是 Part 2 的主角，这里只用到「接收者 + 查找起点」这一层。
- **`super` 不是参数，是个编译器指示符。** 它告诉编译器：这条消息的方法查找**从父类开始**，但接收者**仍然是 `self`**。所以 `[super class]` 不走 `objc_msgSend`，走的是它的变体 `objc_msgSendSuper2`。

`objc_msgSendSuper2` 的第一个参数是个 `objc_super` 结构体（`message.h:34`）：

```objc
struct objc_super {
    id    receiver;     // 接收者——还是 self（son）
    Class super_class;  // 方法查找的起点
};
```

编译器为 `[super …]` 生成的是 `objc_msgSendSuper2`，不是公开头文件里那个 `objc_msgSendSuper`。两者复用同一个结构体，但第二个字段的含义不一样——`objc-abi.h:237` 写得很清楚：

```objc
// objc_msgSendSuper2() takes the current search class, not its superclass.
objc_msgSendSuper2(struct objc_super *super, SEL op, ...);
```

也就是说，编译器填进 `super_class` 的其实是**当前类 `Son`**，由 `objc_msgSendSuper2` 在运行时**自己取一次 `Son` 的 superclass（`Father`）**当查找起点。为什么不直接填父类？把「取 superclass」留到运行时做，才好配合继承关系的动态变化。

接下来是题眼。不管走 `objc_msgSend` 还是 `objc_msgSendSuper2`，**接收者 `receiver` 自始至终都是 `self`（这个 son 实例）**；`super` 改变的只是**从哪个类开始找 `class` 这个方法**：

- `[self class]`：从 `Son` 开始找 `class`。
- `[super class]`：从 `Father` 开始找 `class`。

但 `Son`、`Father` 都没重写 `class`，最终都落到 `NSObject` 那唯一一份实现（`NSObject.mm:2438`）：

```objc
- (Class)class { return object_getClass(self); }
```

它返回的是 `object_getClass(self)`，而这里的 `self` 就是传进来的 `receiver`，也就是 son 实例。所以两次调用，`object_getClass(self)` 拿到的都是 son 的类——`Son`。

总而言之：`super` 只换了**方法查找的起点**，没换 **receiver**；而 `class` 又只认 `object_getClass(self)`，所以 `[self class]` 和 `[super class]` 殊途同归，都打印 `Son`。

## 入院题三：Category 声明 `+foo`，却实现了 `-foo`

第三题故意让 Category 的声明和实现对不上：头文件声明的是类方法 `+foo`，实现里却写成实例方法 `-foo`。

```objc
@interface NSObject (Sark)
+ (void)foo;
@end

@implementation NSObject (Sark)
- (void)foo {
    NSLog(@"IMP: -[NSObject (Sark) foo]");
}
@end

[NSObject foo];
[[NSObject new] foo];
```

先说结果：这段代码可能收到“Category 没有实现已声明的 `+foo`”之类的编译警告，但仍能完成编译；运行时两次调用都能找到这里实现的 `-foo`，因此打印两次：

```text
IMP: -[NSObject (Sark) foo]
IMP: -[NSObject (Sark) foo]
```

第二行比较直白。`[[NSObject new] foo]` 的 receiver 是 NSObject 实例，实例方法查找从 `NSObject` 类对象开始，而 Category 实现的 `-foo` 正好被加到了 `NSObject` 的实例方法列表中：

```text
NSObject 实例 --isa--> NSObject 类 --找到 -foo--> IMP
```

第一行才是题眼。`[NSObject foo]` 的 receiver 是 NSObject 类对象，因此类方法查找从 `NSObject` 元类开始。元类里没有真正实现 `+foo`，但前面已经看到一个特殊连接：**根元类的 `superclass` 指回 `NSObject` 类**。于是查找还能从根元类继续走到 NSObject 类，并在那里撞见 Category 加入的 `-foo`：

```text
NSObject 类 --isa--> NSObject 元类（根元类）
                              |
                              | superclass
                              v
                         NSObject 类 --找到 -foo--> IMP
```

所以这里不是“`+foo` 自动变成了 `-foo`”，也不是声明决定了方法被放在哪里。真正决定存放位置的是 **Category 实现中的 `+` / `-`**：

- `+foo` 会进入元类的方法列表；
- `-foo` 会进入类对象的方法列表；
- 这道题之所以让 `[NSObject foo]` 也碰巧找到 `-foo`，靠的是根元类到根类那条特殊的 `superclass` 连线。

还要注意，这个现象不能随意推广成“任意类都能把实例方法当类方法调用”。例如在 `Sark` 上实现 `-foo` 后调用 `[Sark foo]`，查找会从 `Sark` 元类沿元类继承链向上走，并不会进入 `Sark` 类对象，自然也找不到 `Sark` 的 `-foo`。原题选择 `NSObject`，正是为了利用根类与根元类之间的特殊结构。

这是一道理解类与元类查找路径的实验题，不是可以在业务代码中利用的技巧。声明和实现故意不一致、Category 方法重名都会制造脆弱且难以维护的行为。

## 入院题四：伪造对象与内存地址

前三题已经把 `self/super`、类型判断以及根元类的特殊查找路径串起来了。最后一道更“野”，专门用来把“对象就是一段内存”这件事推到地址级别。

先看代码：

```objc
@interface Sark : NSObject
@property (nonatomic, copy) NSString *name;
- (void)speak;
@end

@implementation Sark
- (void)speak {
    NSLog(@"my name is %@", self.name);
}
@end

id cls = [Sark class];
void *obj = &cls;
[(__bridge id)obj speak];
```

直觉上，`obj` 只是一个栈变量地址，硬桥接成 `id` 再发消息，好像应该崩。但在这个实验里，它可以走到 `speak`，因为 `objc_msgSend` 只关心一件事：`receiver` 指向的那块内存开头，能不能被解释出一个可用的 `isa`。

这里 `cls` 变量里装的是 `[Sark class]`，也就是 Sark 类对象地址；`obj = &cls` 后，`obj` 指向这块栈内存。把 `obj` 当成 `id` 之后，Runtime 会从 `obj` 指向的位置读前 8 字节当作 `isa`。而那里正好放着 Sark 的 Class 地址，于是这块栈内存在 Runtime 看来就“像一个 Sark 实例”。

注意这里的 `self` 和 `_cmd` 仍然是真实存在的隐藏参数：

- `self`：这次消息的 receiver，也就是 `(__bridge id)obj`。
- `_cmd`：当前方法的 selector，也就是 `@selector(speak)`。

所以进入 `-speak` 后，`self` 不是一个正常堆对象，而是那块被伪装成对象的栈内存。

### 为什么打印的可能不是 Sark 的 name

如果 `Sark` 的第一个实例变量是 `name`，那么 `self.name` 本质上就是按 ivar offset 去读：

```objc
*(id *)((char *)self + offset(name))
```

对一个只有 `isa + name` 的简单对象来说，`name` 通常就在对象首地址后一个指针宽度的位置，也就是 `self + 8`。而这个实验里的 `self` 其实是 `&cls`，所以 `self + 8` 读到的不是正常对象里的 `name`，而是栈上紧挨着 `cls` 的下一格内容。

在一些关闭优化的调试场景里，当前函数栈上可能会依次放着这些东西：

1. `self` 隐藏参数
2. `_cmd` 隐藏参数
3. `[super viewDidLoad]` 相关的 `super_class`
4. `[super viewDidLoad]` 相关的 `receiver`
5. 局部变量 `obj`
6. 局部变量 `cls`

具体顺序会受编译器、优化级别和架构影响，不能当成语言规范。但这个实验想表达的点很明确：当你伪造对象时，ivar 访问不会“知道你是假的”，它仍然只做一件事：从 `self + offset` 位置取内存。

所以如果 `self + offset(name)` 恰好落到某个 `ViewController *` 局部值上，`self.name` 打印出来就可能是那个 ViewController；如果你在旁边插入一个字符串变量：

```objc
id cls = [Sark class];
NSString *myName = @"halfrost";
void *obj = &cls;
[(__bridge id)obj speak];
```

那么偏移位置上的内容可能就变成 `myName`，输出也随之变成这个字符串。这不是属性访问有什么特殊魔法，而是裸内存偏移刚好读到了不同的栈内容。

### 这道题真正说明什么

这道题把前面“对象 = isa + ivar”换了一个更底层的表述：

```text
id obj  指向一块内存
obj[0]  必须能解释成 isa / Class
ivar    = (char *)obj + offset
```

正常情况下，这块内存来自 `alloc`，在堆上，首 8 字节由 Runtime 写入合法 isa，后面按 `class_ro_t.ivars` 记录的 offset 摆放成员变量。这道题故意把栈变量地址伪装成对象地址，只是为了证明：Objective-C 对象访问在底层并不神秘，最终就是“从 receiver 指针出发，按固定偏移解释内存”。

LLDB 里可以用 `x` 命令验证这件事：

```text
(lldb) x/4gx obj
```

`x/nfu` 里，`n` 是显示几个单元，`f` 是格式，`u` 是每个单元的大小。比如 `g` 表示 8 字节，`x/4gx` 就是“按 16 进制打印 4 个 8 字节”。用它看 `obj` 附近的内存，就能直接观察第一格是不是 Sark Class，下一格又被 `self.name` 当成了什么。

这也是这道题最有价值的地方：它不是鼓励你写这种代码，而是逼你承认 Runtime 眼里的对象没有神秘外壳。只要一块内存开头能被解释成类，消息发送和 ivar 偏移就会按对象规则继续跑，至于这块内存是不是真的来自 `alloc`，那是另一个层面的安全问题。


# At Last

## 参考与感谢

本文在学习和整理 Objective-C Runtime 相关内容时，参考了以下优秀资料，在此表示感谢：

1. [Apple Developer Documentation - Objective-C Runtime](https://developer.apple.com/documentation/objectivec/objective-c-runtime)

2. [Apple Open Source - objc4 Runtime Source Code](https://opensource.apple.com/source/objc4/)

3. [WWDC 2020 - Advancements in the Objective-C runtime](https://developer.apple.com/videos/play/wwdc2020/10163/)

4. [掘金 - WWDC20 iOS14 Runtime 优化 1：Class 结构体变化](https://juejin.cn/post/6846687597478019079)

5. [Mike Ash - Friday Q&A 2009-03-20: Objective-C Messaging](https://www.mikeash.com/pyblog/friday-qa-2009-03-20-objective-c-messaging.html)

6. [Mike Ash - Friday Q&A 2017-06-30: Dissecting objc_msgSend on ARM64](https://www.mikeash.com/pyblog/friday-qa-2017-06-30-dissecting-objc_msgsend-on-arm64.html)

7. [Cocoa Samurai - Understanding the Objective-C Runtime](https://cocoasamurai.blogspot.com/2010/01/understanding-objective-c-runtime.html)

8. [Always Processing - Objective-C Internals](https://alwaysprocessing.blog/series/objc-internals)

9. [sunnyxx - 重识 Objective-C Runtime](https://blog.sunnyxx.com/2016/08/13/reunderstanding-runtime-0/)

10. [sunnyxx - 神经病院 objc runtime 入院考试](https://blog.sunnyxx.com/2014/11/06/runtime-nuts/)

11. [sunnyxx - objc category 的秘密](https://blog.sunnyxx.com/2014/03/05/objc_category_secret/)

12. [Draveness - 深入解析 ObjC 中方法的结构](https://draveness.me/method-struct/)

13. [WWDC 2013 Session 404 - Advances in Objective-C](https://nonstrict.eu/wwdcindex/wwdc2013/404/)

14. [Greg Parker - Classes and metaclasses](https://sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html)

15. [Draveness - 对象是如何初始化的（iOS）](https://draven.co/object-init/)

16. [Draveness - 你真的了解 load 方法么？](https://draveness.me/load/)

17. [Draveness - 关联对象 AssociatedObject 完全解析](https://draveness.me/ao/)

18. [BOB's Blog - Objective-C Runtime 相关优化与底层分析](https://blog.devtang.com/)

19. [bestswifter - 深入理解 Objective-C Runtime](https://github.com/bestswifter/blog/blob/master/articles/objc-runtime.md)

20. [Garan no dou - Objective-C 中的类和对象](https://blog.ibireme.com/)

21. [Tenloy's Blog - ObjC Runtime 总结](https://tenloy.github.io/)
22. 
23. https://juejin.cn/post/6846687597478019079?searchId=20260802142016822CE31D1CB65C4E568B
24. https://github.com/Biscoffee/apple-docs-vault/blob/agent/reader-navigation-r01/blogs/zh/cocoawithlove/what-is-a-meta-class-in-objective-c-cocoa-with-love.md
25. https://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/?utm_source=chatgpt.com

---

感谢以上作者和资料对 Objective-C Runtime、消息发送、isa、类与元类、方法缓存、Category、关联对象、`+load` 等内容的深入分析。

本文仅作为个人学习整理，若有理解不当之处，仍以 Apple 官方文档和 objc4 源码为准。

始于 2026.5.27
