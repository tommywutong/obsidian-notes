# OC 1.0
```Objective-c
typedef struct objc_class *Class;

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```
```Objective-C
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
    
#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars  //一级指针                           OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists    //二级指针                OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
    
} OBJC2_UNAVAILABLE;
```
在Objc2以前，class源码如上。

> ivars 是 objc_ivar_list 成员变量列表的指针；methodLists 是指向 objc_method_list 指针的指针。*methodLists 是指向方法列表的指针。运行时可以把新的方法列表指针挂到 methodLists 管理的列表里，这就是早期 Runtime 中 Category 附加方法的核心思路之一。Category 不能添加 ivar，则和对象内存布局不能在运行时随意改变有关。

一级指针的话，`methodLists` 直接指向一个方法列表，想新增方法只能重新分配内存、拷贝旧列表、追加新方法，代价很高。

二级指针的话，`methodLists` 指向的是**一个指针数组**，每个元素指向一个独立的方法列表：
```
methodLists ──→ [ *list1, *list2, *list3, ... ]
                    ↓       ↓       ↓
                  原始方法  Category Category
                  列表      A的方法   B的方法
```
加载一个 Category 时，Runtime 不需要改写原来的方法列表内容，而是把 Category 自己的方法列表作为一个新的列表指针挂进去。可以把它理解成：**新增一组方法列表入口，而不是把方法逐个拷贝进原列表**。

`ivars` 是一级指针，且不能像方法列表一样动态扩展，根本原因是：**对象的内存布局不能在类已经投入运行后随意改变**。对象实例需要多大、每个 ivar 位于哪里，都会影响对象创建、成员访问和父子类布局。如果运行时强行插入一个新 ivar，后面 ivar 的偏移量、对象大小、已经创建的实例都会受到影响。

所以 `ivars` 根本没有设计成可动态扩展的，不是技术上做不到，而是**做了也没用，内存布局不允许**。

至于“Category 不能添加属性”这句话，更准确地说应该是“Category 不能添加带存储的属性”。
`@property` 可以拆成几类信息：

1. 一组属性元数据
2. 一个 getter 方法声明
3. 一个 setter 方法声明（只读属性没有 setter）
4. 在普通类扩展或类实现中，编译器还可能自动合成 ivar 和 getter/setter 实现

Category 里声明 `@property` 没问题，它会产生属性元数据，也会让编译器知道有 getter/setter 这两个访问入口。但 Category 不能自动合成 ivar，也不会自动生成 getter/setter 实现。所以准确的说法是：**Category 可以添加属性声明/属性元数据，但不能添加真正的 ivar 存储；如果要让这个属性可用，必须自己实现 getter/setter，通常再配合关联对象存值**。
这也是为什么用 Associated Objects 能绕过去——它根本没有修改对象的内存布局，而是在对象外部维护了一张独立的哈希表来存值，只是"看起来"像属性而已。


# OC 2.0

2006 年，苹果在 WWDC 大会上发布了 [Objective-C 2.0](https://en.wikipedia.org/wiki/Objective-C#Objective-C_2.0)，做了一系列重要改动，包括早期的垃圾回收（现已废弃）、non-fragile ABI、runtime 性能优化等。这意味着上一节那段代码，以及所有带 `OBJC2_UNAVAILABLE` 标记的内容，都从 2006 年起永远告别了我们，只停留在历史文档里。

虽然 OC 1.0 的代码过时了，但理解它仍然有意义——尤其是 `methodLists` 那个二级指针的设计思想（外层管理多个方法列表，每个方法列表里再保存多个方法）在 OC 2.0 中以另一种形式延续了下来。

## OC 2.0 干的最关键的事：拆分 ro 和 rw

OC 2.0 把类的元数据拆成了两块：

```
Class
 └─ bits ─→ class_rw_t（运行时可写）
              ├─ ro ─→ class_ro_t（只读、编译期固定）
              │         ├─ baseMethods       ← 类自己写的方法
              │         ├─ ivars             ← 内存布局（不可改）
              │         ├─ baseProperties
              │         └─ baseProtocols
              ├─ methods       ← Category 方法附加到这里
              ├─ properties
              └─ protocols
```

- **`class_ro_t`**（read only）：编译期定死的东西，放在 Mach-O 的只读段。包含 `ivars`、`instanceSize`、`baseMethods` 等等。
- **`class_rw_t`**（read write）：运行时才创建，可写。Category 方法、`class_addMethod` 加的方法，全部进这里。

**为什么要这么拆？** 还是回到那条主线——

- ivar 影响内存布局，必须编译期定死 → 放 `ro`，只读，省内存又安全；
- 方法、协议、属性元数据不影响内存布局，运行期可以追加 → 放 `rw`，可写。

> WWDC 2020 之后，新版 objc4 进一步把 `class_rw_t` 中的可变数组外移到 `class_rw_ext_t`（rwe）里、按需分配，进一步省内存。具体细节我们留到后面 "目标类那一侧：class_rw_t 与 class_rw_ext_t" 一节展开。这里只要先记住一句抽象：**编译期固定的东西在 ro，运行期可附加的东西在 rw（或 rwe）管理的列表里**。

记住这张图，后面所有源码都是在这个结构上做手术。

---

# category简介

- **Category 用来给一个已经存在的类添加额外方法，而且不需要继承。**
- 可以把类的实现分开在几个不同的文件里面。这样做有几个显而易见的好处：a)可以减少单个文件的体积， b)可以把不同的功能组织到不同的category里， c)可以由多个开发者共同完成一个类， d)可以通过动态库或 bundle 间接影响某些 Category 是否被加载。
- 声明私有方法

还有广大开发者衍生出的其他使用场景
- 模拟多继承
- 把framework私有方法公开


