---
title: "【iOS】block"
slug: "ios-block"
published: 2026-03-08
description: "block 对应结构体的定义如下： 图中我们可以看到，isa其实有六个部分 isa指针，所以对象都有isa指针。这就证明了 block其实本质上就是一个Objective C对象 ，他的值通常是这三种 NSConcreteGlobalBlock (全局区：没捕获任何外部变量) N"
tags: ["iOS", "Objective-C", "Block"]
category: "iOS"
draft: false
---

## block


![图片](./csdn-158811388-ios-block-images/01-6a109bda543e4fffb15e270ba4f33619.png)


对应结构体的定义如下：


```objective-c
struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};

struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};
```


图中我们可以看到，isa其实有六个部分


- isa指针，所以对象都有isa指针。这就证明了**block其实本质上就是一个Objective-C对象** ，他的值通常是这三种 `_NSConcreteGlobalBlock` (全局区：没捕获任何外部变量)
- `_NSConcreteStackBlock` (栈区：捕获了变量，但还没被强引用)
- `_NSConcreteMallocBlock` (堆区：被拷贝到了堆上，生命周期由你掌控)

- block如果不访问自由变量的话,都是存储在全局区的,如果访问全局变量的话,也是存储在全局区的Block
- block如果访问自由变量的话 如果没有创建block变量,才会创建一个栈区Block变量
- 创建了一个Block变量,且访问自由变量,才会创建出一个堆区的Block,这里创建出堆区Block的原因是 栈区的Block执行了拷贝操作


- flags，他是有个int整型数字，按bit来存储Block的各种附加状态。Runtime需要靠它来判断这个Block的特性
- Reserved，一个保留字段 基本没啥用 主要是为了内存对齐
- invoke，这是一个函数指针 当你在大括号里写下逻辑时，编译器会把括号里的代码抽离出来变为一个普通的C语言函数。invoke指向这个函数的内存地址。当你调用 `myBlock()` 时，底层实际上执行的是 `myBlock->invoke(myBlock, ...)`。**它把 Block 自己作为第一个参数传了进去** 因为只有把 Block 自己传进去，里面的代码才能拿到储存在 Block 尾部的那些“捕获变量”。
- descriptor，他是只想另一个结构体的指针，记录这个Block的辅助信息：size，copy函数指针（当 Block 从栈拷贝到堆（Malloc）时，调用它来保住捕获的对象（比如执行 `retain`）），dispose函数指针（当 Block 从堆上销毁时，调用它来释放捕获的对象（比如执行 `release`）。）
- variables，capture过来的变量，block能够访问它外部的局部变量，就是因为这些变量（或其地址）复制到了结构体中。他是6个部分中**唯一动态变化**的部分，前面的 1~5 是所有 Block 都有的“固定头部”，而第 6 部分是直接**拼贴**在描述信息之后的内存块。block的灵魂就在这里，如果block用到了外部的int a和NSString *b，编译器会在这个位置塞入a的值和b的指针。


### _NSConcreteGlobalBlock


全局的静态block，不会访问任何外部变量，就是NSConcreteGlobalBlock。如下所示：他是一个全局的block，在编译期间就已经决定了。


```objective-c
#include <stdio.h>

int main()
{
    ^{ printf("Hello, World!\n"); } ();
    return 0;
}
```


经过clang命令后 其中关键代码如下：


```objc
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    //  构造函数，用来初始化block
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    printf("Hello, World!\n");
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0) };

int main()
{
    (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA) ();
    return 0;
}
```


__main_block_impl_0就是该block的实现


