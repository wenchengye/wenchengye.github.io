---
title: Android 编译流程概览与编译入口源码分析
catalog: true
subtitle: Android 编译源码学习笔记 0x01
header-img: /img/header_img/platform-source-header.jpg
tags:
  - Android
categories:
  - 源码分析
---

## Why ?
目标：使 Android 编译过程由黑盒变为白盒。  
好处： 

* 理解 android plugin 接口含义和能力，更好的编写编译脚本。  
* 熟悉 android gradle 任务树，方便在编写 gradle 插件时 hook android 编译任务。 
* 看懂编译中间产物、以及熟练的调试编译源码，以便在开发时定位编译时发生的问题。 
* 了解 android 编译 toolChain 和 编译输出，是进一步学习 Android Runtime 的基础。 

## 行文
版本：源码基于 android plugin 3.0.0，gradle 4.1。   
主流程：   
首先，在本篇概览中会概述 Android 的编译流程，浏览 android plugin 的入口源码。   
而后，分别详述编译流程中的各个阶段 ：针对该编译阶段所涉及的 plugin api、project 模型、编译任务、toolChain、输入以及输出进行纵向分析，可能会浏览该阶段编译任务以及 toolChain 的源码。   
最后，会分析 application 工程和 library 工程在编译时的区别。   

副流程：   
Android 编译中涉及到了很多 Optional 的话题，这些话题会渗透到编译流程的各个阶段。将这些话题从主流程中分离出来，以切面的形式归纳为副流程分别单独学习，也许是更好的方式。   

副流程也许会包括 (排在越后面，会被包括的可能性越小)：

* Java8 support。
* Kotlin。
* AppBundle
* 单元测试与集成测试。
* Lint。
* Ndk。
* Instant Run。
* DataBinding。
* Wear & TV support。
* IDE。


## 结构

gradle 编译脚本 &  -> plugin api -> project 模型 -> 编译任务树 & Transtrom 流 -> toolChain -> 中间产物文件流 -> 编译输出

结构基础：gradle api、 gradle、android sdk。

这一篇会包含 plugin api -> project 模型 -> 编译任务树 & Transtrom 流 三个部分；之后在各个编译流程流程中会涉及到 编译任务树 & Transtrom 流  -> toolChain -> 中间产物文件流。   

## android plugin api 概览 

plugin api 是 android 编译系统暴露给开发者的配置接口，以基于 gradle & groovy 的 DSL (Domain-specific language) 组织而成。plugin api 可以大致被划分为 plugin、extension、dsl model 三个部分：

* plugin 是 android 编译系统的入口，开发者通过 apply plugin 来调用编译系统，进而完成暴露 plugin api，构建 project 模型，生成编译任务等一系列流程。
* extension 是承载 dsl model 的对象，它通过 delegate 的方式挂载到扩 gradle project 上，方便开发者通过对其成员赋值，从而完成编译配置过程。 
* dsl model 的接口繁多，是 plugin api 的主体，代表了 android 编译过程的各种配置参数。

### plugins

![Android-Gradle-Plugins](https://i.loli.net/2019/03/07/5c80c1f57f944.jpg)

`BasePlugin` 有多个继承类型，各个继承类型对应了不同类型的 android plugin，以支持不同的 android 编译类型。

* `AppPlugin` (com.android.application)，构建 app 工程。
* `LibraryPlugin` (com.android.library)，构建 library 工程。
* `FeaturePlugin` (com.android.feature)，构建 app-bundle。
* `InstantAppPlugin` (com.android.instantapp)，执行 intant-run。
* `TestPlugin` （com.android.test)，用单独工程执行 instrumented-test。

android plugin 绝大部分配置过程代码都包含在 `BasePlugin` 中，`BasePlugin` 的各个子类仅仅创建并返回了类型不同的 `BaseExtension`、`VariantFactory`、`TaskManager` 实现类对象。基于这些不同的实现对象，使各种编译类型在 plugin api、project 模型和编译任务等方面略有不同。

### extensions






## android 的 project 模型概览

## android 的编译任务树概览

## 在读源码之前

### 源码工程 -- :gradle 和 :gradle-core

:gradle 包含了 android plugin 链接到 gradle project 的 anchor 类，即一些 plugin 和 extension 类，这里就是 android plugin 的入口。    
:gradle-core 是 android plugin 的实现工程，里面包含了 project 模型的管理类、 构建任务树的管理类、任务的实现类等。 这些类被 :gralde 工程中的 plugin 调用，处理 开发者编译脚本向 extension 中提交的信息、建立 project 模型、创建 android task，并经过各类 android task 链接到 android 构建的 toolchain。    

### 源码阅读的起点 -- `BasePlugin.apply()`

`BasePlugin` 是所有 android plugin 的基类，`BasePlugin.apply()` 方法可以视为 android plugin 的程序入口。这个方法涵盖了 android plugin 配置阶段的全部过程， 以 `BasePlugin.apply()` 为起点可以了解 android plugin 工程模型的创建过程和任务树的创建过程。

## 源码分析

### `BasePlugin::apply()`

* 进行一系列的 validate 检查。
* 读取 gradle.properties 文件中所有与 android plugin 有关的配置，并根据这些配置初始化一些编译参数。
* 调用`configureProject()`、`configureExtension()`、`createTasks()` 三个方法；这三个方法就是 android plugin 配置阶段的主体操作。

```
 protected void apply(@NonNull Project project) {
        ...

        //AndroidBasePlugin 会被所有 plugin 类型 apply，它不做任何事情，只是方便判断是否有某种 android 的 plugin 类被 applay 过了。
        project.getPluginManager().apply(AndroidBasePlugin.class);
        
        // 检查 builder.jar 的版本号是否和 android plugin 的版本号一致，否则抛异常。
        // 如果使用了 'com.android.tools.build:gradle-experimental' 中的  gradle-experimental 插件，
        // 并且其版本和 'com.android.tools.build:gradle' 的版本不一致，
        // 那么就会出现 builder.jar 的版本号是否和 android plugin 的版本号不一致的情况。
        // TODO: what's gradle-experimental
        checkPluginVersion();

        this.project = project;

        // ProjectOptions 是一个 Immutable 模型对象，它包含了很多类型为 Option<String> 对象。
        // ProjectOptions 实际上是一个 key-value 形式的配置表，包含了 gradle.properties 文件中所有与 android plugin 有关的配置。
        // 各类配置被定义在 :gradle-core > com.android.build.gradle.options 包下面的枚举类中。
        // 这些配置大多以 'android.' 开头，详见附表部分。
        this.projectOptions = new ProjectOptions(project);

        // 由  {@GradleProperties 'android.threadPoolSize'} 配置来设置 ExecutorSingleton 的线程池尺寸。
        ExecutionConfigurationUtil.setThreadPoolSize(projectOptions);

        //在 window 平台上，gradle 工程 根目录的绝对路径上不能有非ASCII字符，否则就抛出异常，原因参见 http://b.android.com/95744；可以配置{@GradleProperties 'android.overridePathCheck'} = true 屏蔽这个检查。
        checkPathForErrors();

        //gradle 工程中，不能存在两个同名的子工程，否则就抛出异常。
        checkModulesForErrors();
        
        // PluginInitializer 用于验证两件事情：
        // 在一次构建中，不同子工程使用的 android plugin 版本一致。
        // 在一次构建中，android plugin 的 class 文件是被同一个 classloader 加载的。
        PluginInitializer.initialize(project, projectOptions);

        // ProfilerInitializer 通过向 gradle 注册监听，记录*当前子工程*所有 gradle task 的执行时间。
        // 当构建结束之后，将所记录的时间输出到 'project.getRootProject()/build/android-profile/profile-'YYYY-MM-dd-HH-mm-ss-SSS'.rawproto' 文件中。
        // TODO: 这个文件的作用是什么？猜测是给 IDE 用的。
        ProfilerInitializer.init(project, projectOptions);

        ...

        // 涵盖 BasePlugin 主体操作的三个 private 方法。 
        configureProject()
        configureExtension()
        createTasks()
    }
```

#### `PluginInitializer`

`PluginInitializer` 这个类比较有意思，它的作用是验证 android plugin 在各个子工程之间一致性。具体来说包含如下两方面验证：
* 在一次构建中，不同子工程使用的 android plugin 版本一致。
* 在一次构建中，android plugin 的 class 文件是被同一个 classloader 加载的。
那么问题来了，无论是不同的子工程使用了不同的 android plugin 版本，还是 android plugin 的 class 文件被不同的 classloader 加载，都会在 jvm 中形成两套相互独立的 class，要如何检测这种情况呢。
`PluginInitializer` 有两个静态变量分别负责上述两种检查，分别是 `projectToPluginVersionMap` 和 `loadedPluginClass`，这两个变量是通过 `JvmWideVariable` 获得的。`JvmWideVariable` 提供了一种直接在 jvm 创建对象的方式，并通过字符串 key 来获取对象引用。所以 `PluginInitializer` 的 `projectToPluginVersionMap` 和 `loadedPluginClass` 并不是和 class 绑定的静态对象，而是通过字符串 key 直接维护在 jvm 的堆中。只要不同版本的 android plugin 使用相同的 key 从 jvm 中创建和获取对象，那么即使是相互独立的 class 也能取得同一个对象，从而经由不同版本 android plugin 相互协作实现 `PluginInitializer` 所需验证的情况。
可以看见源码中对于 key 常量有如下注释：

>IMPORTANT: This variable's group, name, and type must not be changed across
plugin versions.