Category（分类）看起来和 Extension（扩展）有点相似。Extension（扩展）有时候也被称为 **匿名分类**。但两者实质上是不同的东西。 Extension（扩展）是在编译阶段与该类同时编译的，是类的一部分。而且 Extension（扩展）中声明的方法只能在该类的 `@implementation` 中实现，这也就意味着，你无法对系统的类（例如 NSString 类）使用 Extension（扩展）。

而且和 Category（分类）不同的是，Extension（扩展）不但可以声明方法，还可以声明成员变量，这是 Category（分类）所做不到的。

---

# category 数据结构

所有的OC类和对象，在runtime层都是用struct表示的，category也不例外，在runtime层，category用结构体category_t（在objc-runtime-new.h中可以找到此定义），它包含了：

- 1)、类的名字（name）
- 2)、类（cls）
- 3)、category中所有给类添加的实例方法的列表（instanceMethods）
- 4)、category中所有添加的类方法的列表（classMethods）
- 5)、category实现的所有协议的列表（protocols）
- 6)、category中添加的所有属性（instanceProperties）

```Objective-C
struct category_t {
    const char *name;  // 名字
    classref_t cls;  // 要附加的目标类
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;  //添加的类方法列表，被附加到元类上
    struct protocol_list_t *protocols;//协议列表
    struct property_list_t *instanceProperties;//属性列表，但它不代表真的生成了_name成员变量，普通 Category 不能直接添加 ivar，因此在 Category 里只会生成 property metadata，不会自动生成存储空间。如果你想让这个属性真的能存值，需要自己实现 getter / setter，通常用关联对象：
};
```
> category中没有ivars，也就是没有成员变量列表，这就解释了为什么它可以添加属性声明/属性元数据，但是不能直接添加成员变量。因为对象的内存布局由类的 ivar list 决定。
一个对象创建时，它占多少内存、每个 ivar 在哪里，已经由类结构在 realization 后确定下来，运行期间添加成员变量会破坏原有内存布局。

Category 是后期附加元数据，不会改变对象内存大小

```Objective-C
struct method_list_t : entsize_list_tt<method_t, method_list_t, 0x3> {
    // 成员变量和方法
};

template <typename Element, typename List, uint32_t FlagMask>
struct entsize_list_tt {
    uint32_t entsizeAndFlags;
    uint32_t count;
    Element first;
};
```

这里的 `entsize_list_tt` 可以理解为一个容器，拥有自己的迭代器用于遍历所有元素。 `Element` 表示元素类型，`List` 用于指定容器类型，最后一个参数为标记位。

虽然这段代码实现比较复杂，但仍可了解到 `method_list_t` 是一个存储 `method_t` 类型元素的容器。`method_t` 结构体的定义如下:

```
struct method_t {
    SEL name;
    const char *types;   //  方法类型编码
    IMP imp;
};

```
当消息发送时，
```Objective-C
1. 从 obj 的 isa 找到类对象 MyClass
2. 先查方法缓存 cache
3. cache 没有，就查方法列表 method_list_t
4. 遍历 method_t
5. 找到 method_t.name == @selector(printName)
6. 拿到 method_t.imp
7. 调用 imp(obj, @selector(printName))
8. 可能把 selector -> imp 缓存起来
```
所以 `method_t` 就是消息发送最终要找的核心记录, 是Objective-C方法在Runtime的核心结构

最后，我们还有一个结构体 `category_list` 用来存储所有的 category，它的定义如下:

```Objective-C
struct locstamped_category_list_t {
    uint32_t count;
    locstamped_category_t list[0];
};
struct locstamped_category_t {
    category_t *cat;   // category本身
    struct header_info *hi;  // category来自的mach-o镜像信息
};
typedef locstamped_category_list_t category_list;
```

> `locstamped_category_t`：Category + 来源镜像
>`locstamped_category_list_t`/`category_list`：多个带来源信息的 Category

除了标记存储的 category 的数量外，`locstamped_category_list_t` 结构体还声明了一个长度为零的数组。这类写法常用于表达“结构体尾部跟着一段可变长度数据”；标准 C99 的 flexible array member 写法通常是 `list[]`，而 `list[0]` 更像是编译器扩展/历史写法。


可以这样理解：

- category_t 是一个具体 Category 的 runtime 本体；
- method_list_t 是 category_t 中保存实例方法/类方法的列表；
- method_list_t 继承自 entsize_list_tt，内部存放多个 method_t；
- method_t 是一个具体方法，包含 SEL、types、IMP；
- locstamped_category_t 是 category_t 的包装，额外记录它来自哪个 Mach-O 镜像；
- locstamped_category_list_t 是多个 locstamped_category_t 的列表；
- category_list 是 locstamped_category_list_t 的别名，runtime 用它保存某个类待附加的 Category 集合。

```Objective-C
category_list / locstamped_category_list_t
    └── locstamped_category_t
            ├── category_t *cat
            │       ├── method_list_t *instanceMethods
            │       │       └── entsize_list_tt<method_t, method_list_t, ...>
            │       │               └── method_t[]
            │       │                       ├── SEL name
            │       │                       ├── const char *types
            │       │                       └── IMP imp
            │       │
            │       ├── method_list_t *classMethods
            │       │       └── entsize_list_tt<method_t, method_list_t, ...>
            │       │               └── method_t[]
            │       │
            │       ├── protocol_list_t *protocols
            │       └── property_list_t *instanceProperties
            │
            └── header_info *hi
```
以上是 **Category 这一侧** 的数据结构。但 Category 加载本质上是"把 Category 的内容附加到目标类上"，所以在进加载流程之前，还需要看一下 **目标类那一侧** 的数据结构——也就是 `class_rw_t`。

---

# 目标类那一侧：class_rw_t 与 class_rw_ext_t

OC 2.0 那一节我们已经画过类结构图，知道 Category 加载的最终落点是 `class_rw_t.methods` / `properties` / `protocols` 这三个数组。这一节展开看这三个数组本身长什么样、Category 是怎么挂上去的，以及新版的 `class_rw_ext_t` 又是怎么回事。我们重点看 `methods`，`properties` 和 `protocols` 是同一套机制。