- 一个block实际就是一个对象，他主要由一个isa和一个impl和一个descriptor组成。
- 由于 clang 改写的具体实现方式和 LLVM 不太一样，并且这里没有开启 ARC。所以这里我们看到 isa 指向的还是`_NSConcreteStackBlock`。但在 LLVM 的实现中，开启 ARC 时，block 应该是 _NSConcreteGlobalBlock 类型，具体可以看 [《objective-c-blocks-quiz》](http://blog.parse.com/2013/02/05/objective-c-blocks-quiz/) 第二题的解释。
- impl 是实际的函数指针，本例中，它指向 __main_block_func_0。这里的 impl 相当于之前提到的 invoke 变量，只是 clang 编译器对变量的命名不一样而已。
- descriptor 是用于描述当前这个 block 的附加信息的，包括结构体的大小，需要 capture 和 dispose 的变量列表等。结构体大小需要保存是因为，每个 block 因为会 capture 一些变量，这些变量会加到 __main_block_impl_0 这个结构体中，使其体积变大。在该例子中我们还看不到相关 capture 的代码，后面将会看到。


### _NSConcreteStackBlock


保存在栈的block，当函数返回时被销毁。可以这么理解，他就是引用了外部变量的block


```objective-c
#include <stdio.h>

int main() {
    int a = 100;
    void (^block2)(void) = ^{
        printf("%d\n", a);
    };
    block2();

    return 0;
}
```


转化后代码如下，NSConcreteStackBlock内部会有一个结构体__main_block_impl_0，这个结构体会保存外部变量，使其体积变大。而这就导致了NSConcreteStackBlock并不像宏一样，而是一个动态的对象。而它由于没有被持有，所以在它的内部，它也不会持有其外部引用的对象。


```objective-c
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int a;	//多了一个被捕获的变量a
  //	构造函数，cpp的初始化列表中，它把外面传进来的100，复制了一份
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) {
        impl.isa = &_NSConcreteStackBlock;	//分配在栈上
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int a = __cself->a; // bound by copy（复制过来的）
    printf("%d\n", a);
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main()
{
    int a = 100;
    void (*block2)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a);
    ((void (*)(__block_impl *))((__block_impl *)block2)->FuncPtr)((__block_impl *)block2);

    return 0;
}
```


- __main_block_impl_0 中增加了一个变量 a，在 block 中引用的变量 a 实际是在申明 block 时，被复制到 `__main_block_impl_0` 结构体中的那个变量 a。因为这样，我们就能理解，在 block 内部修改变量 a 的内容，不会影响外部的实际变量 a。
- __main_block_impl_0 中由于增加了一个变量 a，所以结构体的大小变大了，该结构体大小被写在了 `__main_block_desc_0` 中。
- `__main_block_func_0`中的a是值拷贝，如果此时在block内部实现中作 a++操作，是有问题的，会造成编译器的代码歧义，即此时的a是只读的.

**总结：** block捕获外部变量时，在`内部自动生成同一个属性来保存`


![在这里插入图片描述](./csdn-158811388-ios-block-images/02-16f180f3bef047d0a8b0b4df0f5b971b.png)


我们在变量前面加一个__block关键字后，重新观察生成代码：


```objective-c
struct __Block_byref_i_0 {
    void *__isa;
    __Block_byref_i_0 *__forwarding;	//新增结构体，用于保存我们要capture并且修改变量的i
    int __flags;
    int __size;
    int i;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_i_0 *i; // by ref   达到修改外部变量的作用
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_i_0 *_i, int flags=0) : i(_i->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_i_0 *i = __cself->i; // bound by ref

    printf("%d\n", (i->__forwarding->i));
    (i->__forwarding->i) = 1023;
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->i, (void*)src->i, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->i, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);	//Block被拷贝到堆上时，把内部的i也copy进去
    void (*dispose)(struct __main_block_impl_0*);//	Block从堆上销毁时，把i一同销毁
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main()
{
    __attribute__((__blocks__(byref))) __Block_byref_i_0 i = {(void*)0,(__Block_byref_i_0 *)&i, 0, sizeof(__Block_byref_i_0), 1024};
    void (*block1)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_i_0 *)&i, 570425344);
    ((void (*)(__block_impl *))((__block_impl *)block1)->FuncPtr)((__block_impl *)block1);
    return 0;
}
```


- 我们观察到新增了一个名为 __Block_byref_i_0 的结构体，用来保存我们要 capture 并且修改的变量 i的指针和值。他的里面有一个isa指针，也就意味着他变成了一个对象。这样在Block从栈被拷贝到堆的时候，底层才能通过引用计数来管理他的生命周期。
- __main_block_impl_0 中引用的是 __Block_byref_i_0 的结构体指针，这样就可以达到修改外部变量的作用。
- 我们需要负责 __Block_byref_i_0 结构体相关的内存管理，所以 __main_block_desc_0 中增加了 copy 和 dispose 函数指针，对于在调用前后修改相应变量的引用计数。
- (i->__forwarding->i) = 1023; 为什么要这么绕一圈？ 刚执行在栈上时，这个指针指向他自己，forwarding相对于在改变自己的内存。当Block被拷贝到堆上（比如使用GCD异步）时，系统会把这个`__Block_byref_i_0`结构体原封不动复制在堆上一份，此时有两个i结构体，一个在栈一个在堆，系统会把那个旧结构体的`forwarding`指针强行掰弯，让他指向堆上的那个新结构体，同时，堆上新结构体的`__forwarding`指针指向他自己。这样无论从外面栈上的作用域去访问 还是从Block内部（堆上的作用域）去访问i，所有人都要经过`i->__forwarding->i`，所以对最终访问到的，永远是堆上的同一块内存。


![在这里插入图片描述](./csdn-158811388-ios-block-images/03-e351e7c781a347febeae43c76f85036c.png)


### _NSConcreteMallocBlock


保存在堆上的block，当引用计数为0时会被销毁。NSConcreteMallocBlock 类型的 block 通常不会在源码中直接出现，因为默认它是当一个 block 被 copy 的时候，才会将这个 block 复制到堆中。以下是一个 block 被 copy 时的示例代码 (来自 [这里](http://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/))，可以看到，在第 8 步，目标的 block 类型被修改为 _NSConcreteMallocBlock。


```objective-c
static void *_Block_copy_internal(const void *arg, const int flags) {
    struct Block_layout *aBlock;
    const bool wantsOne = (WANTS_ONE & flags) == WANTS_ONE;

    // 1
    if (!arg) return NULL;

    // 2
    aBlock = (struct Block_layout *)arg;

    // 3
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        // latches on high
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }

    // 4
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }

    // 5
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if (!result) return (void *)0;

    // 6
    memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first

    // 7
    result->flags &= ~(BLOCK_REFCOUNT_MASK);    // XXX not needed
    result->flags |= BLOCK_NEEDS_FREE | 1;

    // 8
    result->isa = _NSConcreteMallocBlock;

    // 9
    if (result->flags & BLOCK_HAS_COPY_DISPOSE) {
        (*aBlock->descriptor->copy)(result, aBlock); // do fixup
    }

    return result;
}
### 变量的复制


对应block外的变量引用，block默认是将其复制到其数据结构中实现访问，


![在这里插入图片描述](./csdn-158811388-ios-block-images/04-9f437aa583a34cf89bb873866b498792.png)


对于用__block修饰的外部变量引用，block通过复制其地址来实现


![在这里插入图片描述](./csdn-158811388-ios-block-images/05-1f9902ac6a1f4056a3d2263c24c32cc0.png)


### Block的三层拷贝


想要分析block的三层copy，首先需要知道外部变量的种类有哪些，其中用的最多的是`BLOCK_FIELD_IS_OBJECT`和`BLOCK_FIELD_IS_BYREF`


```objc
// Runtime support functions used by compiler when generating copy/dispose helpers

// Values for _Block_object_assign() and _Block_object_dispose() parameters
enum {
    // see function implementation for a more complete description of these fields and combinations
    //普通对象，即没有其他的引用类型,也就是我们任何的一个Object都是这个逻辑
    BLOCK_FIELD_IS_OBJECT   =  3,  // id, NSObject, __attribute__((NSObject)), block, ...
    //block类型作为变量，block套block
    BLOCK_FIELD_IS_BLOCK    =  7,  // a block variable
    //经过__block修饰的变量和对象
    BLOCK_FIELD_IS_BYREF    =  8,  // the on stack structure holding the __block variable
    //weak 弱引用变量
    BLOCK_FIELD_IS_WEAK     = 16,  // declared __weak, only used in byref copy helpers
    //返回的调用对象 - 处理block_byref内部对象内存会加的一个额外标记，配合flags一起使用
    BLOCK_BYREF_CALLER      = 128, // called from __block (byref) copy/dispose support routines.
};
#### __Block_copy_internal


```objc
static void *_Block_copy_internal(const void *arg, const int flags)
{
    struct Block_layout *aBlock;
    const bool wantsOne = (WANTS_ONE & flags) == WANTS_ONE;

    if (!arg) return NULL;

    aBlock = (struct Block_layout *)arg; // 强制转化成Block_layout对象,防止对外界造成影响
    if (aBlock->flags & BLOCK_NEEDS_FREE)         // NSConcreteMallocBlock
      //是否需要释放
    {
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL)     // NSConcreteGlobalBlock
      //如果是全局block,直接返回
    {
        return aBlock;
    }
		//	为栈block或者是堆区block,由于堆区需要申请内存,所以是栈区的block的操作
                                                 // Its a stack block.  Make a copy.
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if (!result) return (void *)0;
  //通过memmove内存拷贝,将aBlock按位拷贝到result
    memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first
    // reset refcount
    result->flags &= ~(BLOCK_REFCOUNT_MASK);    // XXX not needed
    result->flags |= BLOCK_NEEDS_FREE | 1;
    result->isa = _NSConcreteMallocBlock;	//isa指针被重写 变身堆block
    if (result->flags & BLOCK_HAS_COPY_DISPOSE) 	//如果 Block 捕获了对象（比如强引用了 self），或者使用了 __block 变量，它的 flags 里就会带有 BLOCK_HAS_COPY_DISPOSE 标记。
    {
            (*aBlock->descriptor->copy)(result, aBlock); // do fixup
      // 对__block对象 把它也拷贝到堆上，并把那个forwarding指针指向堆区新位置
    }

    return result;
}
```


这个copy就是栈block到堆block到核心，无论你在上层写 `[block copy]` 还是 ARC 帮你自动 copy，全天下的 Block 最终都要流经这个。


#### __Block_object_assign


只有当你的 `__block` 修饰的不仅是个基本类型，而是一个**真正的 Objective-C 对象**时（比如 `__block NSObject *obj`），才会触发这最深的一层。现在`Block_byref` 结构体已经搬到堆上了，它里面还装着一个 `NSObject *obj` 的指针！这个 `NSObject` 可是由 ARC（引用计数）管理的。既然堆上的 `Block_byref` 现在持有了它，就必须对它负责！堆上的 `Block_byref` 结构体会调用 `_Block_object_assign`，并传入代号 `BLOCK_FIELD_IS_OBJECT` (值为 3)。系统一看，懂了，对肚子里这个真正的 OC 对象执行一次 `retain` 操作（引用计数 +1），确保它不会被提前释放。


- 如果是普通对象（3 - IS_OBJECT），则交给`系统arc处理`，并`拷贝对象指针`，即`引用计数+1`，所以外界变量不能释放
- 如果是`block类型`的变量（7 - IS_BLOCK），则通过`_Block_copy`操作，将block从`栈区拷贝到堆区`
- 如果是`__block修饰`的变量（8 - IS_BYREF），调用`_Block_byref_copy`函数 进行内存拷贝以及常规处理
- 如果是弱引用（16 - IS_WEAK），他只做弱引用绑定，不增加引用计数，防止循环引用


#### __Block_byref_copy


如果block的标识位里有`_Block_byref_copy`有 `BLOCK_HAS_COPY_DISPOSE`，并且捕获的变量带有 `__block` 前缀。发生第二层拷贝


```objective-c
static struct Block_byref *_Block_byref_copy(const void *arg) {

    //强转为Block_byref结构体类型，保存一份
    struct Block_byref *src = (struct Block_byref *)arg;

    if ((src->forwarding->flags & BLOCK_REFCOUNT_MASK) == 0) {
        // src points to stack 申请内存
        struct Block_byref *copy = (struct Block_byref *)malloc(src->size);
        copy->isa = NULL;
        // byref value 4 is logical refcount of 2: one for caller, one for stack
        copy->flags = src->flags | BLOCK_BYREF_NEEDS_FREE | 4;
        //block内部持有的Block_byref 和 外界的Block_byref 所持有的对象是同一个，这也是为什么__block修饰的变量具有修改能力
        //copy 和 scr 的地址指针达到了完美的同一份拷贝，目前只有持有能力
        copy->forwarding = copy; // patch heap copy to point to itself // 把堆区的指针指向自己
        src->forwarding = copy;  // patch stack to point to heap copy // 把栈区的指针指向堆区,保证数值一致
        copy->size = src->size;
        //如果有copy能力
        if (src->flags & BLOCK_BYREF_HAS_COPY_DISPOSE) { // 有自己的一个copy的逻辑
            // Trust copy helper to copy everything of interest
            // If more than one field shows up in a byref block this is wrong XXX
            //Block_byref_2是结构体，__block修饰的可能是对象，对象通过byref_keep保存，在合适的时机进行调用
            struct Block_byref_2 *src2 = (struct Block_byref_2 *)(src+1);
            struct Block_byref_2 *copy2 = (struct Block_byref_2 *)(copy+1);
            copy2->byref_keep = src2->byref_keep; // 调用自定义复制逻辑
            copy2->byref_destroy = src2->byref_destroy;

            if (src->flags & BLOCK_BYREF_LAYOUT_EXTENDED) {
                struct Block_byref_3 *src3 = (struct Block_byref_3 *)(src2+1);
                struct Block_byref_3 *copy3 = (struct Block_byref_3*)(copy2+1);
                copy3->layout = src3->layout;
            }
            //等价于 __Block_byref_id_object_copy
            (*src2->byref_keep)(copy, src);
        }
        else {
            // Bitwise copy.
            // This copy includes Block_byref_3, if any.
            memmove(copy+1, src+1, src->size - sizeof(*src)); //如果为简单类型就直接进行一个内存拷贝
        }
    }
    // already copied to heap
    else if ((src->forwarding->flags & BLOCK_BYREF_NEEDS_FREE) == BLOCK_BYREF_NEEDS_FREE) {
        latching_incr_int(&src->forwarding->flags);
    }

    return src->forwarding;
}
```


咱们之前说过，加了 `__block` 的变量会被编译器变成一个庞大的结构体（`Block_byref`）。这个结构体最开始也是在栈上的，也需要被搬到堆上面。这一层解决了__block变量在堆区的存活和数据同步问题，如果是类似`__block int a = 10`，拷贝到这里也结束了。


总结：


- 第一层通过_Block_copy实现对象的自身拷贝,从栈区拷贝到堆区
- 第二层通过调用_Block_byref_copy这个来实现对于对象拷贝成Block_byref类型
- 第三次调用_Block_object_assign对于__block修饰的当前变量内部对象的 内存 管理

**当且仅当用__block变量的时候才会有三次拷贝.**


### 循环引用


- Weak wrong dance在这里不过多赘述，给出几行总结 如果block内部没有直接嵌套block，直接使用`__weak`修饰self
- 如果block内部嵌套block，需要同时使用`__weak` 和 `__strong`

- __block修饰符。我们可以采用`__block`修饰符,在主动调用完后手动释放self ```objective-c
__block PersonViewController* vc = self;
    self.testBlock = ^(void){
        NSLog(@"%@", vc.name);
        vc = nil;
    };
self.testBlock();
```
- 对象self作为参数。主要是把对象self作为参数,提供给block内部使用,不会有引用计数问题


```objective-c
self.testBlock = ^(PersonViewController* vc){
        NSLog(@"%@", vc.name);
    };
### ACR与block类型的影响


> 在 ARC 开启的情况下，将只会有 NSConcreteGlobalBlock 和 NSConcreteMallocBlock 类型的 block。


在上面介绍NSConcreteStackBlock的时候，是在ARC环境下跑的，而打印出来的日志明确的显示出，当时的block类型为NSConcreteStackBlock。


而实际上，为什么大家普遍会认为ARC下不存在NSConcreteStackBlock呢？


**这是因为本身我们常常将block赋值给变量，而ARC下默认的赋值操作是strong的，到了block身上自然就成了copy，所以常常打印出来的block就是NSConcreteMallocBlock了。**


原本的 NSConcreteStackBlock 的 block 会被 NSConcreteMallocBlock 类型的 block 替代。证明方式是以下代码在 XCode 中，会输出 ` `。在苹果的 [官方文档](http://developer.apple.com/library/ios/#releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html) 中也提到，当把栈中的 block 返回时，不需要调用 copy 方法了。


我个人认为这么做的原因是，由于 ARC 已经能很好地处理对象的生命周期的管理，这样所有对象都放到堆上管理，对于编译器实现来说，会比较方便。

---

原文发布于 CSDN：[【iOS】block](https://blog.csdn.net/2402_86720949/article/details/158811388)
