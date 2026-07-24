
`dyld（the dynamic link editor）`是苹果的**动态链接器**，是苹果操作系统的重要组成部分，在`app`被编译打包成可执行文件格式的`Mach-O`文件后，交由`dyld`负责连接，并加载程序。

- `dyld`的作用：加载各个库，也就是`image`**镜像文件**，由`dyld`从内存中读到表中，加载主程序，`link`链接各个动静态库，进行主程序初始化。

![截屏2023-02-02 下午4.17.22.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65680fa54a614b56997cd531c98e5a60~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

**调用流程**：`start` -> `dyld4::prepare` -> `dyld4::APIs::runAllInitializersForMain` -> `dyld4::Loader::runInitializersBottomUpPlusUpwardLinks` -> `dyld4::Loader::runInitializersBottomUp` -> `dyld4::RuntimeState::notifyObjCInit` -> `load_images` -> `[ViewController load]`。

# dyld源码

## start

``` Objective-C
namespace dyld4 {
...
...
// dyld的入口点。内核加载dyld并跳转到__dyld_start，它设置一些寄存器并调用这个函数。
// 注意:这个函数永远不会返回，它调用exit()。
// 因此，堆栈保护程序是无用的，因为永远不会执行epilog。
// 标记函数no-return禁用堆栈保护。
// 堆栈保护器也会导致armv7k代码生成的问题，因为它通过prolog中的GOT槽访问随机值，但dyld还没有rebased。
void start(const KernelArgs* kernArgs, void* prevDyldMH) __attribute__ ((noreturn)) __asm("start");
void start(const KernelArgs* kernArgs, void* prevDyldMH)
{
    // 发出kdebug跟踪点来指示dyld引导程序已经启动 <rdar://46878536>
    // 注意:这是在dyld rebased之前调用的，所以kdebug_trace_dyld_marker()不能使用任何全局变量
    dyld3::kdebug_trace_dyld_marker(DBG_DYLD_TIMING_BOOTSTRAP_START, 0, 0, 0, 0);

    // 走所有的fixups chains 和 rebase dyld
    // 注意:withChainStarts()和fixupAllChainedFixups()不能使用任何静态数据指针，因为它们还没有rebased
    const MachOAnalyzer* dyldMA = getDyldMH(); //获取Mach-O解析器，用于解析Mach-O格式文件内容
    uintptr_t            slide  = dyldMA->getSlide();// 会生成一个随机数(ASLR)
    if ( !dyldMA->inDyldCache() ) {
        assert(dyldMA->hasChainedFixups());
        __block Diagnostics diag;
        dyldMA->withChainStarts(diag, 0, ^(const dyld_chained_starts_in_image* starts) {
            dyldMA->fixupAllChainedFixups(diag, starts, slide, dyld3::Array<const void*>(), nullptr);
        });
        diag.assertNoError();

        // make __DATA_CONST read-only (kernel maps it r/w)
        dyldMA->forEachSegment(^(const MachOAnalyzer::SegmentInfo& segInfo, bool& stop) {
            if ( segInfo.readOnlyData ) {
                const uint8_t* start = (uint8_t*)(segInfo.vmAddr + slide);
                size_t         size  = (size_t)segInfo.vmSize;
                sSyscallDelegate.mprotect((void*)start, size, PROT_READ);
            }
        });
    }
    
    // 现在，我们可以调用使用DATA的函数
    mach_init();// mach消息初始化
    
    // 为stack canary设置随机值
    __guard_setup(kernArgs->findApple()); //栈溢出保护

    // 设置以便open_with_subsystem()工作
    _subsystem_init(kernArgs->findApple());

    // 在__DATA_CONST中将ProcessConfig对象变为只读之前，使用placement new来构造它
    handleDyldInCache(dyldMA, kernArgs, (MachOFile*)prevDyldMH);

    bool useHWTPro = false;
    // 在自己的分配池中创建一个分配器Allocator
    Allocator& allocator = Allocator::persistentAllocator(useHWTPro);

    // 使用placement new在Allocator池中构造ProcessConfig对象
    // 构建进程配置
    ProcessConfig& config = *new (allocator.aligned_alloc(alignof(ProcessConfig), sizeof(ProcessConfig))) ProcessConfig(kernArgs, sSyscallDelegate, allocator);//构建进程配置
    
#if !SUPPPORT_PRE_LC_MAIN
    // 堆栈分配RuntimeLocks。它们不能在通常为只读的Allocator池中
    RuntimeLocks  sLocks;
#endif

    // 在分配器中创建API（也称为RuntimeState）对象
    APIs& state = *new (allocator.aligned_alloc(alignof(APIs), sizeof(APIs))) APIs(config, allocator, sLocks);

#if !TARGET_OS_SIMULATOR
    // FIXME：我们应该更早地移动它，但目前我们需要在设置压缩信息之前将运行时状态初始化，
    // 直到我们这样做之前，压缩信息可能会错过某些早期的dyld崩溃
    auto processSnapshot = state.getCurrentProcessSnapshot();
    processSnapshot->setPlatform((uint64_t)state.config.process.platform);
    processSnapshot->setDyldState(dyld_process_state_dyld_initialized);
    FileRecord cacheFileRecord = state.fileManager.fileRecordForVolumeDevIDAndObjID(gProcessInfo->sharedCacheFSID, gProcessInfo->sharedCacheFSObjID);
    if (cacheFileRecord.exists()) {
        auto sharedCache = SharedCache(state.ephemeralAllocator, std::move(cacheFileRecord), processSnapshot->identityMapper(), (uint64_t)config.dyldCache.addr, gProcessInfo->processDetachedFromSharedRegion);
        processSnapshot->addSharedCache(std::move(sharedCache));
    }
    
    // 将dyld添加到压缩信息
    if ( dyldMA->inDyldCache() && processSnapshot->sharedCache() ) { ··· } else { ··· }
    
    // 将主可执行文件添加到压缩信息
    FileRecord mainExecutableFile;
    if (state.config.process.mainExecutableFSID && state.config.process.mainExecutableObjID) { ··· } else { ··· }
    auto mainExecutableImage = Image(state.ephemeralAllocator, std::move(mainExecutableFile), processSnapshot->identityMapper(), (const mach_header*)state.config.process.mainExecutable);
    processSnapshot->addImage(std::move(mainExecutableImage));
    processSnapshot->setInitialImageCount(state.initialImageCount());
    state.commitProcessSnapshot();
#endif

    // -------------重点入口-------------------
    // 加载所有的库与程序
    MainFunc appMain = prepare(state, dyldMA);

    // 现在将所有dyld分配的数据结构设置为只读
    state.decWritable();

    // 调用main()，如果它返回，调用exit()并返回结果
    // 注意:这是经过组织的，以便在程序的主线程中回溯时只在“main”下面显示“start”。
    int result = appMain(state.config.process.argc, state.config.process.argv, state.config.process.envp, state.config.process.apple);

    // 如果我们到达这里，main（）返回（与程序调用exit（）相反）
#if TARGET_OS_OSX
    // <rdar://74518676>libSystemHelpers没有为模拟器设置，因此直接调用_exit（）
    if ( MachOFile::isSimulatorPlatform(state.config.process.platform) )
        _exit(result);
#endif
    state.libSystemHelpers->exit(result);
}
...
...
}// namespace
```

- ①.`dyld4`跟`dyld3`不同就是，`main`没有了返回值，是通过`exit`结束！无法通过`epilog`退出代码，它释放堆栈空间并返回到调用者。
- ②. `start`中读取`mach-o`文件中的符号地址都是**虚拟地址**，在程序启动的时候，系统会生成一个随机数(`ASLR`),使用虚拟地址加上`ASLR`才是物理地址,也就是程序真正调用的地址。我们把从**虚拟地址**换算成**物理地址**的过程称之为`rebase`。

## prepare

``` Objective-C
// 加载任何相关的dylib并将它们绑定在一起。
// 返回main()在target中的地址。
__attribute__ ((noinline)) static MainFunc prepare(APIs& state, const MachOAnalyzer* dyldMH)
{
    // ===== 阶段一：初始化 gProcessInfo =====
    // gProcessInfo 是全局结构体 dyld_all_image_infos，存储进程中所有镜像的元信息
    // 供调试器、CrashReporter 等工具读取
    gProcessInfo->terminationFlags = 0;        // 0 = 崩溃时正常显示调用栈
    gProcessInfo->platform         = (uint32_t)state.config.process.platform;
    gProcessInfo->dyldPath         = state.config.process.dyldPath;

    // kdebug 性能追踪：记录 App 启动时间点
    uint64_t launchTraceID = 0;
    if ( dyld3::kdebug_trace_dyld_enabled(DBG_DYLD_TIMING_LAUNCH_EXECUTABLE) ) {
        launchTraceID = dyld3::kdebug_trace_dyld_duration_start(DBG_DYLD_TIMING_LAUNCH_EXECUTABLE, (uint64_t)state.config.process.mainExecutable, 0, 0);
    }

    // ===== 阶段二：模拟器路径分叉 =====
    // 如果是模拟器程序，直接转发给 prepareSim()，走完全独立的加载流程，真机继续往下
#if TARGET_OS_OSX
    const bool isSimulatorProgram = MachOFile::isSimulatorPlatform(state.config.process.platform);
    if ( const char* simPrefixPath = state.config.pathOverrides.simRootPath() ) {
        if ( isSimulatorProgram ) {
            char simDyldPath[PATH_MAX];
            strlcpy(simDyldPath, simPrefixPath, PATH_MAX);
            strlcat(simDyldPath, "/usr/lib/dyld_sim", PATH_MAX);
            return prepareSim(state, simDyldPath); // 转入模拟器专属流程
        }
    }
#endif

    // ===== 阶段三：Loader 选择（PrebuiltLoader vs JustInTimeLoader）=====
    // dyld4 双模式解析：
    //   PrebuiltLoader  —— 上次启动序列化到磁盘的快照，直接复用，启动更快（非首次启动）
    //   JustInTimeLoader —— 实时解析 Mach-O 构建 Loader，用于首次启动或文件变更后
    state.initializeClosureMode();
    const PrebuiltLoaderSet* mainSet    = state.processPrebuiltLoaderSet();
    Loader*                  mainLoader = nullptr;
    if ( mainSet != nullptr ) {
        mainLoader = (Loader*)mainSet->atIndex(0); // 使用预构建 Loader
        state.loaded.reserve(state.initialImageCount());
    }
    if ( mainLoader == nullptr ) {
        // 没有预构建，实时构建 JustInTimeLoader
        state.loaded.reserve(512);
        Diagnostics buildDiag;
        mainLoader = JustInTimeLoader::makeLaunchLoader(buildDiag, state, state.config.process.mainExecutable,
                                                        state.config.process.mainExecutablePath, nullptr);
        if ( buildDiag.hasError() )
            halt(buildDiag.errorMessage());
    }
    state.setMainLoader(mainLoader);
    state.notifyDebuggerLoad(mainLoader); // 通知调试器主程序已加载

    // ===== 阶段四：加载插入的 dylib（DYLD_INSERT_LIBRARIES）=====
    // 遍历 DYLD_INSERT_LIBRARIES 中指定的所有 dylib，逐一加载
    // 插入的 dylib 被放在 topLevelLoaders 最前面，保证先于主程序初始化
    // 这是越狱注入、fishhook 等 hook 技术的底层入口
    STACK_ALLOC_OVERFLOW_SAFE_ARRAY(Loader*, topLevelLoaders, 16);
    topLevelLoaders.push_back(mainLoader);
    Loader::LoadChain   loadChainMain { nullptr, mainLoader };
    Loader::LoadOptions options;
    options.staticLinkage = true;
    options.launching     = true;
    options.insertedDylib = true;
    options.canBeDylib    = true;
    options.rpathStack    = &loadChainMain;
    state.config.pathOverrides.forEachInsertedDylib(^(const char* dylibPath, bool& stop) {
        Diagnostics insertDiag;
        if ( Loader* insertedDylib = (Loader*)Loader::getLoader(insertDiag, state, dylibPath, options) ) {
            topLevelLoaders.push_back(insertedDylib);
            state.notifyDebuggerLoad(insertedDylib);
        }
        else if ( insertDiag.hasError() && !state.config.security.allowInsertFailures ) {
            halt(insertDiag.errorMessage());
        }
    });

    // ===== 阶段五：递归加载所有依赖 =====
    // 对主程序和每个插入的 dylib，递归加载其依赖的所有动态库
    // 这是启动过程中最耗时的部分，构建出完整的依赖图
    // 完成后写入 gProcessInfo->initialImageCount，让 CrashReporter 区分
    // 启动时加载的库（此处）与运行时 dlopen 加载的库
    Diagnostics depsDiag;
    options.insertedDylib = false;
    for ( Loader* ldr : topLevelLoaders ) {
        ldr->loadDependents(depsDiag, state, options);
        if ( depsDiag.hasError() ) {
            gProcessInfo->terminationFlags = 1; // 不显示无意义的调用栈
            halt(depsDiag.errorMessage());
        }
    }
    // 通知调试器所有新加载的 image
    {
        STACK_ALLOC_VECTOR(const Loader*, newLoaders, state.loaded.size());
        for (const Loader* ldr : state.loaded)
            newLoaders.push_back(ldr);
        uintptr_t topCount = topLevelLoaders.count();
        std::span<const Loader*> unnotifiedNewLoaders(&newLoaders[topCount], newLoaders.size() - topCount);
        state.notifyDebuggerLoad(unnotifiedNewLoaders);
        state.notifyDtrace(newLoaders); // 通知内核 dtrace 静态探针
    }
    gProcessInfo->initialImageCount = state.loaded.size(); // 记录启动时镜像数量

    // 将所有非缓存库加入永久地址范围（不可卸载）
    STACK_ALLOC_ARRAY(const Loader*, nonCacheNeverUnloadLoaders, state.loaded.size());
    for (const Loader* ldr : state.loaded) {
        if ( !ldr->dylibInDyldCache )
            nonCacheNeverUnloadLoaders.push_back(ldr);
    }
    state.addPermanentRanges(nonCacheNeverUnloadLoaders);

    // ===== 阶段六：构建 WeakDefMap =====
    // 在绑定（binding）之前先构建 weak symbol 定义表
    // 确保所有 __weak 符号找到唯一实现，避免多库同名 weak symbol 冲突
    if ( state.config.process.proactivelyUseWeakDefMap ) {
        state.weakDefMap = new (state.persistentAllocator.malloc(sizeof(WeakDefMap))) WeakDefMap();
        STACK_ALLOC_VECTOR(const Loader*, allLoaders, state.loaded.size());
        for (const Loader* ldr : state.loaded)
            allLoaders.push_back(ldr);
        Loader::addWeakDefsToMap(state, allLoaders);
    }

    // ===== 阶段七：构建 Interposing 表 =====
    // 收集所有 __DATA,__interpose 段中的函数替换对（interposing tuples）
    // 为后续 fixup 阶段使用，是 fishhook 等 hook 框架的基础机制
    state.buildInterposingTables();

    // ===== 阶段八：Fixups（Rebase + Bind）—— 最核心步骤 =====
    // applyFixups()：对每个 image 执行 rebase（修正 ASLR 偏移）、bind（绑定符号到实际地址）
    // applyCachePatches()：修补共享缓存中未找到的 GOT 槽位
    // doSingletonPatching()：处理单例模式下的符号覆盖
    // handleStrongWeakDefOverrides()：处理主程序 strong symbol 覆盖 dylib weak symbol（C++ 规范要求）
    {
        dyld3::ScopedTimer timer(DBG_DYLD_TIMING_APPLY_FIXUPS, 0, 0, 0);
        DyldCacheDataConstLazyScopedWriter cacheDataConst(state);
        if ( !mainLoader->isPrebuilt )
            JustInTimeLoader::handleStrongWeakDefOverrides(state, cacheDataConst); // C++ weak-def 覆盖处理
        for ( const Loader* ldr : state.loaded ) {
            Diagnostics fixupDiag;
            ldr->applyFixups(fixupDiag, state, cacheDataConst, true); // rebase + bind
            if ( fixupDiag.hasError() )
                halt(fixupDiag.errorMessage());
            ldr->applyCachePatches(state, cacheDataConst); // 修补 GOT
        }
        state.doSingletonPatching(); // 单例修补
    }
    // 将 interposing 替换规则应用到共享缓存
    if ( !state.interposingTuplesAll.empty() ) {
        Loader::applyInterposingToDyldCache(state);
    }

    // ===== 阶段九：连接 libdyld.dylib =====
    // 通过 libdyld.dylib 的 __DATA,__dyld4 段，将 dyld 内部运行时状态暴露给用户空间
    // 使 dlopen/dlsym 等 API 能回调到 dyld4 的实现
    // 同时初始化 argc/argv/envp 等进程变量
    LibdyldDyld4Section* libdyld4Section = nullptr;
    if ( state.libdyldLoader != nullptr ) {
        const MachOLoaded* libdyldML = state.libdyldLoader->loadAddress(state);
        uint64_t sectSize;
        libdyld4Section = (LibdyldDyld4Section*)libdyldML->findSectionContent("__DATA", "__dyld4", sectSize, true);
        if ( libdyld4Section != nullptr ) {
            libdyld4Section->apis          = &state;       // 全局 APIs 对象（RuntimeState）
            libdyld4Section->allImageInfos = gProcessInfo; // 全局镜像信息
            state.vars                     = &libdyld4Section->defaultVars;
            state.vars->mh                 = state.config.process.mainExecutable;
            *state.vars->NXArgcPtr         = state.config.process.argc;
            *state.vars->NXArgvPtr         = (const char**)state.config.process.argv;
            *state.vars->environPtr        = (const char**)state.config.process.envp;
            *state.vars->__prognamePtr     = state.config.process.progname;
        }
        else {
            halt("compatible libdyld.dylib not found");
        }
    }
    if ( state.libSystemLoader == nullptr )
        halt("program does not link with libSystem.B.dylib");

    // ===== 阶段十：序列化 PrebuiltLoaderSet（首次启动优化）=====
    // 如果本次用 JustInTimeLoader 启动（首次 / 文件变更），将 Loader 信息序列化保存到磁盘
    // 下次启动直接读取，跳过解析步骤，显著加快冷启动速度
#if !TARGET_OS_SIMULATOR
    if ( needToWritePrebuiltLoaderSet ) {
        dyld3::ScopedTimer timer(DBG_DYLD_TIMING_BUILD_CLOSURE, 0, 0, 0);
        Diagnostics              prebuiltDiag;
        const PrebuiltLoaderSet* prebuiltAppSet = PrebuiltLoaderSet::makeLaunchSet(prebuiltDiag, state, missingPaths);
        if ( (prebuiltAppSet != nullptr) && prebuiltDiag.noError() ) {
            if ( state.saveAppPrebuiltLoaderSet(prebuiltAppSet) )
                state.setSavedPrebuiltLoaderSet();
            prebuiltAppSet->deallocate();
        }
    }
#endif

    // ===== 阶段十一：运行所有初始化器 =====
    // 按依赖顺序（自底向上）依次执行每个 image 的初始化器：
    //   - __attribute__((constructor)) 函数
    //   - ObjC +load 方法
    //   - C++ 静态对象构造函数
    // 这是 main() 被调用前最后一个阶段，也是 hook 框架和越狱注入执行的时机
    dyld4::notifyMonitoringDyldBeforeInitializers();
    state.runAllInitializersForMain();

    // ===== 阶段十二：返回 main() 地址 =====
    // 将主程序 main() 的函数指针交回 start()，由 start() 完成内存锁定后直接调用
    return result;
}
```

## runAllInitializersForMain

``` Objective-C
// 这是dyldMain.cpp中提取出来的一部分，以支持使用crt1.o的旧macOS应用程序
void APIs::runAllInitializersForMain()
{
    // ===== 阶段一：禁用 page-in linking =====
    // page-in linking 是 dyld4 的懒加载优化，仅用于启动流程，dlopen() 不使用
    // 此时所有依赖已加载完毕，关闭该机制防止后续误触发
    if ( !config.security.internalInstall || (config.process.pageInLinkingMode != 3) )
        config.syscall.disablePageInLinking();

    // ===== 阶段二：优先初始化 libSystem =====
    // libSystem 是所有其他库的基础（提供 malloc/pthread/stdio 等），必须最先初始化
    // beginInitializers() 标记该 Loader 进入初始化阶段
    // runInitializers() 执行 libSystem 自身的 __attribute__((constructor)) 函数
    const_cast<Loader*>(this->libSystemLoader)->beginInitializers(*this);
    this->libSystemLoader->runInitializers(*this);
    gProcessInfo->libSystemInitialized = true; // 标记 libSystem 已完成初始化

    // libSystem 初始化完成后，立即通知 ObjC runtime 运行 libSystem 子 dylib 上的 +load 方法
    // 必须在 libSystem 就绪后才能安全调用 ObjC 运行时
    this->notifyObjCInit(this->libSystemLoader);

    // ===== 阶段三：初始化 /usr/lib/system/ 下的子 dylib =====
    // libSystem 由多个子库组成（如 libsystem_c.dylib、libsystem_malloc.dylib 等）
    // 按顺序初始化所有安装路径以 "/usr/lib/system/lib" 开头的 dylib
    // 用安装名称而非路径判断，是为了兼容 DYLD_LIBRARY_PATH 覆盖的情况
    // 使用下标迭代而非迭代器，防止 +load 中触发 dlopen 导致数组扩容使迭代器失效
    for ( uint32_t i = 0; i != this->loaded.size(); ++i ) {
        const Loader* ldr = this->loaded[i];
        if ( (ldr->dylibInDyldCache || ldr->analyzer(*this)->isDylib()) &&
             (strncmp(ldr->analyzer(*this)->installName(), "/usr/lib/system/lib", 19) == 0) ) {
            const_cast<Loader*>(ldr)->beginInitializers(*this);
            this->notifyObjCInit(ldr);       // 通知 ObjC 运行该库上的 +load
            const_cast<Loader*>(ldr)->runInitializers(*this);
        }
    }

#if TARGET_OS_OSX
    // ===== macOS 专属：PID 1（launchd）扫描根目录 =====
    // 如果当前进程是 PID 1（即 launchd），异步扫描系统根目录
    // libSystemHelpers version >= 5 才支持此接口
    if ( (this->config.process.pid == 1) && (this->libSystemHelpers->version() >= 5) ) {
        this->libSystemHelpers->run_async(&ProcessConfig::scanForRoots, (void*)&this->config);
    }
#endif

    // ===== 阶段四：自底向上运行所有其他初始化器 =====
    // runInitializersBottomUpPlusUpwardLinks() 按依赖顺序（底层库先于上层库）递归执行：
    //   - __attribute__((constructor)) 函数
    //   - ObjC +load 方法（通过 notifyObjCInit 回调）
    //   - C++ 静态对象构造函数
    // 插入的 dylib（DYLD_INSERT_LIBRARIES）排在 loaded 数组前面，因此先于主程序初始化
    // 同样用下标迭代，防止初始化器中 dlopen 导致数组扩容
    // 遇到主可执行文件时立即停止——主程序初始化完成即意味着所有库都已就绪
    for ( uint32_t i = 0; i != this->loaded.size(); ++i ) {
        const Loader* ldr = this->loaded[i];
        ldr->runInitializersBottomUpPlusUpwardLinks(*this);
        // 到达主可执行文件时退出循环
        // 通常主程序是 loaded[0]，但若有 N 个插入库，则为第 N 个
        if ( ldr->analyzer(*this)->isMainExecutable() )
            break;
    }
    // 至此所有初始化器执行完毕，控制权交回 prepare()，由其返回 main() 地址
}
```

①. 返回`main`函数地址，所有配置完后 `_dyld_register_driverkit_main`就是我们自定义工程的`main`地址。

②. `gProcessInfo`是存储`dyld`所有`images`镜像信息的结构体，保存着`mach_header`、`dyld_uuid_info`、`dyldVersion`等等信息。

③. 创建`just-in-time`，是`dyld4`一个新特性。`dyld4`在保留了`dyld3`的 `mach-o` 解析器基础上，同时也引入了 `just-in-time` 的加载器来优化。

- `dyld3` 出于对启动速度的优化的目的, 增加了**预构建(闭包)**。`App`第一次启动或者App发生变化时会将部分启动数据创建为闭包存到本地，那么App下次启动将不再重新解析数据，而是直接读取闭包内容。当然前提是应用程序和系统应很少发生变化，但如果这两者经常变化等, 就会导闭包丢失或失效。
    
- `dyld4` 采用了 `pre-build` + `just-in-time` 的**双解析模式**，**预构建**`pre-build` 对应的就是 `dyld3`中的闭包，`just-in-time`可以理解为**实时解析**。当然`just-in-time` 也是可以利用 `pre-build` 的缓存的，所以性能可控。有了`just-in-time`, 目前应用首次启动、系统版本更新、普通启动，`dyld4` 则可以根据缓存是否有效去选择合适的模式进行解析。
    

## runInitializersBottomUpPlusUpwardLinks

``` Objective-C
void Loader::runInitializersBottomUpPlusUpwardLinks(RuntimeState& state) const
{
    // incWritable()：初始化器执行期间需要写入内存（如 ObjC 类注册、C++ 静态变量赋值）
    // 临时开放写权限，执行完后 decWritable() 再锁回只读
    state.incWritable();

    // ===== 第一轮：自底向上递归初始化，收集 upward link =====
    // upward link（向上链接）：A 依赖 B，同时 B 也反向依赖 A，形成循环依赖
    // 例如：libSystem.dylib 与 libobjc.dylib 之间存在 upward link
    // runInitializersBottomUp() 遇到 upward link 时不递归进入，
    // 而是将其收集到 danglingUpwards 列表，留到本轮结束后统一处理
    STACK_ALLOC_ARRAY(const Loader*, danglingUpwards, state.loaded.size());
    this->runInitializersBottomUp(state, danglingUpwards);

    // ===== 第二轮：处理悬挂的 upward link =====
    // 对第一轮收集到的 upward 节点重新跑一遍 runInitializersBottomUp
    // 此时这些节点的下游依赖已全部初始化完毕，可以安全初始化
    STACK_ALLOC_ARRAY(const Loader*, extraDanglingUpwards, state.loaded.size());
    for ( const Loader* ldr : danglingUpwards ) {
        ldr->runInitializersBottomUp(state, extraDanglingUpwards);
    }

    // ===== 第三轮（防御）：处理二阶 upward link =====
    // 极端情况下 upward link 可能有两层（如 A->B->C 且 C->A）
    // 若第二轮又产生了新的 danglingUpwards，再跑一轮确保全部初始化
    if ( !extraDanglingUpwards.empty() ) {
        danglingUpwards.resize(0);
        for ( const Loader* ldr : extraDanglingUpwards ) {
            ldr->runInitializersBottomUp(state, danglingUpwards);
        }
    }

    // 恢复只读保护
    state.decWritable();
}
```

## runInitializersBottomUp

``` Objective-C
void Loader::runInitializersBottomUp(RuntimeState& state, Array<const Loader*>& danglingUpwards) const
{
    // ===== 幂等保护：避免重复初始化 =====
    // beginInitializers() 用原子操作将状态从"未初始化"改为"初始化中"
    // 如果已经是"初始化中"或"已完成"，返回 true，直接退出
    // 这是防止循环依赖死循环的关键：A->B->A 时第二次遇到 A 直接返回
    if ( (const_cast<Loader*>(this))->beginInitializers(state) )
        return;

    // ===== 深度优先递归：先初始化所有下游依赖 =====
    // 遍历当前 image 的所有依赖（由 LC_LOAD_DYLIB 等 Load Command 解析出来）
    // 策略：先递归初始化依赖，再初始化自己——保证底层库先于上层库就绪
    const uint32_t depCount = this->dependentCount();
    for ( uint32_t i = 0; i < depCount; ++i ) {
        DependentKind childKind;
        if ( Loader* child = this->dependent(state, i, &childKind) ) {
            if ( childKind == DependentKind::upward ) {
                // upward link：该依赖反过来也依赖当前库，形成循环
                // 不能递归进入，否则死循环；改为收集到 danglingUpwards，留给上层统一处理
                if ( !danglingUpwards.contains(child) )
                    danglingUpwards.push_back(child);
            }
            else {
                // 普通依赖（downward link）：直接递归初始化
                child->runInitializersBottomUp(state, danglingUpwards);
            }
        }
    }

    // ===== 通知 ObjC runtime：运行 +load 方法（必须在 C++ 初始化器之前）=====
    // notifyObjCInit() 回调到 libobjc 的 load_images()，触发该 image 中所有 +load
    // +load 调用顺序：父类先于子类，类先于分类
    state.notifyObjCInit(this);

    // ===== 运行当前 image 的所有 C 初始化器 =====
    // 包括：__attribute__((constructor)) 函数、C++ 静态对象构造函数
    // 执行顺序由 Mach-O __DATA,__mod_init_func 段中的函数指针列表决定
    this->runInitializers(state);
}
```

## notifyObjCInit

``` Objective-C
void RuntimeState::notifyObjCInit(const Loader* ldr)
{
    // ===== 双重条件门控，避免无效调用 =====
    // _notifyObjCInit 是 libobjc 注册的回调函数指针（即 load_images）
    // libobjc 在自身初始化时通过 _dyld_objc_notify_register() 注册此回调
    // 两个条件须同时满足才触发：
    //   1. libobjc 已初始化并注册了回调（_notifyObjCInit != nullptr）
    //   2. 该 image 可能含有 +load 方法（mayHavePlusLoad == true）
    //      此标志在 Mach-O 加载时扫描 ObjC section 预判，避免无谓的函数调用
    if ( (_notifyObjCInit != nullptr) && ldr->mayHavePlusLoad ) {
        const MachOLoaded* ml  = ldr->loadAddress(*this);
        const char*        pth = ldr->path();

        // kdebug 性能计时：记录 ObjC 初始化耗时，可在 Instruments 中看到
        dyld3::ScopedTimer timer(DBG_DYLD_TIMING_OBJC_INIT, (uint64_t)ml, 0, 0);

        if ( this->config.log.notifications )
            this->log("objc-init-notifier called with mh=%p, path=%s\n", ml, pth);

        // ===== 回调 libobjc 的 load_images() =====
        // load_images(path, mh) 执行流程：
        //   1. 扫描该 image 的 __DATA,__objc_nlclslist（non-lazy class 列表）
        //   2. 对每个有 +load 方法的类/分类，按继承链顺序调用 +load
        //   3. +load 顺序：父类先于子类，类先于分类，同级按编译顺序
        //   4. 每个类的 +load 全局只调用一次，不受 +initialize 影响
        // 注意：此时 main() 尚未执行，autorelease pool 由 libobjc 在 +load 调用前后自动管理
        _notifyObjCInit(pth, ml);
    }
    // 若 _notifyObjCInit 为 nil（libobjc 尚未初始化）或 mayHavePlusLoad 为 false，
    // 直接跳过——这保证 libSystem 子库在 ObjC runtime 就绪前也能安全调用此函数
}
```

## _dyld_objc_notify_register

``` Objective-C
// libobjc 在自身 +load 初始化时调用此函数，向 dyld 注册三个核心回调
// 这是 dyld 与 ObjC runtime 之间的唯一「桥梁」——只有通过这个注册，
// dyld 才知道每个 image 加载完毕后该通知谁来执行 +load 方法
//
// 原型（dyld_priv.h）：
//   void _dyld_objc_notify_register(
//       _dyld_objc_notify_mapped    mapped,    // 新 image 映射时触发
//       _dyld_objc_notify_init      init,      // image 初始化时触发（即 notifyObjCInit 回调的目标）
//       _dyld_objc_notify_unmapped  unmapped   // image 卸载时触发
//   );
//
// libobjc 在 _objc_init() 中的调用：
void _objc_init(void)
{
    // ... ObjC runtime 内部初始化（锁、哈希表、ARC 等）...

    // 注册三个回调到 dyld RuntimeState：
    //   map_images   → 对应 dyld 的 notifyObjCMapped，扫描 ObjC 元数据（class / protocol / category）
    //   load_images  → 对应 dyld 的 notifyObjCInit，按顺序触发所有 +load 方法
    //   unmap_image  → 对应 dyld 的 notifyObjCUnmapped，清理已卸载 image 的 ObjC 数据
    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
    // 注册完成后，dyld 内部的 RuntimeState::_notifyObjCInit 指针就指向了 load_images
    // 之后每次 notifyObjCInit() 中的 _notifyObjCInit(pth, ml) 调用，
    // 本质上就是 libobjc 的 load_images(path, mach_header*)
}

// ===== 时序关系 =====
// 1. dyld start()
// 2. dyld prepare() → runAllInitializersForMain()
// 3.   libSystem 初始化 → libSystem 内部调用 libdispatch_init, pthread_init...
// 4.   libobjc 初始化 → _objc_init() → _dyld_objc_notify_register()
//       ↑ 此时 RuntimeState::_notifyObjCInit 才非 nullptr
// 5.   之后每个 image 的 notifyObjCInit() 才能真正触发 load_images
// 6.   load_images → 按继承链顺序调用每个类的 +load
// 7.   所有初始化器跑完 → prepare() 返回 main() 地址 → start() 调用 main()
```

## +load 与 +initialize 的区别

``` Objective-C
// ===== +load =====
// 触发时机：image 被 dyld 加载时（main() 之前），由 libobjc 的 load_images() 调用
// 调用方：dyld 通过 notifyObjCInit → load_images 调用，不走消息发送
// 调用条件：只要类/分类存在 +load 实现，必定调用，无需任何对象存在
// 调用次数：每个类/分类的 +load 全局只调用一次
// 调用顺序：父类 > 子类 > 分类（分类之间按编译顺序）
// 线程：主线程，autorelease pool 由 libobjc 自动管理
// 是否继承：不继承，子类不实现则不调用父类的 +load
// 适用场景：method swizzling、注册工厂类、早期 hook（因为此时 runtime 已就绪但 main 未执行）
+ (void)load {
    // 此时 ObjC runtime 已完全就绪（_objc_init 已执行）
    // 但 UIApplicationMain 尚未调用，UI 相关操作不安全
    // 不要在此处做耗时操作：+load 是串行执行的，会阻塞 App 启动
}

// ===== +initialize =====
// 触发时机：类第一次收到消息时（懒加载），由 objc_msgSend 触发
// 调用方：runtime 内部，通过消息发送机制（走 objc_msgSend）
// 调用条件：类第一次被使用（不一定发生）
// 调用次数：每个类只调用一次，但子类未实现时会调用父类的（与 +load 相反）
// 线程安全：有线程安全保证（runtime 内部加锁），可能在任意线程触发
// 是否继承：继承，子类不实现则调用父类的 +initialize（可能被调用多次——每个未实现的子类各一次）
+ (void)initialize {
    // 安全做法：加类判断防止父类被多次调用
    if (self == [MyClass class]) {
        // 第一次使用 MyClass 时执行的初始化逻辑
    }
}

// ===== 核心区别速查 =====
// 特性               +load                    +initialize
// 调用时机           main() 前，镜像加载时       类第一次收到消息时
// 调用方式           直接函数指针调用             objc_msgSend
// 是否必定调用       是（只要有实现）             否（类可能从未被使用）
// 父类优先           是                          是（但子类不实现时父类会被多调）
// 继承               不继承                      继承
// 线程               主线程                      任意线程（有锁保护）
// 适合              method swizzle / hook        懒初始化静态变量
```


![截屏2023-02-07 上午3.06.29.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae7b5f036e3a4e719b942d888f95aaa7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)
![截屏2023-02-06 下午10.48.12.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edf5053ea3a5436a9c25428627200920~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)