## class_rw_t 的早期定义

`bits` 中包含一个指向 `class_rw_t` 结构体的指针。早期版本的定义如下：

```Objective-C
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
};
```

在 Objective-C 2.0 的 non-fragile ABI 中，类的实例布局信息保存在 `class_ro_t` 中，例如 `instanceStart` 和 `instanceSize` 分别描述本类实例变量的起始位置和对象实例总大小。更重要的是，ivar 的偏移量不再被编译期完全写死，而是通过 `ivar_list_t` 中的 offset 指针等运行时元数据进行间接访问。这样当父类新增实例变量导致实例大小变化时，runtime 可以在类 realization 阶段重新调整子类 ivar 的偏移量，从而避免子类必须重新编译。这就是 Objective-C 2.0 ABI 稳定性的一部分。

但这并不意味着 Category 可以在运行时给一个已经存在的类插入 ivar。non-fragile ABI 解决的是"父类布局变化时，子类不必重新编译"的问题；Category 加载发生在类的运行时元数据附加阶段，它只能附加方法、属性元数据和协议，不能改变对象实例大小。

## method_array_t：可扩展的"方法列表的列表"

`class_rw_t.methods` 的类型是 `method_array_t`，继承自 `list_array_tt`。它是一个用于管理若干个 list 的容器：对 `methods` 来说，它管理的是多个 `method_list_t`，每个 `method_list_t` 里面再存多个 `method_t`。从逻辑上可以类比为二维数组。

```Objective-C
template <typename Element, typename List>
class list_array_tt {
    struct array_t {
        uint32_t count;
        List* lists[0];
    };

    // 内部用一个指针/位标记来表示 empty / single list / array
};

class method_array_t : public list_array_tt<method_t, method_list_t>;
```

`Element` 是元数据的类型（如 `method_t`），`List` 是用于存储元数据的一维数组（如 `method_list_t`）。

`list_array_tt` 有三种状态：

1. **空**：没有任何 `method_list_t`，类比为 `[]`；
2. **单列表**：直接指向一个 `method_list_t`，类比为 `[[m1, m2]]`；
3. **多列表**：指向一个 `array_t`，里面保存多个 `method_list_t*`，类比为 `[[m1, m2], [m3, m4]]`。

类刚 realize 后，通常会把 `class_ro_t.baseMethods` 放进 `class_rw_t.methods`。如果类本身没有方法，可能为空；如果有方法，可能是单列表状态。当 runtime 后续通过 Category、`class_addMethod` 等方式追加新的方法列表时，`methods` 通常会从空 / 单列表形态扩展为多列表形态。

这一节最重要的事实：**Category 加载时，会把 Category 自己的 `method_list_t` 整体作为一个"指针"挂到 `method_array_t` 的最前面，而不是把方法逐个拷贝进原列表**。这正好对应文章开头讲 OC 1.0 时的二级指针思想。

## attachLists：把新列表插到最前面

附加的核心函数是 `attachLists`：

```Objective-C
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;
    uint32_t oldCount = array()->count;
    uint32_t newCount = oldCount + addedCount;
    setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
    array()->count = newCount;
    // 把旧列表整体往后挪 addedCount 格
    memmove(array()->lists + addedCount, array()->lists,
            oldCount * sizeof(array()->lists[0]));
    // 把新列表写到最前面
    memcpy(array()->lists, addedLists,
           addedCount * sizeof(array()->lists[0]));
}
```

逻辑很直白：`realloc` 扩容 → `memmove` 把旧列表整体往后挪 → `memcpy` 把新列表写到前面。**新列表永远排在前面**——这是后面"覆盖"假象的物理基础。

实际代码会更复杂一些。当 `list_array_tt` 处于"单列表"状态时，并不需要在原地扩容空间，而是只要重新申请一块内存，并将最后一个位置留给原来的集合即可。这样只多花费一点空间，但减少了内存拷贝，是一个空间换时间的权衡。无论走哪条路径，**参数列表都会被添加到二维数组的前面**这一事实不变。

## 新版的变化：class_rw_ext_t

WWDC 2020 之后（对应 objc4-818 及以后的版本），苹果对这一块做了进一步优化：把 `class_rw_t` 中的 `methods` / `properties` / `protocols` 三个数组**外移**到一个新结构 `class_rw_ext_t`（简称 **rwe**）里，**按需分配**。

理由很现实——**绝大多数类一辈子都不会有 Category、不会被 `class_addMethod` 修改**。给每个类都分配一个完整的、可写的 `class_rw_t`（包含三个数组）是浪费，因为这块属于 dirty memory，内存代价很高。新版变成：

```
class_rw_t.ro_or_rw_ext  （一个 union 风格的指针）
        ├─ 平时        → 直接指向 class_ro_t（只读，可共享 page）
        └─ 类被动态修改后 → 指向 class_rw_ext_t（包含 methods/properties/protocols）
```

触发 rwe 创建的关键函数是 `extAllocIfNeeded`：

```Objective-C
class_rw_ext_t *extAllocIfNeeded() {
    auto v = get_ro_or_rwe();
    if (fastpath(v.is<class_rw_ext_t *>())) {
        return v.get<class_rw_ext_t *>(...);   // 已经有了，直接返回
    } else {
        return extAlloc(v.get<const class_ro_t *>(...));  // 没有，现在创建
    }
}
```

谁会触发？**附加 Category、`class_addMethod`、`class_addProtocol` 这些动态修改类的操作**。所以一个类只要"被 Category 附加过一次"，rwe 就会被分配出来。

但要意识到一点：**rwe 是性能优化，不是机制变化**。从理解 Category 的角度，仍然可以把整个模型抽象成"编译期固定信息在 ro，运行期可附加的方法 / 属性 / 协议在 rw（或 rw_ext）管理的列表里"。具体源码对照见下一节。