另外，什么情况下会发生 android plugin 被不同的 classloader 加载呢？源码的异常信息中给了如下解释，如果有兴趣可以看[issue链接](https://d.android.com/r/tools/buildscript-classpath-check.html)

> Due to a limitation of Gradle’s new variant-aware dependency management, loading the Android Gradle plugin in different class loaders leads to a build error.   
> This can occur when the buildscript classpaths that contain the Android Gradle plugin in sub-projects, or included projects in the case of composite builds, are set differently.   
> To resolve this issue, add the Android Gradle plugin to only the buildscript classpath of the top-level build.gradle file.   
> In the case of composite builds, also make sure the build script classpaths that contain the Android Gradle plugin are identical across the main and included projects.   
> If you are using a version of Gradle that has fixed the issue, you can disable this check by setting android.enableBuildScriptClasspathCheck=false in the gradle.properties file.   
> To learn more about this issue, go to https://d.android.com/r/tools/buildscript-classpath-check.html.   


```
public final class PluginInitializer {

    // Map<gradle project 对象 , android plugin 版本的> , 所用版本的 android plugin 共用一个对象，用于判断是否存在多个版本。
    // IMPORTANT: This variable's group, name, and type must not be changed across plugin versions.
    @NonNull
    private static final ConcurrentMap<Object, String> projectToPluginVersionMap =
            Verify.verifyNotNull(
                    new JvmWideVariable<>(
                                    
                                    "PLUGIN_VERSION_CHECK",
                                    "PROJECT_TO_PLUGIN_VERSION",
                                    new TypeToken<ConcurrentMap<Object, String>>() {},
                                    ConcurrentHashMap::new)
                            .get());

    // AndroidBasePlugin.class 的引用，对于同一个 android plugin 版本的多次加载而言，共用一个对象，用于判断同一个版本是否被多个 classloader 加载。
    @NonNull
    private static final AtomicReference<Class<?>> loadedPluginClass =
            Verify.verifyNotNull(
                    new JvmWideVariable<>(
                                    PluginInitializer.class.getName(),
                                    "loadedPluginClass",
                                    Version.ANDROID_GRADLE_PLUGIN_VERSION,
                                    new TypeToken<AtomicReference<Class<?>>>() {},
                                    () -> new AtomicReference<>(null))
                            .get());

    public static void initialize(
        
        // BuildSessionImpl 提供了一整次 gradle 构建结束的监听 (例如一次命令行输入全部执行完成)。
        // 这里注册监听，当一整次构建结束之后，重置 projectToPluginVersionMap 和 loadedPluginClass。
        // 由于 gradle 有进程重用机制，多次构建可能发生在同一个进程；只需要保证在一整次构建中，版本一致性，所以构建完成后重置状态。
        BuildSessionImpl.getSingleton().initialize(project.getGradle());
        BuildSessionImpl.getSingleton()
                .executeOnceWhenBuildFinished(
                        PluginInitializer.class.getName(),
                        "resetPluginCheckVariables",
                        () -> {
                            projectToPluginVersionMap.clear();
                            loadedPluginClass.set(null);
                        });

        // 向 projectToPluginVersionMap 中写入当前工程对象和现在所执行的 android plugin 版本，如果 Map 中有两个不一样的 android plugin 版本，那么就抛出异常。
        synchronized (projectToPluginVersionMap) {
            verifySamePluginVersion(
                    projectToPluginVersionMap, project, Version.ANDROID_GRADLE_PLUGIN_VERSION);
        }

        // 向 loadedPluginClass 写入现在所执行的 android plugin 的 AndroidBasePlugin.class 对象，如果 loadedPluginClass 保存了和写入的 AndroidBasePlugin.class 对象不同的值，则抛出异常。
        // 配置 {@GradleProperties 'android.enableBuildScriptClasspathCheck'} = false 来屏蔽这个检查。
        verifyPluginLoadedOnce(
                loadedPluginClass,
                AndroidBasePlugin.class,
                projectOptions.get(BooleanOption.ENABLE_BUILDSCRIPT_CLASSPATH_CHECK));
    }

```

### `BasePlugin::apply() > configureProject()`

`configureProject()` 做了一些零散的不易归类初始化任务，这些任务中需要进一步分析的也不多。

```
private void configureProject() {

    // ExtraModelInfo 存储一些不易归类的 extra 信息 ：
    // * 存储构建过程中的检测到的 waring 和 error 信息，并输出这些信息, 根据这次构建是由 ide 触发的还是开发者触发的，信息可能以 'MACHINE_PARSABLE' 或 'HUMAN_READABLE' 两种格式输出。
    // TODO: 在后续源码分析中，补充在 ExtraModelInfo 其他信息的作用
    extraModelInfo = new ExtraModelInfo(projectOptions, project.getLogger());

    // 检查当前 gradle 版本不低于 当前运行的 android plugin 所要求的最低版本，否则抛出异常。
    // 配置 {@GradleProperties 'android.overrideVersionCheck'} = true，可以仅输出 waring 信息不抛出异常。
    checkGradleVersion();
    
    // SdkHandler 是 android sdk 信息的封装类，在构造方法中会寻找 andorid sdk 的路径。
    sdkHandler = new SdkHandler(project, getLogger());

    // 当 {@GradleProperties 'android.builder.sdkDownload'} = true、并且当前构建不是由 IDE 触发、并且当前构建的 gradle 建参数不包含 '--offline' 这个三个条件成立时：
    // SdkHandler 会在使用 android sdk 中工具时自动下载，这样即使不安装 android sdk 也能完成 android 工程构建。
    //  SdkLibData 时下载信息的抽象，包括 Downloader 对象和 SettingsController 的实现。
    // SettingsController 是注入下载配置的接口，默认实现中包括了指定使用 https、从{@GradleProperties 'android.sdk.channel'} 中获取下载channel、从 System.getProperties() 获取网络代理信息等。
    if (!project.getGradle().getStartParameter().isOffline() 
                && projectOptions.get(BooleanOption.ENABLE_SDK_DOWNLOAD)
                && !projectOptions.get(BooleanOption.IDE_INVOKED_FROM_IDE)) {
            SdkLibData sdkLibData = SdkLibData.download(getDownloader(), getSettingsController());
            sdkHandler.setSdkLibData(sdkLibData);
    }  

    // AndroidBuilder 是构建 toolchain 的接口类。
    androidBuilder = new AndroidBuilder(
                project == project.getRootProject() ? project.getName() : project.getPath(),
                creator,
                new GradleProcessExecutor(project),
                new GradleJavaProcessExecutor(project),
                extraModelInfo,
                getLogger(),
                isVerbose());

    // @DataBinding 初始化 DataBindingBuilder。
    [+] {...}

    // apply 了 JavaBasePlugin 插件和 JacocoPlugin 插件
    project.getPlugins().apply(JavaBasePlugin.class);
    project.getPlugins().apply(JacocoPlugin.class);

    // 替换 {@Task assemble} 的描述，{@Task assemble} 在 apply JavaBasePlugin.class 之后创建。
    project.getTasks().getByName("assemble")
        .setDescription("Assembles all variants of all applications and secondary packages.");

    project.getGradle().addBuildListener(
        buildFinished(BuildResult buildResult) -> {
            // 构建结束后一些 release 工作。
            ExecutorSingleton.shutdown();
            sdkHandler.unload();

            // 清理 PreDexCache 中的缓存，并将其写在 '{@File gradleProjectRoot}/build/intermediates/dex-cache/cache.xml' 文件
            // TOOD : 分析 Dex 过程时在回来看这里。
            PreDexCache.getCache().clear(
                FileUtils.join(
                    project.getRootProject().getBuildDir(),
                    FD_INTERMEDIATES,
                    "dex-cache",
                    "cache.xml"),
                getLogger());
                Main.clearInternTables();
        }
    )

    // 任务树生成后进行检查：如果任务树中存在 DexTransform 类型的任务，
    // PreDexCache 从 '{@File gradleProjectRoot}/build/intermediates/dex-cache/cache.xml' 文件加载缓存。
    // TOOD : 分析 Dex 过程时在回来看这里。
    project.getGradle().getTaskGraph().addTaskExecutionGraphListener(
        (taskGraph) -> {
            for (Task task : taskGraph.getAllTasks()) {
                if (task instanceof TransformTask) {
                    Transform transform = ((TransformTask) task).getTransform();
                    if (transform instanceof DexTransform) {
                        PreDexCache.getCache().load(
                            FileUtils.join(
                                project.getRootProject().getBuildDir(),
                                FD_INTERMEDIATES,
                                dex-cache",
                                "cache.xml"));
                        break;
                    }
                }
            }
        }
    }
```

#### `SdkHandler` 寻找 android sdk 路径

```
public class SdkHandler {

    public SdkHandler(@NonNull Project project,
                      @NonNull ILogger logger) {
        this.logger = logger;
        findLocation(project);
    }

    private void findLocation(@NonNull Project project) {
        
        //@Test 
        [+] {...}

        //读取 '{@File gradleProjectRoot}/local.properties'
        File rootDir = project.getRootDir();
        File localProperties = new File(rootDir, FN_LOCAL_PROPERTIES);
        Properties properties = new Properties();
        [+] {...}

        // 按照如下优先级寻找 android sdk 路径：
        // local.properties 的 'sdk.dir' 属性 > local.properties 的 'android.dir' 属性 > 
        // ANDROID_HOME 环境变量 > android.home 环境变量。
        Pair<File, Boolean> sdkLocation = findSdkLocation(properties, rootDir);
        sdkFolder = sdkLocation.getFirst();
        
        // isRegularSdk 的作用见下面 getSdkLoader() 方法前
        isRegularSdk = sdkLocation.getSecond();

        //按照如下优先级寻找 ndk路径
        // local.properties 的 'ndk.dir' 属性 > ANDROID_NDK_HOME 环境变量
        // > '{@File sdkFolder}/ndk-bundle'
        ndkFolder = NdkHandler.findNdkDirectory(properties, rootDir);

        // @Ndk 从 local.properties 中读出 'cmake.dir'，并赋值给 cmakePathInLocalProp 属性(File 类型)。
        [+] {...}
    }


    // 仅当 android sdk 路径是由 local.properties 的 'android.dir' 属性定义时，isRegularSdk 为 false。
    // 这时认为当前构建使用 "Platform-based" 的 android sdk，其文件布局与开发使用的 android sdk 不同，
    // 所以在解析 android sdk 文件布局时使用 PlatformLoader 而非 DefaultSdkLoader。
    // 在依赖 android 源码中的 android sdk 而非发布版的 android sdk 会出现这种情况。
    public synchronized SdkLoader getSdkLoader() {
        if (sdkLoader == null) {
            if (isRegularSdk) {
                sdkLoader = DefaultSdkLoader.getLoader(sdkFolder);
            } else {
                sdkLoader = PlatformLoader.getLoader(sdkFolder);
            }
        }
        return sdkLoader;
    }
}
```

### `BasePlugin::apply() > configureExtension()`

* 创建了 extension 对象并将其注册到 gradle project 中，extension 对象包含 android plugin 和用户构建脚本之间的所有 api 对象，这些 api 对象定义的 android plugin api 并承载了构建脚本中的开发者配置。
* 初始化 project 模型和任务树有关的管理类：TaskManager、VariantManager、VariantFactory，这3个管理类可以说是配置阶段的核心；这些管理类通过注册 api 对象监听事件，收集开发者构建脚本中的配置信息。
* 创建 FileCache、GlobalScope 等一些上下文对象。

完成 `configureExtension()` 执行后 android plugin 就已经做好执行构建脚本，获取配置信息的准备了。

```
private void configureExtension() {

    // 创建了承载 BuildType、ProductFlavor、SigningConfig 的3个容器。
    // 这3个容器作为创建 extension 的参数，通过 extension 链接到编译脚步，收集开发者配置。
    final NamedDomainObjectContainer<BuildType> buildTypeContainer =
            project.container(BuildType.class,
                    new BuildTypeFactory(instantiator, project, extraModelInfo));
    final NamedDomainObjectContainer<ProductFlavor> productFlavorContainer =
            project.container(ProductFlavor.class,
                    new ProductFlavorFactory(
                            instantiator, project, project.getLogger(), extraModelInfo));
    final NamedDomainObjectContainer<SigningConfig> signingConfigContainer =
            project.container(SigningConfig.class, new SigningConfigFactory(instantiator));

    // TODO：BaseVariantOutput 
    final NamedDomainObjectContainer<BaseVariantOutput> buildOutputs =
            project.container(BaseVariantOutput.class);
    project.getExtensions().add("buildOutputs", buildOutputs);

    // 创建 extension 对象，可以看到之前创建的4个容器最终由 extension 对象持有。
    // 创建 extension 的过程会完成以下工作：
    // * 完成 andorid plugin 所有 api 对象的创建 ( DebugConfig 对象、SourceSet 容器、AaptOptions 对象、DexOptions 对象等）；
    // * 通过对 SourceSet 容器的监听调用 gradle api ，创建 Configuration 对象；
    // *为 android plugin api 对象添加默认配置。
    extension =
            createExtension(
                    project,
                    projectOptions,
                    instantiator,
                    androidBuilder,
                    sdkHandler,
                    buildTypeContainer,
                    productFlavorContainer,
                    signingConfigContainer,
                    buildOutputs,
                    extraModelInfo);

    // @Ndk 创建 NdkHandler。
    [+] {...}

    // 创建 FileCache 和 GlobalScope 对象，FileCache 被 GlobalScope 对象持有。
    
    // 当 '{@GradleProperties android.enableBuildCache}' = true 时创建 FileCache。
    // FileCache 默认缓存路径是 '{@File androidHomeDir}/build-cache'，可以通过 '{@GradleProperties android.buildCacheDir}' 自定义缓存路径。
    // {@File androidHomeDir} 是选取规则在见后面设置 debug.keystore 的代码分析。
    // FileCache 的实际作用留到它被使用时再分析。
    @Nullable
    FileCache buildCache = BuildCacheUtils.createBuildCacheIfEnabled(project, projectOptions);

    // GlobalScope 对象是当前 android 工程的上下文对象，它只负责持有对象，方便获取。
    // GlobalScope 对象 之后分别被 TaskManager、VariantManager、VariantFactory 3个对象持有。
    GlobalScope globalScope =
            new GlobalScope(
                    project,
                    projectOptions,
                    androidBuilder,
                    extension,
                    sdkHandler,
                    ndkHandler,
                    registry,
                    buildCache);

    // 创建 TaskManager、VariantManager、VariantFactory 3个管理类。
    variantFactory = createVariantFactory(globalScope, instantiator, androidBuilder, extension);
    taskManager =
            createTaskManager(
                    globalScope,
                    project,
                    projectOptions,
                    androidBuilder,
                    dataBindingBuilder,
                    extension,
                    sdkHandler,
                    ndkHandler,
                    registry,
                    threadRecorder);
    variantManager =
            new VariantManager(
                    globalScope,
                    project,
                    projectOptions,
                    androidBuilder,
                    extension,
                    variantFactory,
                    taskManager,
                    threadRecorder);


    // 为 BuildType、ProductFlavor、SigningConfig 的3个容器注册 Add 事件监听。
    signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);
    buildTypeContainer.whenObjectAdded(
            buildType -> {
                SigningConfig signingConfig =
                        signingConfigContainer.findByName(BuilderConstants.DEBUG);
                buildType.init(signingConfig);
                variantManager.addBuildType(buildType);
            });
    productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);

    // 为 BuildType、ProductFlavor、SigningConfig 的3个容器注册 remove 事件监听。
    // 禁止开发者对这三个配置做 remove 操作，如果发生就抛出异常。
    signingConfigContainer.whenObjectRemoved(
            new UnsupportedAction("Removing signingConfigs is not supported."));
    buildTypeContainer.whenObjectRemoved(
            new UnsupportedAction("Removing build types is not supported."));
    productFlavorContainer.whenObjectRemoved(
            new UnsupportedAction("Removing product flavors is not supported."));

    // 向 BuildType、ProductFlavor、SigningConfig 3个容器创建 default 成员。
    // 大多数 VariantFactory 的实现，在这里创建了 'debug' SigningConfig、'debug' BuildType、'release' BuildType。
    variantFactory.createDefaultComponents(
            buildTypeContainer, productFlavorContainer, signingConfigContainer);
}
```

#### `BuildType`、`ProductFlavor`、`SigningConfig` 容器内容构造

这3个 NamedDomainObjectContainer<T> 类型的容器，在构造时传入 NamedDomainObjectFactory<T> 提供容器内容对象的构造方法。`BuildType` 和 `ProductFlavor` 构造方法都直接调用了构造函数，只有 `SigningConfig` 做了特殊处理。

```
public class SigningConfigFactory implements NamedDomainObjectFactory<SigningConfig> {

    ...

    @NonNull
    public SigningConfig create(@NonNull String name) {
        SigningConfig signingConfig = instantiator.newInstance(SigningConfig.class, name);

        // 如果 SigningConfig 对象的名字是 'debug'，那么用 android 环境的 debug keystore 来初始化这个对象。
        // debug keystore 存放的位置在 '{@File androidHomeDir}/debug.keystore'；
        // debug keystore 配置：store pw -> 'android'，key alias -> 'AndroidDebugKey'，key pw -> 'android'。
        // {@File androidHomeDir} 为 '{@File homeDir}/.android' ，{@File homeDir} 按以下优先级寻找, ，没找到就会抛异常：
        // ANDROID_SDK_HOME 环境变量或系统属性 > TEST_TMPDIR 环境变量 > user.home 系统属性 > HOME 环境变量 
        if (BuilderConstants.DEBUG.equals(name)) {
            try {
                signingConfig.initWith(
                        DefaultSigningConfig.debugSigningConfig(
                                new File(KeystoreHelper.defaultDebugKeystoreLocation())));
            } catch (AndroidLocation.AndroidLocationException e) {
                throw new BuildException("Failed to get default debug keystore location.", e);
            }
        }
        return signingConfig;
    }

    ...
}
```

### `BuildType`、`ProductFlavor`、`SigningConfig` Add 操作监听

总体来说，VariantManager 监听了 `BuildType`、`ProductFlavor`、`SigningConfig` 对象的创建操作，并将被创建的对象分别保存在 map 容器中。

```
private void configureExtension() {
    ...

    // VariantManager.addSigningConfig() 仅仅将 SigningConfig 对象加入了 VariantManager 的 map 容器持有。
    signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);

    // 这里看起来好像用 'debug' SigningConfig 初始化了所有 BuildType 对象。
    // 实际上 BuildType.init(SigningConfig) 中有判断，只有 BuildType 的名字也是 'debug' 时才会用调用 BuildType.setSigningConfig(SigningConfig)。
    buildTypeContainer.whenObjectAdded(
            buildType -> {
                SigningConfig signingConfig =
                        signingConfigContainer.findByName(BuilderConstants.DEBUG);
                buildType.init(signingConfig);
                variantManager.addBuildType(buildType);
            });
    productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);

    ...
}

```

```
public class VariantManager implements VariantModel {
    ...

    // BuildType 对象的加入引发 SourceSet 和 BuildTypeData 对象的创建。
    public void addBuildType(@NonNull CoreBuildType buildType) {
        String name = buildType.getName();

        // 检查 BuildType 名字的合法性，名字不能以 'androidTest' 或 'test' 开头；名字不能为 'lint'。
        checkName(name, "BuildType");

        // BuildType 名字不能和任何 ProductFlavor 相同。
        if (productFlavors.containsKey(name)) {
            throw new RuntimeException("BuildType names cannot collide with ProductFlavor names");
        }

        // 创建和 BuildType 同名的 SourceSet。
        DefaultAndroidSourceSet mainSourceSet = (DefaultAndroidSourceSet) extension.getSourceSets().maybeCreate(name);

        // @Test，创建名为 '{$BuildTypeName}AndroidTest' 和 '{$BuildTypeName}UnitTest' 的 SourceSet。
        [+] {...}

        // 创建 BuildTypeData，其持有 BuildType 对象和 3个刚刚创建 SourceSet。
        BuildTypeData buildTypeData =
                new BuildTypeData(
                        buildType, project, mainSourceSet, androidTestSourceSet, unitTestSourceSet);
        buildTypes.put(name, buildTypeData);
    }


    public void addProductFlavor(@NonNull CoreProductFlavor productFlavor) {
        String name = productFlavor.getName();

        // 检查 BuildType 名字的合法性，名字不能以 'androidTest' 或 'test' 开头；名字不能为 'lint'。
        checkName(name, "ProductFlavor");

        // ProductFlavor 名字不能和任何 BuildType 相同。
        if (buildTypes.containsKey(name)) {
            throw new RuntimeException("ProductFlavor names cannot collide with BuildType names");
        }

        // 创建和 ProductFlavor 同名的 SourceSet。
        DefaultAndroidSourceSet mainSourceSet = (DefaultAndroidSourceSet) extension.getSourceSets().maybeCreate(
                productFlavor.getName());

        // @Test，创建名为 '{$BuildTypeName}AndroidTest' 和 '{$BuildTypeName}UnitTest' 的 SourceSet。
        [+] {...}

        // 创建 ProductFlavorData，其持有 ProductFlavor 对象和 3个刚刚创建 SourceSet。
        ProductFlavorData<CoreProductFlavor> productFlavorData =
                new ProductFlavorData<>(
                        productFlavor,
                        mainSourceSet,
                        androidTestSourceSet,
                        unitTestSourceSet,
                        project);
        productFlavors.put(productFlavor.getName(), productFlavorData);
    }

    ...
}

```

#### `BuildTypeData`、`ProductFlavorData` 的创建

BuildTypeData、ProductFlavorData 的基类均为 VariantDimensionData，在导出类中没有特殊操作，仅仅额外持有了 BuildType 和 ProductFlavor 对象，生成任务树之后会额外持有 {@Task assemble{$BuildType}} 和 {@Task assemble{$ProductFlavor}} 的引用。

```
public class VariantDimensionData {
    ...

    private final DefaultAndroidSourceSet sourceSet;
    private final DefaultAndroidSourceSet androidTestSourceSet;
    private final DefaultAndroidSourceSet unitTestSourceSet;

    public VariantDimensionData(
            @NonNull DefaultAndroidSourceSet sourceSet,
            @Nullable DefaultAndroidSourceSet androidTestSourceSet,
            @Nullable DefaultAndroidSourceSet unitTestSourceSet,
            @NonNull Project project) {
        this.sourceSet = sourceSet;
        this.androidTestSourceSet = androidTestSourceSet;
        this.unitTestSourceSet = unitTestSourceSet;

        final ConfigurationContainer configurations = project.getConfigurations();

        // @Test 
        // 让名为 '{$VariantDimensionName}AndroidTestImplementation' 和 '{$VariantDimensionName}UnitTestImplementation' 的 Configuration 继承 '{$VariantDimensionName}Implementation'
        // 让名为 '{$VariantDimensionName}AndroidTestRuntimeOnly' 和 '{$VariantDimensionName}UnitTestRuntimeOnly' 的 Configuration 继承 '{$VariantDimensionName}RuntimeOnly'
        if (androidTestSourceSet != null) {
            makeTestExtendMain(sourceSet, androidTestSourceSet, configurations);
        }
        if (unitTestSourceSet != null) {
            makeTestExtendMain(sourceSet, unitTestSourceSet, configurations);
        }

    }

    ...
}
```

#### `BasePlugin::configureExtension() > createExtension()`

`createExtension()` 是个抽象函数，不过实际上 BasePlugin 的各个实现类只是构造了不同的 BaseExtension 类的实现，没有在这个方法中做别的事情。
各个 BaseExtension 类的实现区别也很小，所以这里只会提到 BaseExtension 和 TestedExtension。

BaseExtension
    -> TestExtension
    -> TestedExtension
        -> AppExtension
        -> LibraryExtension
            -> FeatureExtension
    -> InstantAppExtension 

在 BaseExtension 的构造函数中：
* 完成 andorid plugin 所有 api 对象的创建 ( DebugConfig 对象、SourceSet 容器、AaptOptions 对象、DexOptions 对象等）；
* 通过对 SourceSet 容器的监听调用 gradle api ，创建 Configuration 对象；
* 为 android plugin api 对象添加少许默认配置。

*创建 Configuration 的过程使用伪代码展现*

```
public abstract class BaseExtension implements AndroidConfig {
    ...

    BaseExtension(
            @NonNull final Project project,
            @NonNull final ProjectOptions projectOptions,
            @NonNull Instantiator instantiator,
            @NonNull AndroidBuilder androidBuilder,
            @NonNull SdkHandler sdkHandler,
            @NonNull NamedDomainObjectContainer<BuildType> buildTypes,
            @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavors,
            @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigs,
            @NonNull NamedDomainObjectContainer<BaseVariantOutput> buildOutputs,
            @NonNull ExtraModelInfo extraModelInfo,
            final boolean publishPackage) {
        
        // @Simplify 保存构造函数传入的参数。
        [+] {...}

        logger = Logging.getLogger(this.getClass());

        // 创建 DefaultConfig 对象，DefaultConfig 对象是一个名为 'main' 的特殊的 ProductFlavor，DefaultConfig 和 ProductFlavor 有共同的父类。
        defaultConfig =
                instantiator.newInstance(
                        DefaultConfig.class,
                        BuilderConstants.MAIN,
                        project,
                        instantiator,
                        project.getLogger(),
                        extraModelInfo);

        // @Simplify 创建各种 Options 类型对象，都用于描述 android plugin api 并承载编译脚本配置。
        [+] {...}

        // 创建 SourceSet 容器，容器中的使用 DefaultAndroidSourceSet 类构造。
        sourceSetsContainer =
                project.container(
                        AndroidSourceSet.class,
                        new AndroidSourceSetFactory(instantiator, project, publishPackage));

        
        // 监听 SourceSet 容器的 Add 操作。
        // 当有新的 SoucreSet 加入容器之后，初始化 SourceSet 的默认布局，建与之相对应的 Configuration，设置被创建的 Configuration 之间的 extendsFrom 关系。
        // 这是段较长的 labor 代码，改为用伪代码分析
        sourceSetsContainer.whenObjectAdded() {
            // @Pseudocode
            sourceSet -> {
                ConfigurationContainer configurations = project.getConfigurations();

                String sourceSetName = sourceSet.getName();

                // 创建名为 '{SouceSetName}Api'、'{SouceSetName}Implementation'、
                // '{SouceSetName}RuntimeOnly'、'{SouceSetName}CompileOnly' 的 SourceSet。
                // 创建名为 '{SouceSetName}Compile'、'{SouceSetName}Provided' 的 SourceSet。
                // 在 App 工程中创建 '{SouceSetName}Apk'；在 Lib 工程中创建 '{SouceSetName}Publish'。
                // 对于名为 'main' 的 sourceSet 而言，创建 Configuration 的命名规则特殊处理，不加前缀。
                Configuration compile = createConfiguration(configurations, sourceSetName + 'Compile');
                if ({in AppExtension}) {
                    Configuration apk = createConfiguration(configurations, sourceSetName + 'Apk');
                } else {
                    Configuration apk = createConfiguration(configurations, sourceSetName + 'Publish');
                }
                Configuration provided = createConfiguration(configurations, sourceSetName + 'Provided');
                Configuration api = createConfiguration(configurations, sourceSetName + 'Api');
                Configuration implementation = createConfiguration(configurations, sourceSetName + 'Implementation');
                Configuration runtimeOnly = createConfiguration(configurations, sourceSetName + 'RuntimeOnly');
                Configuration compileOnly = createConfiguration(configurations, sourceSetName + 'CompileOnly');

                // compile、apk、provided 3个 Configuration 属于 Depercated 设定。
                // 当这3个 Configuration 有任何 dependency 增加时输出 warning。
                compile.getAllDependencies().whenObjectAdded(
                    new DeprecatedConfigurationAction());
                apk.getAllDependencies().whenObjectAdded(
                    new DeprecatedConfigurationAction());
                provided.getAllDependencies().whenObjectAdded(
                    new DeprecatedConfigurationAction());

                // 设置 extendsFrom 关系。
                api.extendsFrom(compile);
                implementation.extendsFrom(api);
                runtimeOnly.extendsFrom(apk);
                compileOnly.extendsFrom(provided);

                // @Wear 创建名为 '{SouceSetName}WearApp' 的 Configuration。
                [+] {...}

                // '{SouceSetName}AnnotationProcessor' 的 Configuration。
                // 用于做 apt。
                createConfiguration(configurations, sourceSetName + 'AnnotationProcessor');

                // 设置 sourceSet 的默认布局：
                // java -> 'src/{SouceSetName}/java'
                // javaResources -> 'src/{SouceSetName}/resources'
                // res -> 'src/{SouceSetName}/res'
                // assets -> 'src/{SouceSetName}/assets'
                // manifest -> 'src/{SouceSetName}/AndroidManifest.xml'
                // aidl -> 'src/{SouceSetName}/aidl'
                // renderscript -> 'src/{SouceSetName}/rs'
                // jni -> 'src/{SouceSetName}/jni'
                // jniLibs -> 'src/{SouceSetName}/jniLibs'
                // shaders -> 'src/{SouceSetName}/shaders'
                sourceSet.setRoot(String.format("src/%s", sourceSet.getName()));
            }
        }

        // @Test 创建名为 'androidTestUtil' 的 Configuration
        [+] {...}
        
        // 创建名为 'main' 的 SourceSet。
        sourceSetsContainer.create(defaultConfig.getName());

        // 设置一些默认的编译脚本配置。
        // 设置默认的 buildTools Revision，如果开发者没有在编译脚本中指定，就使用默认版本。不同版本的 android plugin 定一个默认 buildTools 版本不同。
        buildToolsRevision = AndroidBuilder.DEFAULT_BUILD_TOOLS_REVISION;
        // 这个函数会对 DefaultConfig 对象做一些有关矢量图的配置。 
        setDefaultConfigValues();
    }

    ...
}
```

在 TestedExtension 的构造函数中创建了名为 'androidTest' 和 'test' 的 SourceSet，与 'main' SourceSet 对应是默认存在基础 SourceSet。

```
// TestedExtension 是 AppExtension、LibraryExtension、FeatureExtension 的父类。
public abstract class TestedExtension extends BaseExtension implements TestedAndroidConfig {

    ...
    public TestedExtension(
            @NonNull Project project,
            @NonNull ProjectOptions projectOptions,
            @NonNull Instantiator instantiator,
            @NonNull AndroidBuilder androidBuilder,
            @NonNull SdkHandler sdkHandler,
            @NonNull NamedDomainObjectContainer<BuildType> buildTypes,
            @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavors,
            @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigs,
            @NonNull NamedDomainObjectContainer<BaseVariantOutput> buildOutputs,
            @NonNull ExtraModelInfo extraModelInfo,
            boolean isDependency) {
        super(
                project,
                projectOptions,
                instantiator,
                androidBuilder,
                sdkHandler,
                buildTypes,
                productFlavors,
                signingConfigs,
                buildOutputs,
                extraModelInfo,
                isDependency);
        // 创建名为 'androidTest' 和 'test' 的 SourceSet。
        getSourceSets().create(ANDROID_TEST.getPrefix());
        getSourceSets().create(UNIT_TEST.getPrefix());
    }

    ...
}
```

#### 构造 `TaskManager`、`VariantManager`、`VariantFactory`

TaskManager 通过 `createTaskManager()` 方法创建，VariantFactory 通过 `createVariantFactory` 方法创建。这两个方法均为抽象方法，在各种 plugin 的实现中，仅仅是调用了各种 TaskManager 和 VariantFactory 实现的构造函数，并无额外的代码。

VariantManager 的构造函数中针对 DebugConfig 对象(DebugConfig 对象在 BaseExtension 构造时被创建)做了一些处理。


```
public class VariantManager implements VariantModel {
    ...

    public VariantManager(
            @NonNull GlobalScope globalScope,
            @NonNull Project project,
            @NonNull ProjectOptions projectOptions,
            @NonNull AndroidBuilder androidBuilder,
            @NonNull AndroidConfig extension,
            @NonNull VariantFactory variantFactory,
            @NonNull TaskManager taskManager,
            @NonNull Recorder recorder) {
        ...

        DefaultAndroidSourceSet mainSourceSet =
                (DefaultAndroidSourceSet) extension.getSourceSets().getByName(extension.getDefaultConfig().getName());

        DefaultAndroidSourceSet androidTestSourceSet = null;
        DefaultAndroidSourceSet unitTestSourceSet = null;
        if (variantFactory.hasTestScope()) {
            androidTestSourceSet =
                    (DefaultAndroidSourceSet) extension.getSourceSets()
                            .getByName(ANDROID_TEST.getPrefix());
            unitTestSourceSet =
                    (DefaultAndroidSourceSet) extension.getSourceSets()
                            .getByName(UNIT_TEST.getPrefix());
        }

        // 使用 DebugConfig 对象，'main'、'androidTest'、'test' 3个 SourceSet，
        // 创建了 defaultConfigData 对象。
        // 'main'、'androidTest'、'test' 3个 SourceSet 都是在 extension 中直接创建的，而不是由 BuildType 或 ProductFlavor 触发生成的。
        // 正如 DebugConfig 对象是一个特殊的 ProductFlavor，defaultConfigData 引用也持有了一个特殊的 ProductFlavorData。

        this.defaultConfigData =
                new ProductFlavorData<>(
                        extension.getDefaultConfig(),
                        mainSourceSet,
                        androidTestSourceSet,
                        unitTestSourceSet,
                        project);
    }

    ...
}

```

### BasePlugin::apply() > createTasks()

运行到这里时，创建管理类、监听链建立、设置默认配置布局这些工作都完成了，终于要创建 gradle task 了。

```
private void createTasks() {
    taskManager.createTasksBeforeEvaluate(
        new TaskContainerAdaptor(project.getTasks())));

    project.afterEvaluate(
        project ->
            () -> createAndroidTasks(false);
```

`createTasks()` 将创建 gradle task 的工作分为两部分，一部分在 evaluate 开始之前，用于创建不构建脚本配置影响的任务；另一部分在 evaluate 结束之后，这部分创建那些的任务取决于构建脚本配置。

#### `TaskManager::createTasksBeforeEvaluate()`

TaskManager 的绝大部分实没有扩展 `createTasksBeforeEvaluate()` (只有 InstantAppTaskManager 创建了多创建了一个 gradle task)，所以对于各类 android plugin 在这里创建的任务都是相同的。

在这个方法中创建了一些 Anchor 任务（没有实际内容，仅作为上游节点使用），和一些独立任务例如和 Lint 和 单元测试相关的任务。

*下面代码使用伪代码展现*

```
public abstract class TaskManager {
    public void createTasksBeforeEvaluate(@NonNull TaskFactory tasks) {

        create "uninstallAll" task as anchor task. 
            it "Uninstall all applications."

        create "deviceCheck" task as anchor task.
            it "Runs all device checks using Device Providers and Test Servers."

        create "connectedCheck" task as anchor task.
            it "Runs all device checks on currently connected devices."

        create "preBuild" task as anchor task.
            it "Lead all build tasks."

        create "extractProguardFiles" task as ExtractProguardFiles.class type.
            let "extractProguardFiles" dependsOn "preBuild"

        create "sourceSets" task as SourceSetsTask.class type.
            it "Prints out all the source sets defined in this project."

        create "assembleAndroidTest" task as anchor task.
            it "Assembles all the Test applications."

        create "compileLint" task as LintCompile.class type.

        create "lint" task as LintGlobalTask.class type.
            it "Runs lint on all variants."
            let "check" task (which from java plugin) dependsOn "lint" task.

        create "lintChecks" configuration.
            it "Configuration to apply external lint check jar"
            GlobalScope will hold "lintChecks" configuration

        if ({@Field buildCache} is not null) {
            create "cleanBuildCache" task as CleanBuildCache.class type.
                it "Deletes the build cache directory."
        }

        create "resolveConfigAttr" task as ConfigAttrTask.class type.
            set resolveConfigAttr.resolvable = true

        create "consumeConfigAttr" task as ConfigAttrTask.class type.
            set consumeConfigAttr.consumable true.
    }
}
```

#### `BasePlugin::createAndroidTasks()`

`BasePlugin::createAndroidTasks()` 是创建 BuildVariant 和 Android 编译 task 的主流程，其中包含了一些检查和准备工作，创建 task 的代码主要包含在 `VariantManager::createAndroidTasks` 中。

在这里主要做完成完成了：
* 加载 Android Sdk 信息和添加 Android Sdk 提供的 Maven Repo
* 创建 lint 相关的全局 task
* 基于编译脚本输入创建 VariantData 对象
* 基于 VariantData 信息创建编译 task
* 完成 Variant 对象的创建。

```
final void createAndroidTasks(boolean force) {

    // @Simplify
    // 确保 buildToolsVersion 和 compileSdkVersion 被设置了。
    // 确保 JavaPlugin.class 没有被 apply 过。
    [+]{...}

    // 初始化 Android SDK Target 信息。 
    ensureTargetSetup();

    if (hasCreatedTasks) {
        return;
    }
    hasCreatedTasks = true;
    // 禁止再修改 Extension 对象，编译脚本配置到此截止。
    extension.disableWrite();

    // 创建 PrepareLintJar.class 类型的 "prepareLintJar" 任务。
    taskManager.configureCustomLintChecks(new TaskContainerAdaptor(project.getTasks()));

    // 将 Android SDK 包含的 Repositories 加入的 Maven Repo 中，并将这些 Repositories 调整到 Maven Repo 列表的最前面。
    sdkHandler.addLocalRepositories(project);

    // @DataBinding，如果使用了 DataBinding，则向已有 configuration 添加一些依赖。
    [+]{...}

    // 创建 VariantData 对象和编译 Task。
    variantManager.createAndroidTasks();

    // 依照 VariantData 对象，创建 Variant 对象。
    ApiObjectFactory apiObjectFactory =
            new ApiObjectFactory(
                    androidBuilder,
                    extension,
                    variantFactory,
                    instantiator,
                    project.getObjects());
    for (VariantScope variantScope : variantManager.getVariantScopes()) {
        BaseVariantData variantData = variantScope.getVariantData();
        apiObjectFactory.create(variantData);
    }

    // 在 Variant 对象创建完成之后，创建全局 "lint" Task。
    taskManager.configureGlobalLintTask(variantManager.getVariantScopes());

    // @IDE
    [+]{...}
}

...

private void ensureTargetSetup() {    
    TargetInfo targetInfo = androidBuilder.getTargetInfo();
    if (targetInfo == null) {
        if (extension.getCompileOptions() == null) {
            throw new GradleException("Calling getBootClasspath before compileSdkVersion");
        }

        // 获取 Android SDK Target 信息，并将其设置到 androidBuiler 中。
        sdkHandler.initTarget(
                extension.getCompileSdkVersion(),
                extension.getBuildToolsRevision(),
                extension.getLibraryRequests(),
                androidBuilder,
                SdkHandler.useCachedSdk(projectOptions));
        // 确保 Android SDK 中没有安装了 platform tools，如果没有就下载它。
        sdkHandler.ensurePlatformToolsIsInstalled(extraModelInfo);
    }
}
```

`SdkHandler::initTarget()` 中通过 SdkLoader 对象加载 Android Sdk 中的各种工具路径。
SdkLoader 的不同实现对应不同的 Android Sdk 文件布局，通常使用 `DefaultSdkLoader` 。

```
class SdkHandler {
    ...

     public void initTarget(
                @NonNull String targetHash,
                @NonNull Revision buildToolRevision,
                @NonNull Collection<LibraryRequest> usedLibraries,
                @NonNull AndroidBuilder androidBuilder,
                boolean useCachedVersion) {
            //检查targetHash和buildToolRevision不为空，否则抛异常。
            ...

            // 通过getSdkLoader()创建SdkLoader对象
            // 如果useCachedVersion && sSdkLoader，那么重用旧的SdkLoader。
            synchronized (LOCK_FOR_SDK_HANDLER) {
                if (useCachedVersion && sSdkLoader == null) {
                } else {
                    sSdkLoader = getSdkLoader();
                }
                sdkLoader = sSdkLoader;
            }

            // 通过SdkLoader -> AndroidSdkHandler获得SdkInfo.
            // SdkInfo包含annotations.jar和adb执行文件的位置。
            // annotations.jar位置：${sdkLocation}/tools/support/annotations.jar
            // adb文件位置：${sdkLocation}/platform-tools/adb
            SdkInfo sdkInfo = sdkLoader.getSdkInfo(logger);

            // 通过SdkLoader -> AndroidSdkHandler获得TargetInfo
            // TargetInfo包括IAndroidTarget(包装target信息),BuildToolInfo(包装build-tools信息)。
            TargetInfo targetInfo = sdkLoader.getTargetInfo(
                    targetHash,
                    buildToolRevision,
                    logger,
                    sdkLibData);

            androidBuilder.setSdkInfo(sdkInfo);
            androidBuilder.setTargetInfo(targetInfo);
            androidBuilder.setLibraryRequests(usedLibraries);

            // Check if platform-tools are installed. We check here because realistically, all projects
            // should have platform-tools in order to build.
            ProgressIndicator progress = new ConsoleProgressIndicator();
            AndroidSdkHandler sdk = AndroidSdkHandler.getInstance(getSdkFolder());
            LocalPackage platformToolsPackage =
                    sdk.getLatestLocalPackageForPrefix(SdkConstants.FD_PLATFORM_TOOLS, true, progress);
            if (platformToolsPackage == null) {
                //如果sdkLibData.useSdkDownload()就尝试下载；否则什么都不做。
                sdkLoader.installSdkTool(sdkLibData, SdkConstants.FD_PLATFORM_TOOLS);
            }
        }

    ...
}
```

// `configureCustomLintChecks` 和 `configureGlobalLintTask` 这两个方法都跟 Lint 任务创建相关，一个在创建 Variant 对象之前调用，一个在其后调用。

```
class TaskMananger {
    ...

    // "prepareLintJar" 任务，这个任务会将 {@File lint.jar} 拷贝到 {@File build/intermediates/lint/lint.jar}。
    public void configureCustomLintChecks(@NonNull TaskFactory tasks) {
        File lintJar = FileUtils.join(globalScope.getIntermediatesDir(), "lint", FN_LINT_JAR);

        AndroidTask<PrepareLintJar> copyLintTask =
        getAndroidTasks()
                .create(tasks, new PrepareLintJar.ConfigAction(globalScope, lintJar));
        globalScope.addTaskOutput(LINT_JAR, lintJar, copyLintTask.getName());
    }

    ...

    public void configureGlobalLintTask(@NonNull final Collection<VariantScope> variants) {

        final TaskFactory tasks = new TaskContainerAdaptor(project.getTasks());

        // 筛选 'non testing' && 'non feature' 的 BuildVariant。
        List<VariantScope> filteredVariants =
                variants.stream().filter(TaskManager::isLintVariant).collect(Collectors.toList());
        if (filteredVariants.isEmpty()) {
            return;
        }

        // 创建全局的 'lint' 任务。
        androidTasks.configure(
                tasks, new LintGlobalTask.GlobalConfigAction(globalScope, filteredVariants));

        // 将 'lint.jar' 文件加入每个 BuildVariant 的输出文件集中。
        FileCollection lintJarCollection = globalScope.getOutput(LINT_JAR);
        File lintJar = lintJarCollection.getSingleFile();
        for (VariantScope scope : variants) {
            scope.addTaskOutput(LINT_JAR, lintJar, PrepareLintJar.NAME);
        }
    }

    ...
}
```

在 `VariantManager::createAndroidTasks()` 中：
* `populateVariantDataList()` 创建 VariantData 对象。
* `TaskManager::createTopLevelTestTasks` 创建一些和单元测试有关的顶层 task。
* `createTasksForVariantData()` 创建编译相关的所有 task。
* `TaskManager::createReportTasks` 创建输出信息的工具 task。

```
class VariantManager {
    ...

    public void createAndroidTasks() {
        // 这是一个抽象方法，只有在 LibraryVariantFactory 中有实现。
        // LibraryVariantFactory 的实现中，确保 BuildType 和 ProductFlavor 没有配置
        // applicationId 或 applicationIdSuffix。
        variantFactory.validateModel(this);

        // 这又是一个抽象方法，在 BaseVariantFactory 的实现中检查，
        // 如果使用了 'android-apt' 插件，则提示开发者使用更新的 'annotationProcessor' 配置。
        variantFactory.preVariantWork(project);

        // 创建 VariantData 对象。
        final TaskFactory tasks = new TaskContainerAdaptor(project.getTasks());
        if (variantScopes.isEmpty()) {
            populateVariantDataList();
        }

        //@Test, 创建一些和单元测试有关的顶层 task。
        taskManager.createTopLevelTestTasks(tasks, !productFlavors.isEmpty());

        // 创建基于 BuildVariant 的编译 task。
        for (final VariantScope variantScope : variantScopes) {
            createTasksForVariantData(tasks, variantScope)
        }

        // 创建一些输出信息的 task。
        taskManager.createReportTasks(tasks, variantScopes);
    }

    ...

    public void createReportTasks(TaskFactory tasks, final List<VariantScope> variantScopes) {
    
        create "androidDependencies" as DependencyReportTask type.

        create "signingReport" as SigningReportTask type.
    }

    ...
}
```

#### 创建 VariantData 对象

`VariantManager::populateVariantDataList()` 依赖开发者编译脚本的配置，生成全部 VariantData 对象，进一步完成 project 模型的搭建。

```
class VariantManager {
    ...

    public void populateVariantDataList() {
        List<String> flavorDimensionList = extension.getFlavorDimensionList();

        if (productFlavors.isEmpty()) {
            // 注册 Gradle Transform，用于解压缩并转移所依赖的 library 提供的 aar 文件。
            configureDependencies();
            // 如果没有配置任何 ProductFlavor，使用空参数直接构造 VariantData 对象。
            createVariantDataForProductFlavors(Collections.emptyList());
        } else {
            // 如果配置了 ProductFlavor，检查配置合法性，并根据配置构造 VariantData 对象。
            // 确保每个 ProductFlavor 都有 Dimension
            if (flavorDimensionList == null || flavorDimensionList.isEmpty()) {
                // @Simplify 从 Android Gradle 3.0 之后，所有的 ProductFlavor 都需要有 Dimension
                // 参见：https://d.android.com/r/tools/flavorDimensions-missing-error-message.html
                [+]{...}
            } else if (flavorDimensionList.size() == 1) {
                // @Simplify 如果仅有一个 Dimension，将每个没有配置 Dimension 的 ProductFlavor 设为这个唯一的 Dimension。
                [+]{...}
            }

            // 注册 Gradle Transform，用于解压缩并转移所依赖的 library 提供的 aar 文件。
            configureDependencies();

            // 下面这两个调用生成了 flavorComboList，
            // 这个列表中的每个元素代表了一种 ProductFlavor 的组合方式
            // (按照 Dimension 从高到低的方式)
            Iterable<CoreProductFlavor> flavorDsl =
                    Iterables.transform(
                            productFlavors.values(),
                            ProductFlavorData::getProductFlavor);
            List<ProductFlavorCombo<CoreProductFlavor>> flavorComboList =
                    ProductFlavorCombo.createCombinations(
                            flavorDimensionList,
                            flavorDsl);
            // 以每一种 ProductFlavor 组合为输入，构造 VariantData 对象。
            for (ProductFlavorCombo<CoreProductFlavor>  flavorCombo : flavorComboList) {
                createVariantDataForProductFlavors(
                        (List<ProductFlavor>) (List) flavorCombo.getFlavorList());
            }
        }
    }

    ...

    private void createVariantDataForProductFlavors(
            @NonNull List<ProductFlavor> productFlavorList) {

        // getVariantConfigurationTypes() 是一个抽象方法，通常情况下只会返回长度为1的列表。
        // 例如 ApplicationVariantFactory 会返回 VariantType.DEFAULT;
        // LibraryVariantFactory 会返回 VariantType.LIBRARY;
        // 这里实际上就是将 VariantType 加入构造 VariantData 的参数。
        for (VariantType variantType : variantFactory.getVariantConfigurationTypes()) {
            createVariantDataForProductFlavorsAndVariantType(productFlavorList, variantType);
        }
    }

    private void createVariantDataForProductFlavorsAndVariantType(
            @NonNull List<ProductFlavor> productFlavorList, @NonNull VariantType variantType) {

        // @Test, 在工程支持单元测试的情况下，获取单元测试的目标 BuildTypeData
        // 默认情况下单元测试的目标 BuildType 是 'debug'
        BuildTypeData testBuildTypeData = null;
        [+]{...}

        BaseVariantData variantForAndroidTest = null;

        CoreProductFlavor defaultConfig = defaultConfigData.getProductFlavor();

        Action<com.android.build.api.variant.VariantFilter> variantFilterAction =
                extension.getVariantFilter();

        // @IDE
        [+]{...}

        for (BuildTypeData buildTypeData : buildTypes.values()) {
            boolean ignore = false;

            // 检查编译脚本中的 VariantFilter 配置，
            // 以确认当前的 BuildType & ProductFlavor 组合是否要被忽略。
            if (variantFilterAction != null) {
                variantFilter.reset(
                        defaultConfig,
                        buildTypeData.getBuildType(),
                        variantType,
                        productFlavorList);

                variantFilterAction.execute(variantFilter);
                ignore = variantFilter.isIgnore();
            }

            if (!ignore) {
                // 通过 BuildType & ProductFlavor 组合 创建 VariantData 对象。 
                BaseVariantData variantData =
                        createVariantDataForVariantType(
                                buildTypeData.getBuildType(),
                                productFlavorList,
                                variantType,
                                false);
                // 将 VariantScope 对象保存在 variantScopes 中。
                // VariantScope 对象和 VariantData 对象是一一对应、相互持有的关系。
                addVariant(variantData);

                // @Simplify
                [+]{...}
                
                if (variantFactory.hasTestScope()) {
                    if (buildTypeData == testBuildTypeData) {
                        variantForAndroidTest = variantData;
                    }

                    // @Test, 创建单元测试的 VariantData 对象。
                    [+]{...}
                }
            }
        }

        if (variantForAndroidTest != null) {
            //@Test, 创建单 AndroidTest 的 VariantData 对象。
            [+]{...}
        }
    }

    ...

    // 通过 BuildType & ProductFlavor 组合 创建 VariantData 对象。 
    private BaseVariantData createVariantDataForVariantType(
            @NonNull com.android.builder.model.BuildType buildType,
            @NonNull List<? extends ProductFlavor> productFlavorList,
            @NonNull VariantType variantType,
            boolean componentPluginUsed) {
        BuildTypeData buildTypeData = buildTypes.get(buildType.getName());
        final DefaultAndroidSourceSet sourceSet = defaultConfigData.getSourceSet();

        // 以 DefaultConfig 和 buildType 为基础创建 VariantConfiguration 对象，
        // 这个对象代表了 BuildType & ProductFlavor 组合的配置，并且会将脚本中的的配置进行合并，确定最终的配置。
        GradleVariantConfiguration variantConfig =
            GradleVariantConfiguration.getBuilderForExtension(extension)
                .create(
                    globalScope.getProjectOptions(),
                    defaultConfigData.getProductFlavor(),
                    sourceSet,
                    getParser(sourceSet.getManifestFile()),
                    buildTypeData.getBuildType(),
                    buildTypeData.getSourceSet(),
                    variantType,
                    signingOverride);

        // @Simplify
        [+]{...}

        // 依次将 ProductFlavor 对象加入 VariantConfiguration 对象，每次加入操作都会进行配置合并，保存在 'mergedFlavor' 中。
        // 合并时大部分属性会进行覆盖，高维 ProductFlavor > 低维 ProductFlavor > DefaultConfig。
        // 但是 applicationIdSuffix 和 versionNameSuffix 两个配置会将各个维度与 DefaultConfig 拼接到一起。
        // 此外 JavaCompileOptions、NdkOptions 等 Options 会进行覆盖合并，BuildType > 高维 ProductFlavor > 低维 ProductFlavor > DefaultConfig。
        for (ProductFlavor productFlavor : productFlavorList) {
            ProductFlavorData<CoreProductFlavor> data = productFlavors.get(
                    productFlavor.getName());

            String dimensionName = productFlavor.getDimension();
            if (dimensionName == null) {
                dimensionName = "";
            }

            variantConfig.addProductFlavor(
                    data.getProductFlavor(),
                    data.getSourceSet(),
                    dimensionName);
        }

        // 创建 BuildType & ProductFlavor 拼接而成的 SourceSet。
        // 名字分别为 '{BuildVaraintName}{ProductFlavorList}' 和 '{ProductFlavorList}'。
        NamedDomainObjectContainer<AndroidSourceSet> sourceSetsContainer = extension.getSourceSets();
        createCompoundSourceSets(productFlavorList, variantConfig, sourceSetsContainer, null);

        // 下面的5个 step 收集了 Variant 的所有相关的 SourceSet，分别是：
        // variant-specific, build type, multi-flavor, flavor1, flavor2, ..., defaultConfig.
        // 后面会使用这些 SourceSets 创建 VariantDependencies 对象。
        final List<DefaultAndroidSourceSet> variantSourceSets =
                Lists.newArrayListWithExpectedSize(productFlavorList.size() + 4);

        // 1. add the variant-specific if applicable.
        if (!productFlavorList.isEmpty()) {
            variantSourceSets.add((DefaultAndroidSourceSet) variantConfig.getVariantSourceProvider());
        }

        // 2. the build type.
        variantSourceSets.add(buildTypeData.getSourceSet());

        // 3. the multi-flavor combination
        if (productFlavorList.size() > 1) {
            variantSourceSets.add((DefaultAndroidSourceSet) variantConfig.getMultiFlavorSourceProvider());
        }

        // 4. the flavors.
        for (ProductFlavor productFlavor : productFlavorList) {
            variantSourceSets.add(productFlavors.get(productFlavor.getName()).getSourceSet());
        }

        // 5. The defaultConfig
        variantSourceSets.add(defaultConfigData.getSourceSet());

        // 创建 BaseVariantData 对象。
        BaseVariantData variantData =
                variantFactory.createVariantData(variantConfig, taskManager, recorder);

        // 创建 VariantDependencies 对象并由 BaseVariantData 对象持有。
        // 构造 VariantDependencies 对象的过程中，创建了几个新的 Configuration：
        // '{VariantName}CompileClasspath'，'{VariantName}RuntimeClasspath' 等。
        // 由于每个 SourceSet 都会对应一个 Configuration，
        // 所以以通过 variantSourceSets 参数可以获取所有与 Variant 的所有相关的 Configuration。
        // 这里新创建的 Configuration 会视情况依赖 与 Variant 的所有相关的 Configuration。
        VariantDependencies.Builder builder =
            VariantDependencies.builder(
                        project, androidBuilder.getErrorReporter(), variantConfig)
                .setConsumeType(
                        getConsumeType(variantData.getVariantConfiguration().getType()))
                .setPublishType(
                        getPublishingType(variantData.getVariantConfiguration().getType()))
                .setFlavorSelection(getFlavorSelection(variantConfig))
                .addSourceSets(variantSourceSets)
                .setBaseSplit(
                        variantType == VariantType.FEATURE && extension.getBaseFeature());
        final VariantDependencies variantDep = builder.build();
        variantData.setVariantDependency(variantDep);

        // 如果需要兼容 4.x 的 MultiDex，
        // 那么添加依赖 'com.android.support:multidex:1.0.2'。
        if (variantConfig.isLegacyMultiDexMode()) {
            project.getDependencies().add(
                    variantDep.getCompileClasspath().getName(), COM_ANDROID_SUPPORT_MULTIDEX);
            project.getDependencies().add(
                    variantDep.getRuntimeClasspath().getName(), COM_ANDROID_SUPPORT_MULTIDEX);
        }

        // @RenderScript
        [+]{...}

        return variantData;
    }

    ...

    private void createCompoundSourceSets(
            @NonNull List<? extends ProductFlavor> productFlavorList,
            @NonNull GradleVariantConfiguration variantConfig,
            @NonNull NamedDomainObjectContainer<AndroidSourceSet> sourceSetsContainer,
            @Nullable BaseVariantData testedVariantData) {
        if (!productFlavorList.isEmpty()) {
            // 创建名为 '{BuildVaraintName}{ProductFlavorList}' 的 SourceSet。
            DefaultAndroidSourceSet variantSourceSet =
                    (DefaultAndroidSourceSet) sourceSetsContainer.maybeCreate(
                            computeSourceSetName(
                                    variantConfig.getFullName(),
                                    variantConfig.getType()));
            variantConfig.setVariantSourceProvider(variantSourceSet);

            if (testedVariantData != null) {
                // @Test,配置 Configuration 的依赖关系。
                [+]{...}
            }
        }

        if (productFlavorList.size() > 1) {
            // 创建名为 '{ProductFlavorList}' 的 SourceSet。
            DefaultAndroidSourceSet multiFlavorSourceSet =
                    (DefaultAndroidSourceSet) sourceSetsContainer.maybeCreate(
                            computeSourceSetName(
                                    variantConfig.getFlavorName(),
                                    variantConfig.getType()));
            variantConfig.setMultiFlavorSourceProvider(multiFlavorSourceSet);

            if (testedVariantData != null) {
                // @Test,配置 Configuration 的依赖关系。
                [+]{...}
            }
        }
    }

    ...
}

```

#### `VariantMananger::configureDependencies()`

在这个方法，通过注册 Gradle Transform 接口，解压缩并转移了所依赖 AAR 中的文件，使这些文件能够参与后续的编译过程。

```
class VariantMananger {
    ...

    public void configureDependencies() {
        final DependencyHandler dependencies = project.getDependencies();

        // register transforms.
        final String explodedAarType = ArtifactType.EXPLODED_AAR.getType();
        dependencies.registerTransform(
                reg -> {
                    reg.getFrom().attribute(ARTIFACT_FORMAT, AndroidArtifacts.TYPE_AAR);
                    reg.getTo().attribute(ARTIFACT_FORMAT, explodedAarType);
                    reg.artifactTransform(ExtractAarTransform.class);
                });

        for (ArtifactType transformTarget : AarTransform.getTransformTargets()) {
            dependencies.registerTransform(
                    reg -> {
                        reg.getFrom().attribute(ARTIFACT_FORMAT, explodedAarType);
                        reg.getTo().attribute(ARTIFACT_FORMAT, transformTarget.getType());
                        reg.artifactTransform(
                                AarTransform.class, config -> config.params(transformTarget));
                    });
        }

        dependencies.registerTransform(
                reg -> {
                    reg.getFrom().attribute(ARTIFACT_FORMAT, explodedAarType);
                    reg.getTo()
                            .attribute(
                                    ARTIFACT_FORMAT,
                                    ArtifactType.SYMBOL_LIST_WITH_PACKAGE_NAME.getType());
                    reg.artifactTransform(LibrarySymbolTableTransform.class);
                });

        for (String transformTarget : JarTransform.getTransformTargets()) {
            dependencies.registerTransform(
                    reg -> {
                        reg.getFrom().attribute(ARTIFACT_FORMAT, "jar");
                        reg.getTo().attribute(ARTIFACT_FORMAT, transformTarget);
                        reg.artifactTransform(JarTransform.class);
                    });
        }

        AttributesSchema schema = dependencies.getAttributesSchema();

        // custom strategy for AndroidTypeAttr
        AttributeMatchingStrategy<AndroidTypeAttr> androidTypeAttrStrategy =
                schema.attribute(AndroidTypeAttr.ATTRIBUTE);
        androidTypeAttrStrategy.getCompatibilityRules().add(AndroidTypeAttrCompatRule.class);
        androidTypeAttrStrategy.getDisambiguationRules().add(AndroidTypeAttrDisambRule.class);

        // custom strategy for build-type and product-flavor.
        setBuildTypeStrategy(schema);

        setupFlavorStrategy(schema);
    }

    ...
}
```

#### `VariantMananger::createTasksForVariantData()`

*下面代码使用伪代码展现*

```
class VariantMananger {
    ...

    public void createTasksForVariantData(
            final TaskFactory tasks, final VariantScope variantScope) {
        
        create "assemble{BuildTypeName}" task as anchor task. 
            it "Assembles all assemble{BuildTypeName} builds."
            let "assemble" dependsOn "assemble{BuildTypeName}".

        // 创建更多基于 Variant 的 assemble task. 
        createAssembleTaskForVariantData(tasks, variantData);

        if (variantType.isForTesting()) {
            // @Test，创建基于 Variant 的单元测试 task.
            [+]{...}
        } else {
            // 创建基于 Variant 的编译 task.
            taskManager.createTasksForVariantScope(tasks, variantScope);
        }
    }

    ...

    // 创建基于 Variant 的 assemble task. 
    private void createAssembleTaskForVariantData(
            TaskFactory tasks, final BaseVariantData variantData) {
        final VariantScope variantScope = variantData.getScope();
        if (variantData.getType().isForTesting()) {

            create "assemble{VariantName}" task as anchor task. 
                set variantScope assemble task with it.

        } else {
            if (productFlavors.isEmpty()) {
                set variantScope assemble task with "assemble{BuildTypeName}".
            } else {
                
                create "assemble{VariantName}" task as anchor task. 
                    set variantScope assemble task with it.
                    let "assemble{BuildTypeName}" dependsOn "assemble{VariantName}".

                for each flavor
                    create "assemble{ProductFlavor}" as anchor task.
                        set ProductFlavorData assemble task with it.
                        let  "assemble{ProductFlavor}" dependsOn "assemble{VariantName}".

                if (variantConfig.getProductFlavors().size() > 1) {

                    create "assmble{ProductFlavorList}" as anchor task.
                        it "Assembles all builds for flavor combination: {ProductFlavorList}".
                        let "assmble{ProductFlavorList}" dependsOn "assemble{VariantName}".
                    
                    let "assemble" dependsOn "assmble{ProductFlavorList}".
                }
            }
        }
    }

    ...
}
```

#### `TaskMananger::createTasksForVariantScope()`

`TaskMananger::createTasksForVariantScope()` 是一个抽象方法，创建基于 Variant 的编译 task，这些编译 task 就是 android 编译 toolchain 的调用者。
这个方法在各个 TaskMananger 子类中实现不同，这里就选 ApplicationTaskManager 的实现来看。

*下面代码使用伪代码展现*

```
class ApplicationTaskManager extends TaskManager {
    ...

    public void createTasksForVariantScope(
            @NonNull final TaskFactory tasks, @NonNull final VariantScope variantScope) {
        BaseVariantData variantData = variantScope.getVariantData();
        assert variantData instanceof ApplicationVariantData;

        // in createAnchorTasks()
        create "pre{VariantName}Build" task as AppPreBuildTask task. 
            set VariantScope prebuild task with it.
            let "pre{VariantName}Build" dependsOn "preBuild".
        if enable code skrink
            let "pre{VariantName}Build" dependsOn "extractProguardFiles".

        create "generate{VariantName}Sources" as anchor task. 
            let "generate{VariantName}Sources" dependsOn "prepareLintJar"

        create "generate{VariantName}Resources" as anchor task. 

        create "generate{VariantName}Assets" as anchor task. 

        // @Test，创建单元测试覆盖率报告 task。
        [+]{...}

        create "compile{VariantName}Sources" as anchor task. 
            let "assemble{VariantName}" dependsOn "compile{VariantName}Sources".

        // in createCheckManifestTask()
        create "check{VariantName}Manifest" as CheckManifest type task. 
            let "check{VariantName}Manifest" dependsOn "pre{VariantName}Build".

        // @Wear
        handleMicroApp(tasks, variantScope);

        // 创建基于依赖的 transform stream
        createDependencyStreams(tasks, variantScope);

        // in createApplicationIdWriterTask()
        create "write{VariantName}ApplicationId" as ApplicationIdWriterTask type task. 
            add "build/intermediates/applicationId/{ProductFlavorName}/{BuildTypeName}" to VariantScope outputs.

        // in createMergeApkManifestsTask()
        create "create{VariantName}CompatibleScreenManifests" as CompatibleScreensManifest type task.

        create "process{VariantName}Manifest" as MergeManifests type task.
            let "process{VariantName}Manifest" dependsOn "check{VariantName}Manifest".

        // in createGenerateResValuesTask()
        create "generate{VariantName}ResValues" as GenerateResValues type task.
            let "generate{VariantName}Resources" dependsOn "generate{VariantName}ResValues".

        // in createRenderscriptTask()
        create "compile{VariantName}Renderscript" as RenderscriptCompile type task.
        if is testing
            let "compile{VariantName}Renderscript" dependsOn "process{VariantName}Manifest".
        else
            let "compile{VariantName}Renderscript" dependsOn "pre{VariantName}Build".

        let "generate{VariantName}Resources" dependsOn "compile{VariantName}Renderscript".

        if not RenderscriptNdkModeEnabled
            let "generate{VariantName}Sources" dependsOn "compile{VariantName}Renderscript".

        // in createMergeResourcesTask()
        create "merge{VariantName}Resources" as MergeResources type task.
            let "merge{VariantName}Resources" dependsOn "generate{VariantName}Resources".

        // in createMergeAssetsTask()
        create "merge{VariantName}Assets" as MergeSourceSetFolders type task.
            let "merge{VariantName}Assets" dependsOn "generate{VariantName}Assets".

        // in createBuildConfigTask()
        create "generate{VariantName}BuildConfig" as GenerateBuildConfig type task.
            let "generate{VariantName}Sources" dependsOn "generate{VariantName}BuildConfig".
            if is testing
                let "generate{VariantName}BuildConfig" dependsOn "process{VariantName}Manifest".
            else 
                let "generate{VariantName}BuildConfig" dependsOn "check{VariantName}Manifest".

        // in createApkProcessResTask()
        create "process{VariantName}Resources" as ProcessAndroidResources type task.
            let "generate{VariantName}Sources" dependsOn "process{VariantName}Resources".

        create "process{VariantName}JavaRes" as Sync type task.
            let "process{VariantName}JavaRes" dependsOn "pre{VariantName}Build".

        // in createAidlTask()
        create "compile{VariantName}Aidl" as AidlCompile type task.
            let "generate{VariantName}Sources" dependsOn "compile{VariantName}Aidl".
            let "compile{VariantName}Aidl" dependsOn "pre{VariantName}Build".

        // in createShaderTask()
        create "merge{VariantName}Shaders" as MergeSourceSetFolders type task.
        create "compile{VariantName}Shaders" as ShaderCompile type task.
            let "compile{VariantName}Shaders" dependsOn "merge{VariantName}Shaders".
            let "generate{VariantName}Assets" dependsOn "compile{VariantName}Shaders".

        // @NDK, ndk 编译任务
        [+]{...}

        // in createMergeJniLibFoldersTasks()
        create "merge{VariantName}JniLibFolders" as MergeSourceSetFolders type task.
            let "merge{VariantName}JniLibFolders" dependsOn "generate{VariantName}Assets".

        create "mergedJniFolder" stream
            type ExtendedContentType.NATIVE_LIBS
            scope Scope.PROJECT
            from {@File 'intermediates/jniLibs/{VariantConfigurationDir}'}
            let stream depend on "merge{VariantName}JniLibFolders" task

        // @NDK, 创建当前工程 ndk 编译产出的 so 文件以及 obj 文件 的 stream
        [+]{...}

        // @RenderScript, 创建当前工程 rs 编译产物的 stream
        [+]{...}

        create "transformNativeLibsWithMergeJniLibsFor{VariantName}" task
            with "mergeJniLibs" transform as MergeJavaResourcesTransform type.
            on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES. 


        // @DataBinding
        createDataBindingTasksIfNecessary(tasks, variantScope);

        // 创建 java 编译 task。
        addCompileTask();

        // @Ndk
        createStripNativeLibraryTask(tasks, variantScope);

        // @Multi
        createSplitTasks(tasks, variantScope);

        // @InstantRun
        [+]{...}

        // 创建打包 task。
        createPackagingTask(tasks, variantScope, buildInfoWriterTask);

        // in createLintTasks()
        create "lint{VariantName}" as LintPerVariantTask type.
    }

    ...
}
```

`TaskMananger::createDependencyStreams` 为 external 依赖（maven 或者 local jar 文件）和工程依赖创建 Android Transform stream 对象，使来自依赖的文件（主要是 class 文件）参与到 Android Transform 过程中。

```
class TaskMananger {
    ...

    protected void createDependencyStreams(
            @NonNull TaskFactory tasks,
            @NonNull final VariantScope variantScope) {
        
        // @Test, 基于测试覆盖率需求，修改 Configuration
        handleJacocoDependencies(variantScope);

        TransformManager transformManager = variantScope.getTransformManager();

        create "ext-libs-classes" stream
            type DefaultContentType.CLASSES
            scope Scope.EXTERNAL_LIBRARIES
            from ArtifactType.CLASSES files in ArtifactScope.EXTERNAL on RUNTIME_CLASSPATH  

        create "ext-libs-res-plus-native" stream
            type DefaultContentType.RESOURCES, ExtendedContentType.NATIVE_LIBS
            scope Scope.EXTERNAL_LIBRARIES
            from ArtifactType.JAVA_RES files in ArtifactScope.EXTERNAL on RUNTIME_CLASSPATH  

        // and the android AAR also have a specific jni folder
        create "ext-libs-native" stream
            type ExtendedContentType.NATIVE_LIBS
            scope Scope.EXTERNAL_LIBRARIES
            from ArtifactType.JNI files in ArtifactScope.EXTERNAL on RUNTIME_CLASSPATH

        // data binding related artifacts for external libs
        // @DataBinding, 创建 dataBinding 需要的 steams
        if (extension.getDataBinding().isEnabled()) {
            {...}
        }

        // for the sub modules, new intermediary classes artifact has its own stream
        create "sub-projects-classes" stream
            type DefaultContentType.CLASSES
            scope Scope.SUB_PROJECTS
            from ArtifactType.CLASSES files in ArtifactScope.MODULE on RUNTIME_CLASSPATH

        // same for the resources which can be java-res or jni
        create "sub-projects-res-plus-native" stream
            type DefaultContentType.RESOURCES, ExtendedContentType.NATIVE_LIBS
            scope Scope.SUB_PROJECTS
            from ArtifactType.JAVA_RES files in ArtifactScope.MODULE on RUNTIME_CLASSPATH

        // and the android library sub-modules also have a specific jni folder
        create "sub-projects-native" stream
            type ExtendedContentType.NATIVE_LIBS
            scope Scope.SUB_PROJECTS
            from ArtifactType.JNI files in ArtifactScope.MODULE on RUNTIME_CLASSPATH

        // provided only scopes.
        create "provided-classes" stream
            type DefaultContentType.CLASSES
            scope Scope.PROVIDED_ONLY
            from ArtifactType.JNI files in ArtifactScope.MODULE on (COMPILE_CLASSPATH minus RUNTIME_CLASSPATH)
        
        // @Test,  创建 test 需要的 steams
        if (variantScope.getTestedVariantData() != null) {
            {...}
        }
    }

    ...
}
```

compile 任务创建流程: 

```
class ApplicationTaskManager extends TaskManager {
    ...

    private void addCompileTask(@NonNull TaskFactory tasks, @NonNull VariantScope variantScope) {
        
        // @DataBinding
        createDataBindingMergeArtifactsTaskIfNecessary(tasks, variantScope);

        // in createJavacTask()
        create "javaPreCompile{VariantName}" as JavaPreCompileTask type.
            let "javaPreCompile{VariantName}" dependsOn "pre{VariantName}Build"

        create "compile{VariantName}JavaWithJavac" as AndroidJavaCompile type.
            let "compile{VariantName}JavaWithJavac" dependsOn "generate{VariantName}Sources"

        // Create the classes artifact for uses by external test modules.
        create "bundleAppClasses{VariantName}" as Jar type.


        // @Java8，一些基于 java8 配置的 validation 工作。
        [+]{...}

        // in addJavacClassesStream(variantScope)
        create "javac-output" stream
            type DefaultContentType.CLASSES, DefaultContentType.RESOURCES
            scope Scope.PROJECT
            from JAVAC output on variantScope

        create "pre-javac-generated-bytecode" stream
            type DefaultContentType.CLASSES, DefaultContentType.RESOURCES
            scope Scope.PROJECT
            from PreJavacGeneratedBytecode

        create "post-javac-generated-bytecode" stream
            type DefaultContentType.CLASSES, DefaultContentType.RESOURCES
            scope Scope.PROJECT
            from PostJavacGeneratedBytecode
        

        // in setJavaCompilerTask()
        let "compile{VariantName}Sources" dependsOn "compile{VariantName}JavaWithJavac".

        // 创建生成 dex 相关的 task。
        createPostCompilationTasks(tasks, variantScope);
    }

    ...
}
```

```
class TaskMananger {
    ...

    public void createPostCompilationTasks(
            @NonNull TaskFactory tasks,
            @NonNull final VariantScope variantScope) {

        checkNotNull(variantScope.getJavacTask());

        final BaseVariantData variantData = variantScope.getVariantData();
        final GradleVariantConfiguration config = variantData.getVariantConfiguration();

        TransformManager transformManager = variantScope.getTransformManager();
        // ---- Code Coverage first -----
        
        // @Test，创建测试覆盖率相关的 transform
        if (isTestCoverageEnabled) {
            createJacocoTransform(tasks, variantScope);
        }

        // in maybeCreateDesugarTask(tasks, variantScope, config.getMinSdkVersion(), transformManager)

        // 如果 java8 支持使用 desuger
        if (variantScope.getJava8LangSupportType() == Java8LangSupport.DESUGAR) {
            create "transformClassWithStackFramesFixerFor{VariantName}" task 
                with "stackFramesFixer" transform as FixStackFramesTransform type.
                on Scope.EXTERNAL_LIBRARIES

            create "transformClassWithDesugarFor{VariantName}" task as TransformTask
                with "desugar" transform as DesugarTransform type.
                on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES

            // 如果 minSdk.getFeatureLevel() 小于 19
            if (minSdk.getFeatureLevel()
                    < DesugarProcessBuilder.MIN_SUPPORTED_API_TRY_WITH_RESOURCES) {
                create "extractTryWithResourcesSupportJar{VariantName}" as ExtractTryWithResourcesSupportJar type.
                    let {@File 'intermediates/processing-tools/runtime-deps/{VariantConfigurationDir}/desugar_try_with_resources.jar'} depend on "extractTryWithResourcesSupportJar{VariantName}".

                create 'runtime-deps-try-with-resources' stream
                    type DefaultContentType.CLASSES
                    scope Scope.EXTERNAL_LIBRARIES
                    from {@File 'intermediates/processing-tools/runtime-deps/{VariantConfigurationDir}/desugar_try_with_resources.jar'}
            }
        }

        AndroidConfig extension = variantScope.getGlobalScope().getExtension();

        // in createMergeJavaResTransform(tasks, variantScope)
        create 'transformResourcesWithMergeJavaResFor{VariantName}' task
            with 'mergeJavaRes' transform as MergeJavaResourcesTransform type
            on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES 
        

        // 处理开发者自行注册的 transform
        // ----- External Transforms -----
        // apply all the external transforms.
        List<Transform> customTransforms = extension.getTransforms();
        List<List<Object>> customTransformsDependencies = extension.getTransformsDependencies();

        for (int i = 0, count = customTransforms.size(); i < count; i++) {
            Transform transform = customTransforms.get(i);

            List<Object> deps = customTransformsDependencies.get(i);
            transformManager
                    .addTransform(tasks, variantScope, transform)
                    .ifPresent(t -> {
                        if (!deps.isEmpty()) {
                            t.dependsOn(tasks, deps);
                        }

                        // if the task is a no-op then we make assemble task depend on it.
                        if (transform.getScopes().isEmpty()) {
                            variantScope.getAssembleTask().dependsOn(tasks, t);
                        }
                    });
        }

        // @IDE 
        // ----- Android studio profiling transforms
        [+]{...}

        // 创建包大小压缩的 transform
        // ----- Minify next -----
        // in maybeCreateJavaCodeShrinkerTransform(tasks, variantScope)
        create 'transformClassAndResourcesWithProguardFor{VariantName}' task
            with 'proguard' as ProGuardTransform type
            on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES

        create 'transformClassWithAndroidGradleClassShrinkerFor{VariantName}' task
            with 'androidGradleClassShrinker' as BuiltInShrinkerTransform type
            on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES

        create 'check{VariantName}ProguardFiles' as CheckProguardFiles type
            let 'transformClassAndResourcesWithProguardFor{VariantName}' or 'transformClassWithAndroidGradleClassShrinkerFor{VariantName}' depend on 'check{VariantName}ProguardFiles'
        
        // in  maybeCreateResourcesShrinkerTransform(tasks, variantScope)
        create 'transformClassWithShrinkResFor{VariantName}' task
            with shrinkRes transform as ShrinkResourcesTransform type
            on empty scope


        // @InstantRun
        // ----- 10x support
        [+] {...}
        
        // ----- Multi-Dex support

        DexingType dexingType = variantScope.getDexingType();

        // Upgrade from legacy multi-dex to native multi-dex if possible when using with a device
        if (dexingType == DexingType.LEGACY_MULTIDEX) {
            if (variantScope.getVariantConfiguration().isMultiDexEnabled()
                    && variantScope
                                    .getVariantConfiguration()
                                    .getMinSdkVersionWithTargetDeviceApi()
                                    .getFeatureLevel()
                            >= 21) {
                dexingType = DexingType.NATIVE_MULTIDEX;
            }
        }

        Optional<AndroidTask<TransformTask>> multiDexClassListTask;

        if (dexingType == DexingType.LEGACY_MULTIDEX) {
            boolean proguardInPipeline = variantScope.getCodeShrinker() == CodeShrinker.PROGUARD;

            // If ProGuard will be used, we'll end up with a "fat" jar anyway. If we're using the
            // new dexing pipeline, we'll use the new MainDexListTransform below, so there's no need
            // for merging all classes into a single jar.
            if (!proguardInPipeline && !usingIncrementalDexing(variantScope)) {
                // Create a transform to jar the inputs into a single jar. Merge the classes only,
                // no need to package the resources since they are not used during the computation.
                 create 'transformClassWithJarMergingFor{VariantName}' task
                    with jarMerging transform as JarMergingTransform type
                    on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES
            }

            // ---------
            // create the transform that's going to take the code and the proguard keep list
            // from above and compute the main class list.
            Transform multiDexTransform;
            if (usingIncrementalDexing(variantScope)) {
                create 'transformClassWithMultidexlistFor{VariantName}' task
                    with multidexlist transform as MainDexListTransform type
                    on empty scope
                    set multiDexTransform with it
            } else {
                create 'transformClassWithMultidexlistFor{VariantName}' task
                    with multidexlist transform as MultiDexTransform type
                    on empty scope
                    set multiDexTransform with it
            }
            multiDexClassListTask =
                    transformManager.addTransform(tasks, variantScope, multiDexTransform);
            multiDexClassListTask.ifPresent(variantScope::addColdSwapBuildTask);
        } else {
            multiDexClassListTask = Optional.empty();
        }

        if (usingIncrementalDexing(variantScope)) {
            // in createNewDexTasks(tasks, variantScope, multiDexClassListTask.orElse(null), dexingType)
            create 'transformClassWithDexBuilderFor{VariantName}' task
                    with dexBuilder transform as DexArchiveBuilderTransform type
                    on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES, InternalScope.MAIN_SPLIT

            if (dexingType != DexingType.LEGACY_MULTIDEX
                && variantScope.getCodeShrinker() == null
                && extension.getTransforms().isEmpty()) {
                
                create 'transformDexArchiveWithExternalLibsDexMergerFor{VariantName}' task
                    with externalLibsDexMerger transform as ExternalLibsMergerTransform type
                    on Scope.EXTERNAL_LIBRARIES
            }

            create 'transformDexArchiveWithDexMergerFor{VariantName}' task
                    with dexMerger transform as DexMergerTransform type
                    on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES, InternalScope.MAIN_SPLIT
            let 'transformDexWithDexMergerFor{VariantName}' depend on 'transformClassWithMultidexlistFor{VariantName}'

        } else {
            // in createDexTasks(tasks, variantScope, multiDexClassListTask.orElse(null), dexingType)

            if (preDexEnabled) {
                create 'transformClassWithPreDexFor{VariantName}' task
                    with preDex transform as PreDexTransform type
                    on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES, InternalScope.MAIN_SPLIT
            }

            if (!preDexEnabled || dexingType != DexingType.NATIVE_MULTIDEX) {
                // run if non native multidex or no pre-dexing
                create 'transformClassWithDexFor{VariantName}' task
                    with dex transform as DexTransform type
                    on Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES, InternalScope.MAIN_SPLIT

                let 'transformClassWithDexFor{VariantName}' depend on 'transformClassWithMultidexlistFor{VariantName}'
            }

        }

        // @InstantRun
        [+]{...}
    }

    ...
}
```

package 任务创建流程

```
class TaskManager {
    ...

    public void createPackagingTask(
            @NonNull TaskFactory tasks,
            @NonNull VariantScope variantScope,
            @Nullable AndroidTask<BuildInfoWriterTask> fullBuildInfoGeneratorTask) {
        ApkVariantData variantData = (ApkVariantData) variantScope.getVariantData();

        boolean signedApk = variantData.isSigned();

        create "package{VariantName}" as PackageApplication type.

        // @InstantRun
        [+]{...}

        let "package{VariantName}" dependsOn "merge{VariantName}Assets".
        let "package{VariantName}" dependsOn "process{VariantName}Resources".

        // @InstantRun
        [+]{...}

        create "validateSigning{VariantName}" as ValidateSigningTask type.
            let "package{VariantName}" dependsOn "validateSigning{VariantName}".

        let "package{VariantName}" dependsOn "compile{VariantName}JavaWithJavac".
        let "assemble{VariantName}" dependsOn package{VariantName}".

        // @InstantRun
        [+]{...}

        // @Multi
        [+]{...}

        if (signedApk) {
            create "install{VariantName}" as InstallVariantTask type.
                let "install{VariantName}" dependsOn "assemble{VariantName}".
        }

        // in maybeCreateLintVitalTask()
        create "lintVital{VariantName}" as LintPerVariantTask type.
            let "lintVital{VariantName}" dependsOn "compile{VariantName}JavaWithJavac".
        let "assemble{VariantName}" dependsOn "lintVital{VariantName}".

        // 如果任务树中包含 "lint"，就不再执行 "lintVital{VariantName}".
        project.getGradle().getTaskGraph().whenReady(
            taskGraph -> {
                if (taskGraph.hasTask(getTaskPath(LINT))) {
                    project.getTasks()
                            .getByName(lintReleaseCheck.getName())
                            .setEnabled(false);
                }});

        create "uninstall{VariantName}" as UninstallTask type.
            let "uninstallAll" dependsOn "uninstall{VariantName}".
    }

    ...
}
```

#### `ApiObjectFactory.create()`

基于 BaseVariantData 对象创建 BaseVariantImpl 对象, 并将 BaseVariantImpl 对象加入 extension 的容器中，到这里 project 模型的搭建完成。

```
public BaseVariantImpl create(BaseVariantData variantData) {
    if (variantData.getType().isForTesting()) {
        // Testing variants are handled together with their "owners".
        createVariantOutput(variantData, null);
        return null;
}

    BaseVariantImpl variantApi =
        variantFactory.createVariantApi(
            instantiator,
            objectFactory,
            androidBuilder,
            variantData,
            readOnlyObjectProvider);

    if (variantApi == null) {
        return null;
    }

    // @Test
    if (variantFactory.hasTestScope()) {
        // 创建 TestVariantImpl 对象和 UnitTestVariantImpl 对象
        [+]{...}
    }

    // in createVariantOutput(variantData, variantApi)
    // 创建 VariantOutputFactory 对象
    variantData.variantOutputFactory =
        new VariantOutputFactory(
            (variantData.getType() == LIBRARY)
                    ? LibraryVariantOutputImpl.class
                    : ApkVariantOutputImpl.class,
            instantiator,
            extension,
            variantApi,
            variantData);

    // 修改 apk 输出的的 versionCode 和 versionName
    variantData
            .getOutputScope()
            .getApkDatas()
            .forEach(
                apkData -> {
                    apkData.setVersionCode(
                        variantData.getVariantConfiguration().getVersionCode());
                    apkData.setVersionName(
                        variantData.getVariantConfiguration().getVersionName());
                    variantData.variantOutputFactory.create(apkData);
                });

    // 将 BaseVariantImpl 对象置入 extension 的 variantList 容器中
    // Only add the variant API object to the domain object set once it's been fully
    // initialized.
    extension.addVariant(variantApi);

    return variantApi;
}
```


### Android Transform 体系

## 附表

### android plugin 支持的 gradle properties