---

# category如何加载

dyld 加载的流程大致是这样：

1. 配置环境变量；
2. 加载共享缓存；
3. 初始化主 APP；
4. 插入动态缓存库；
5. 链接主程序；
6. 链接插入的动态库；
7. 初始化主程序：OC, C++ 全局变量初始化；
8. 返回主程序入口函数。

> 初始化主程序中，Runtime 初始化的大致调用栈如下：

>`dyldbootstrap::start` ---> `dyld::_main` ---> `initializeMainExecutable` ---> `runInitializers` ---> `recursiveInitialization` ---> `doInitialization` ---> `doModInitFunctions` ---> `_objc_init`

在 `_objc_init` 这一步中，`Runtime` 会向 `dyld` 注册回调。之后当 `image` 加载到内存后，`dyld` 会通知 `Runtime` 进行处理：`map_images` 负责解析和注册 image 中的 ObjC 元数据；`_read_images` / `load_categories_nolock` 会读取 Category 列表，并把 Category 的内容附加到目标类或元类上；随后 `load_images` 中会调用 `call_load_methods`，按规则调用 `Class` 和 `Category` 的 `+load` 方法。

>加载Category（分类）的调用栈如下：

> `_objc_init` ---> 注册 dyld 回调 ---> `map_images` / `map_images_nolock` ---> `_read_images` 或 `load_categories_nolock`（读取并附加分类） ---> `load_images`（调用 `+load`）。

**一些说明：**

不同版本的 objc4 在函数拆分和命名上会有差异。旧版源码(可在早期开源版本中找到)用 `addUnattachedCategoryForClass` + `remethodizeClass` + `attachCategories` 三个函数清晰地划分了"登记 → 取出 → 合并"三个步骤。新版源码做了路径优化:对已经 realized 的类直接调用 `attachCategories` 立即合并,只有未 realized 的类才会先放入 unattached category 表,等到类 realization 时再统一附加。

但要意识到,**新旧版的差异是性能优化,不是机制变化**——核心思路都是"延迟合并":Category 加载时如果目标类还没准备好,就先存起来,等准备好了再合。本文下面保留旧版函数名来讲解,因为它把这个思路展示得最完整。理解了旧版,新版只是少了一次绕路。



忽略 `_read_images` 方法中其他与本文无关的代码，得到如下代码：
```Objective-C
// 从当前 Mach-O 镜像 hi 中获取 Category 列表。
// catlist 指向当前镜像中的 category_t 指针数组。
// count 表示当前镜像中 Category 的数量。
category_t **catlist = _getObjc2CategoryList(hi, &count);

// 判断当前镜像是否包含 Category 的 class property 元数据。
// 新版 runtime 中会用这个标志决定是否读取 cat->_classProperties。
bool hasClassProperties = hi->info()->hasCategoryClassProperties();


// 遍历当前镜像中的所有 Category。
for (i = 0; i < count; i++) {

    // 取出一个 Category。
    category_t *cat = catlist[i];

    // 根据 Category 中记录的目标类引用，找到 runtime 中真正的 Class。
    // remapClass 用于处理类重映射、weak-linked class 等情况。
    Class cls = remapClass(cat->cls);

    // 如果目标类不存在，说明这个 Category 无法附加。
    // 常见情况：目标类是 weak-linked，但当前系统中不存在。
    if (!cls) {
        catlist[i] = NULL;
        continue;
    }


    // ============================================================
    // 一、处理 Category 的“实例侧内容”
    //
    // 包括：
    // 1. 实例方法 instanceMethods
    // 2. 协议 protocols
    // 3. 实例属性元数据 instanceProperties
    //
    // 这些内容会附加到普通类对象 cls 上。
    // ============================================================

    bool classExists = NO;

    if (cat->instanceMethods ||
        cat->protocols ||
        cat->instanceProperties)
    {
    // 先把 Category 登记到目标类 cls 对应的待附加列表中。
        //
        // 注意：
        // addUnattachedCategoryForClass 并不是真正合并方法列表。
        // 它只是建立：
        //
        // cls -> category
        //
        // 的关联关系。
        addUnattachedCategoryForClass(cat, cls, hi);

        // 如果目标类已经 realized，
        // 说明它已经完成 runtime 初始化，可以立刻合并 Category 内容。
        if (cls->isRealized()) {

            // 真正把 Category 的实例方法、属性、协议等附加到 cls 上。
            remethodizeClass(cls);

            classExists = YES;
        }
    }


    // ============================================================
    // 二、处理 Category 的“类侧内容”
    //
    // 包括：
    // 1. 类方法 classMethods
    // 2. 协议 protocols
    // 3. 类属性元数据 _classProperties
    //
    // 这些内容会附加到目标类的元类 cls->ISA() 上。
    // ============================================================

    if (cat->classMethods ||
        cat->protocols ||
        (hasClassProperties && cat->_classProperties))
    {
        // 类方法本质上是元类的实例方法，
        // 所以 Category 的类方法要登记到 cls->ISA() 上，
        // 也就是目标类的元类。
        addUnattachedCategoryForClass(cat, cls->ISA(), hi);

        // 如果元类已经 realized，
        // 就立刻把 Category 的类方法、类属性元数据、协议等附加进去。
        if (cls->ISA()->isRealized()) {
            remethodizeClass(cls->ISA());
        }
    }
}
```
这段伪代码主要用到了两个方法：

- `addUnattachedCategoryForClass(cat, cls, hi);` 把分类先登记到目标类的待附加列表里
- `remethodizeClass(cls);` 触发待附加分类真正合并到类的方法、属性、协议列表中

通过这两个方法达到了两个目的：

1. 把 `Category（分类）` 的对象方法、协议、属性添加到类上；
2. 把 `Category（分类）` 的类方法、协议、类属性元数据添加到类的 `metaclass` 上。

下面来说说上边提到的这两个方法。

```Objective-C
// 将一个 Category 暂存到“目标类 cls 的待附加 Category 列表”中。
// 注意：这个函数并不会真正把 Category 的方法、属性、协议合并到类上。
// 真正合并发生在后面的 remethodizeClass(cls) 中。
static void addUnattachedCategoryForClass(category_t *cat,
                                          Class cls,
                                          header_info *catHeader)
{
    // 当前必须已经持有 runtimeLock。
    // 因为这里要修改 runtime 的全局 Category 映射表。
    runtimeLock.assertLocked();

    // 获取全局的“未附加 Category 映射表”。
    //
    // 这个表可以理解为：
    //
    //     Class -> category_list
    //
    // 也就是：
    //
    //     某个类 -> 这个类当前等待附加的 Category 列表
    //
    NXMapTable *cats = unattachedCategories();

    category_list *list;

    // 根据目标类 cls，从映射表中取出它已有的待附加 Category 列表。
    list = (category_list *)NXMapGet(cats, cls);

    if (!list) {
        // 如果这个类还没有待附加 Category 列表，
        // 就创建一个新的 category_list。
        //
        // sizeof(*list)：
        //     category_list 结构体头部大小，里面包含 count。
        //
        // sizeof(list->list[0])：
        //     一个 locstamped_category_t 的大小。
        //
        // 因为 list 里使用了 list[0] 这种零长度数组技巧，
        // 所以创建第一个元素时，需要额外申请一个元素的空间。
        list = (category_list *)calloc(
            sizeof(*list) + sizeof(list->list[0]),
            1
        );
    } else {
        // 如果这个类已经有待附加 Category 列表，
        // 就扩容，多申请一个 locstamped_category_t 的空间。
        //
        // 原来能存 list->count 个 Category，
        // 现在要存 list->count + 1 个。
        list = (category_list *)realloc(
            list,
            sizeof(*list) + sizeof(list->list[0]) * (list->count + 1)
        );
    }

    // 把当前 Category 和它所在的 Mach-O 镜像信息一起存入列表。
    //
    // locstamped_category_t 包含：
    //     cat       : Category 本体
    //     catHeader : Category 来自哪个 Mach-O 镜像
    //
    // list->count++：
    //     先把元素写入当前位置，再把 count 加 1。
    list->list[list->count++] = (locstamped_category_t){ cat, catHeader };

    // 把更新后的 list 重新插入全局映射表。
    //
    // 如果之前 cats 中已经有 cls 对应的旧 list，
    // 这里会用扩容后的新 list 覆盖旧记录。
    NXMapInsert(cats, cls, list);
}
```
`addUnattachedCategoryForClass(cat, cls, hi);` 的执行过程可以参考代码注释。执行完这个方法之后，系统会将当前分类 `cat` 放到该类 `cls` 对应的未依附分类 `unattached` 的列表 `list` 中。这句话有点拗口，简而言之，就是：**先把类和分类做一个关联映射，真正合并稍后发生**。

实际上真正起到添加加载作用的是下边的 `remethodizeClass(cls);` 方法。

```Objective-C
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    // 当前必须已经持有 runtimeLock。
    // 因为这里会读取并修改 runtime 中类的全局结构。
    runtimeLock.assertLocked();

    // 判断当前传入的 cls 是普通类，还是元类。
    //
    // 如果是普通类：
    //     处理 Category 的实例方法、实例属性、协议等。
    //
    // 如果是元类：
    //     处理 Category 的类方法、类属性、协议等。
    isMeta = cls->isMetaClass();

    // 从全局 unattachedCategories 表中取出 cls 对应的待附加 Category 列表。
    //
    // false 表示当前不是在 realizing class 的流程中调用。
    //
    // 返回的 cats 可以理解为：
    //
    //     cls -> [CategoryA, CategoryB, CategoryC]
    //
    // 这些 Category 之前是通过 addUnattachedCategoryForClass(cat, cls, hi)
    // 暂存进去的。
    cats = unattachedCategoriesForClass(cls, false /* not realizing */);

    if (cats) {
        // 真正把 cats 中的 Category 附加到 cls 上。
        //
        // attachCategories 会处理：
        //     1. 方法列表 methods
        //     2. 属性列表 properties
        //     3. 协议列表 protocols
        //
        // 如果 cls 是普通类：
        //     Category 的实例方法会被附加到 cls。
        //
        // 如果 cls 是元类：
        //     Category 的类方法会被附加到 cls。
        //
        // true 表示附加完成后需要刷新方法缓存。
        attachCategories(cls, cats, true /* flush caches */);

        // cats 是从 unattachedCategoriesForClass 取出的临时列表。
        // attach 完成后释放。
        free(cats);
    }
}
```

`remethodizeClass(cls);` 方法主要就做了一件事：调用 `attachCategories(cls, cats, true);` 方法将未依附分类的列表 cats 附加到 cls 类上。所以，我们就再来看看 `attachCategories(cls, cats, true);` 方法。

```Objective-C
static void attachCategories(Class cls,
                             category_list *cats,
                             bool flush_caches)
{
    if (!cats) return;

    if (PrintReplacedMethods) {
        printReplacements(cls, cats);
    }

    // 判断当前 cls 是普通类还是元类。
    // 普通类：处理 Category 的实例方法、实例属性、协议。
    // 元类：处理 Category 的类方法、类属性、协议。
    bool isMeta = cls->isMetaClass();

    // 创建三个临时数组，用来收集所有 Category 中的方法列表、属性列表、协议列表。
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));

    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));

    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // 记录实际收集到的方法列表、属性列表、协议列表数量。
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;

    // 从后往前遍历 Category。
    // 这样可以保证“最新加载的 Category”排在前面。
    int i = cats->count;

    // 标记这些 Category 是否来自 bundle。
    bool fromBundle = NO;

    while (i--) {
        // 取出当前 Category 以及它所在的 Mach-O 镜像信息。
        auto& entry = cats->list[i];

        // 取出当前 Category 的方法列表。
        //
        // 如果 cls 是普通类：
        //     methodsForMeta(false) 取 instanceMethods。
        //
        // 如果 cls 是元类：
        //     methodsForMeta(true) 取 classMethods。
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);

        if (mlist) {
            // 将当前 Category 的方法列表保存到临时方法列表数组中。
            mlists[mcount++] = mlist;

            // 记录是否有来自 bundle 的 Category。
            fromBundle |= entry.hi->isBundle();
        }

        // 取出当前 Category 的属性列表。
        //
        // 如果 cls 是普通类：
        //     通常取 instanceProperties。
        //
        // 如果 cls 是元类：
        //     可能取 classProperties；
        //     如果当前镜像不支持 classProperties，则可能返回 nil。
        property_list_t *proplist =
            entry.cat->propertiesForMeta(isMeta, entry.hi);

        if (proplist) {
            proplists[propcount++] = proplist;
        }

        // 取出当前 Category 声明遵守的协议列表。
        protocol_list_t *protolist = entry.cat->protocols;

        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    // 取出当前类的运行时可变数据 class_rw_t。
    auto rw = cls->data();

    // 对即将附加的方法列表做准备工作。
    //
    // 例如：
    //     排序
    //     修正方法列表
    //     检查特殊方法
    //     标记是否来自 bundle
    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);

    // 将 Category 的方法列表附加到 class_rw_t 的 methods 中。
    //
    // 注意：
    //     attachLists 会把新列表放到旧列表前面。
    //     所以 Category 的同名方法会优先于原类方法被查找到。
    rw->methods.attachLists(mlists, mcount);

    // 临时数组已经被 attachLists 使用完，可以释放。
    free(mlists);

    // 如果需要刷新缓存，并且确实附加了方法列表，就清理方法缓存。
    //
    // 因为 Category 可能添加或“覆盖”同名方法，
    // 如果不清缓存，objc_msgSend 可能继续命中旧 IMP。
    if (flush_caches && mcount > 0) {
        flushCaches(cls);
    }

    // 将 Category 的属性列表附加到 class_rw_t 的 properties 中。
    rw->properties.attachLists(proplists, propcount);

    // 释放属性临时数组。
    free(proplists);

    // 将 Category 的协议列表附加到 class_rw_t 的 protocols 中。
    rw->protocols.attachLists(protolists, protocount);

    // 释放协议临时数组。
    free(protolists);
}
```

`attachCategories` 干的事可以总结成三步：

1. **从后往前遍历** Category，把它们的方法列表收集到一个临时数组 `mlists`；
2. 一次性调用 `rw->methods.attachLists(mlists, mcount)`，把这些列表整体插到 `class_rw_t.methods` 的最前面（参考前面 `class_rw_t` 那一节里讲过的 `attachLists`）；
3. 如果加了方法，**清掉方法缓存**，否则 `objc_msgSend` 可能继续命中旧 IMP。

注意第一步的"从后往前"——这一步加上第二步的"插到前面"，叠加效果就是：**最后登记的 Category，它的方法会出现在 `class_rw_t.methods` 数组的最前面**。这是后面"覆盖"假象的关键。

## 新版源码对照

上面的代码是早期 objc4 的实现。WWDC 2020 之后（objc4-818 及以后），同样这件事的源码组织有了几处变化，但**核心机制完全没变**。整理一下对照表：

| 维度 | 旧版（早期 objc4） | 新版（objc4-818 起） |
|---|---|---|
| 登记函数 | `addUnattachedCategoryForClass(cat, cls, hi)` | `objc::unattachedCategories.addForClass(...)`（C++ 命名空间风格） |
| 触发函数 | `remethodizeClass(cls)` | 被 `attachToClass(cls, previously, flags)` 取代，最终也是调到 `attachCategories` |
| 合并函数 | `attachCategories(cls, cats, flush)` | `attachCategories(cls, cats_list, cats_count, flags)`（签名变了） |
| 方法挂载点 | `rw->methods.attachLists(...)` | `rwe->methods.attachLists(...)`，且 rwe 通过 `extAllocIfNeeded()` 按需创建 |
| 收集临时数组 | `malloc(cats->count * sizeof(...))` | 栈上固定 64 槽位 `ATTACH_BUFSIZ`，满了就 flush 一次 |

新版的 `attachCategories` 大致长这样：

```Objective-C
static void attachCategories(Class cls,
                             const locstamped_category_t *cats_list,
                             uint32_t cats_count,
                             int flags)
{
    constexpr uint32_t ATTACH_BUFSIZ = 64;
    method_list_t   *mlists[ATTACH_BUFSIZ];      // 栈上,固定 64 个,避免 malloc
    property_list_t *proplists[ATTACH_BUFSIZ];
    protocol_list_t *protolists[ATTACH_BUFSIZ];

    uint32_t mcount = 0, propcount = 0, protocount = 0;
    bool fromBundle = NO;
    bool isMeta = (flags & ATTACH_METACLASS);

    // 关键:按需分配 rwe(只有真要附加内容时才创建)
    auto rwe = cls->data()->extAllocIfNeeded();

    for (uint32_t i = 0; i < cats_count; i++) {
        auto& entry = cats_list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            // 栈缓冲区满了就先 flush 一次
            if (mcount == ATTACH_BUFSIZ) {
                prepareMethodLists(cls, mlists, mcount, NO, fromBundle, __func__);
                rwe->methods.attachLists(mlists, mcount);
                mcount = 0;
            }
            // 倒序填充:让最终顺序一次 attachLists 就是对的
            mlists[ATTACH_BUFSIZ - ++mcount] = mlist;
            fromBundle |= entry.hi->isBundle();
        }
        // properties / protocols 同理...
    }

    // 处理剩余的(不满 64 个)
    if (mcount > 0) {
        prepareMethodLists(cls, mlists + ATTACH_BUFSIZ - mcount,
                           mcount, NO, fromBundle, __func__);
        rwe->methods.attachLists(mlists + ATTACH_BUFSIZ - mcount, mcount);
        if (flags & ATTACH_EXISTING) flushCaches(cls);
    }
    // 同样处理 properties / protocols...
}
```

几个变化点的解读：

1. **`extAllocIfNeeded()`**：旧版直接 `cls->data()` 拿到 `class_rw_t`，三个数组就在里面。新版要走 `extAllocIfNeeded()` 才能拿到 rwe——如果这个类还没被动态修改过，rwe 不存在，这一步会创建它。**这就是 WWDC 2020 提到的"按需分配，省 dirty memory"**。
2. **栈缓冲区 + 分块 flush**：旧版 `malloc(cats->count * ...)`，一次性收集所有方法列表。新版用 64 个槽位的栈数组，满了就 flush 一次，避免堆分配——绝大多数类的 Category 数量远小于 64，所以这是常见路径。
3. **倒序填充 `mlists[ATTACH_BUFSIZ - ++mcount] = mlist`**：旧版 `while (i--)` 倒序遍历得到"最新 Category 在前"；新版正序遍历，但是用倒序下标填充缓冲区，**最终效果一样——最新 Category 在 `mlists` 数组的前面**。

总之，新旧版做的还是同一件事：

- 收集所有 Category 的方法列表 →
- `attachLists` 把它们整体插到目标类方法数组的最前面 →
- 清缓存。

下面所有讨论的"覆盖"行为、`+load` 特性、Category 不能加 ivar 等结论，**新旧版完全一致**——它们的根都在更深的设计层（内存布局、消息发送），不在这些函数命名上。

## "覆盖"的论证：消息查找顺序

`objc_msgSend` 查找方法的逻辑（简化版）：

```Objective-C
static method_t *getMethodNoSuper_nolock(Class cls, SEL sel) {
    for (auto mlists = cls->data()->methods.beginLists(),
              end    = cls->data()->methods.endLists();
         mlists != end; ++mlists) {
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;        // 找到就返回,不再继续
    }
    return nil;
}

static method_t *search_method_list(const method_list_t *mlist, SEL sel) {
    for (auto& meth : *mlist) {
        if (meth.name == sel) return &meth;
    }
}
```

**从前往后查，第一个命中就返回**。配上 `attachLists` "新列表在前" 的事实：

> Category 中的同名方法不会**替换**类中原有的方法，但消息发送时**会优先命中** Category 的版本。原方法依然在 `class_rw_t.methods`（或 `rwe->methods`）里，只是被排到了后面，永远轮不到它。

这就是"覆盖"假象的全部秘密。

## Category 加载流程总览

把上面的源码细节收起来，Category 的加载流程可以理解成下面这条链路：

```
1. 编译器把 Category 编译成 category_t
2. category_t 被放进 Mach-O 的 __objc_catlist
3. runtime 在 _read_images 中读取 __objc_catlist
4. 得到 category_t **catlist
5. 遍历每个 category_t
6. 找到它的目标类 cls
7. 调用 addUnattachedCategoryForClass(cat, cls, hi)
8. 把 category_t 和 header_info 包装成 locstamped_category_t
9. 放入 cls 对应的 category_list
10. 如果 cls 已经 realized，调用 remethodizeClass(cls)
11. remethodizeClass 取出 category_list
12. attachCategories 遍历 category_list
13. 从每个 locstamped_category_t 里取出 category_t
14. 从 category_t 里取出 method_list_t
15. 把 method_list_t attach 到 class_rw_t.methods
```

这里最关键的点有三个：

1. Category 不是把方法“拷贝进类原来的方法列表”，而是把自己的 `method_list_t` 挂到类的 `method_array_t` 前面。
2. Category 的实例方法附加到类对象；Category 的类方法附加到元类，因为类方法本质上是元类的实例方法。
3. Category 中的同名方法并没有真正删除或替换原方法，只是在普通消息查找顺序上排在更前面，所以表现为“覆盖”。


# Category 和 Class 的 +load 方法

Category 的方法、属性元数据、协议附加到类上的操作，发生在 `+load` **之前**。换句话说，`+load` 跑起来时，类已经是合并完所有 Category 之后的样子了。

## 调用顺序

1. **主类 `+load` 先调**：按继承关系从父类到子类，同一层级内再受编译/链接顺序影响；
2. **分类 `+load` 后调**：同一个 Mach-O 内通常受编译/链接顺序影响，跨动态库、bundle 或不同 image 时不应依赖这个顺序；
3. **`+load` 默认只调一次**，除非你主动调它。

主类的 `+load` 一定先于分类的 `+load`。但分类之间不是按继承关系调的，所以可能是 `父类 → 子类 → 父类分类 → 子类分类`，也可能是 `父类 → 子类 → 子类分类 → 父类分类`，**不要把多个 Category 的 +load 顺序设计成业务依赖**。

## 为什么 Category 的 +load 不会"覆盖"主类的 +load？

很多人会困惑：前面那一节刚论证过 Category 同名方法会"覆盖"主类的，那为什么主类和所有 Category 的 `+load` 都会被各自调一遍？

**答案是 `+load` 不走 `objc_msgSend`**。

回顾一下"覆盖"的本质——它依赖 `objc_msgSend` 的查找规则：从 `class_rw_t.methods`（或 `rwe->methods`）从前往后查，第一个命中的胜出。Category 的方法被插在前面，所以"赢"了。

但 `+load` 完全不走这条路。runtime 在 `load_images` 阶段会**直接从类的方法列表里把 `+load` 的 IMP 找出来，像调用普通 C 函数那样调用它**——根本不经过方法查找、不经过缓存。所以：

- 主类的 `+load` IMP 在 `class_ro_t.baseMethods` 里，被找到并调用；
- 每个 Category 的 `+load` IMP 在各自 Category 的方法列表里，也分别被找到并调用；
- 它们井水不犯河水，全都执行。

## 对比：+initialize 会被 Category 覆盖

`+initialize` 看起来和 `+load` 很像，但行为差别巨大——因为**`+initialize` 走 `objc_msgSend`**。

`+initialize` 在类第一次收到消息时被 runtime 调用，调用方式就是普通的消息发送。所以：

- 如果 Category 实现了 `+initialize`，**它会按"覆盖"规则把主类的 `+initialize` 屏蔽掉**——查找时 Category 的版本先命中；
- 如果子类没实现 `+initialize`，调用子类会沿继承链找到父类的，导致父类的 `+initialize` 被多次调用（每个未覆盖的子类一次）。

**一句话总结**：`+load` 是"直接拿地址调用"，所以人人有份；`+initialize` 是"走消息发送"，所以会被 Category 覆盖。


# Category 与关联对象

之前提到过，在 Category 中虽然可以声明 `@property`，但是不会自动生成对应的成员变量，也不会自动生成 getter / setter 实现。因此，如果只声明属性但不自己实现访问器，在调用 Category 中声明的属性时就会因为找不到方法而报错。

那么就没有办法使用 Category 中的属性了吗？

答案当然是否定的。我们可以自己实现 getter / setter，并借助**关联对象**（Objective-C Associated Objects）来存值。关联对象能在运行时把任意的值关联到一个对象上。

## 三个 API

```Objective-C
// 1. 设置:给 object 关联一个 (key, value)
void objc_setAssociatedObject(id object, const void *key,
                              id value, objc_AssociationPolicy policy);

// 2. 获取
id objc_getAssociatedObject(id object, const void *key);

// 3. 移除该对象所有关联值(一般不用手动调)
void objc_removeAssociatedObjects(id object);
```
关联对象的精髓只有一句：**值根本没存在对象里，而是存在 runtime 维护的一张全局哈希表里**。

具体来说：

- runtime 维护一张全局表 `AssociationsHashMap`；
- key 是 `(对象指针, 关联key)` 的组合；
- value 是你存的值，连同内存策略（assign / retain / copy）一起包装在 `ObjcAssociation` 里；
- 当对象 `dealloc` 时，runtime 会自动清理这张表里属于它的所有条目（路径是 `objc_destructInstance → _object_remove_assocations`），所以**一般不需要手动调 `objc_removeAssociatedObjects`**。

## 为什么这能绕开"不能加 ivar"

回到文章主线——

**关联对象没有动对象的内存布局**：`class_ro_t.ivars` 没变，`instanceSize` 没变，所有偏移量都没变。值不在对象里，而在外部哈希表里。

所以"看起来"像是给对象加了个属性，本质上是"对象指针 + 关联 key"在外部表里查到的一条记录。这正好呼应文章一开始的判断：

| 类别 | 影响内存布局？ | 处理方式 |
|---|---|---|
| ivar | 是 | 编译期定死，Category 不能加 |
| 方法 | 否 | 进 `class_rw_t.methods`（或 `rwe->methods`），Category 可以加 |
| 属性元数据 | 否 | 进 `class_rw_t.properties`，Category 可以加 |
| 属性的存储 | （绕开） | 不放对象里，放 runtime 全局哈希表 |

Category 不能加 ivar，但**可以在 `class_rw_t` 里附加方法和属性元数据**，再**借助 runtime 的全局哈希表**存值——这两件事合起来，就让 Category 拥有了"看起来能加属性"的能力。


# 其他与 Q&A

**Q：在多个分类中实现同名方法时，会调用哪个？**

普通消息发送会命中方法列表中最靠前的那个实现。通常可以理解为"后附加的分类优先级更高"，但跨 image 的加载顺序不应依赖。在 Xcode 里，"后附加"基本由 **Build Phases → Compile Sources** 顺序决定（排在下面的后编译，通常也后附加）。

**Q：被"覆盖"的原方法还在吗，能调到吗？**

还在，没被删除——`class_rw_t.methods`（或 `rwe->methods`）里旧的 `method_list_t` 仍然存在，只是被排到了后面。但实践上很难直接调到：

- `[super xxx]` 不行——它走的是父类查找；
- 想拿到原方法只能手动通过 `class_copyMethodList` 之类的接口去遍历，绕过 `objc_msgSend`。

正常的 `[obj method]` 调用无法指定调用被遮蔽的主类实现或某个特定分类实现。如果业务确实需要保留旧实现，应该在附加 / 交换前主动保存 IMP，或者干脆避免在多个 Category 中定义同名方法。

**Q：能在 Category 里调到主类的版本吗？**

不能。多个 Category 都同名时，主类的实现一定排在最后，永远不会被 `objc_msgSend` 命中。


# 参考链接

- <https://itcharge.cn/tech/ios-dev/ios-runtime-03/>
- <https://tech.meituan.com/2015/03/03/diveintocategory.html>


## 2026-05-25 12:00:54 Category 加载与 +load 方法调用顺序 ^f15d75
topic:: Category
date:: 2026-05-25 12:00:54
source:: 未知
confidence:: 0.70
tags:: Category
summary:: Category 加载涉及 +load 方法的执行顺序问题。；在 Category 中，+load 方法的调用顺序与主类（原类）相关。；涉及 Objective-C 运行时在类加载阶段的处理。
**来源**: 未知　**confidence**: 0.70

- Category 加载涉及 +load 方法的执行顺序问题。
- 在 Category 中，+load 方法的调用顺序与主类（原类）相关。
- 涉及 Objective-C 运行时在类加载阶段的处理。

### 整理后内容

先调用主类的+load方法。

<details><summary>原文</summary>

先调用主类的+load

</details>

---
