---
title: android-plugin-code-gradle-1
catalog: true
subtitle:
header-img:
tags:
---

## :gradle 和 :gradle-core

:gradle是Android Plugin和Gradle Project链接的类，也就是一些Plugin和Extension类，这里就是Android Plugin的入口；:gradle-core是Android Plugin的实现工程，被:gralde工程调用，提供了各种Mananger类型，处理extension中的信息、生成Variant、创建AndroidTask，并经过各种AndroidTask链接到其他Android构建工具。

## BasePlugin.class

BasePlugin是所有Android Plugin的基类，BasePlugin.apply()方法实现了Android Plugin对Project进行Config的基本过程，通过对BasePlugin.apply()的阅读可以基本了解Android Plugin在Config阶段完成的工作。

### BasePlugin.apply()

```java
 protected void apply(@NonNull Project project) {

        //一些检查和Set工作
        //
        //1. 检查builder工程和当前工程的version一致，否则抛异常，大概是防止分支&依赖错误。
        //version读取自.properties文件。
        //
        //2. 从gradle.properties中(大概)读取'android.threadPoolSize'，并调用ExecutorSingleton.setThreadPoolSize(int)设置线程池大小
        //
        //3. 在window上执行路径上不能有非ASCII字符,否则抛异常,可以通过'android.overridePathCheck'屏蔽这个检查
        //
        //4.检查正在build的工程(可能是某个Android应用工程)，没有同名的gradle subproject，否则抛异常。
        ...

        //AndroidBasePlugin会被所有AndroidPlugin加载，它不做任何事情，只是方便其他插件编写者判断Android Plugin是否被加载过了。
        project.getPluginManager().apply(AndroidBasePlugin.class);
        ...

        //ProjectOptions是Android Plugin所关注的properties配置的抽象类，通过project.getProperties()读取后续需要使用的properties配置。所有需要关注的proerties被定义在:gradle-core > com.android.build.gradle.options包下面的枚举类中。
        this.projectOptions = new ProjectOptions(project);
        ...

        //config project对象，创建一些Manager为BasePlugin的私有成员，并注册一些监听
        configureProject()
        //创建android plugin使用的Extension，并继续创建一些Manager为BasePlugin的私有成员
        configureExtension()
        //创建android plugin使用的Task
        createTasks()
    }
```

### BasePlugin.apply() > configureProject()
config project对象，创建一些Manager为BasePlugin的私有成员，并注册一些监听。

```java
private void configureProject() {
    //一些检查
    //1. 检查当前运行的Gradle版本是否大于Android Plugin所需的最小版本
    ...

    //TODO: ExtraModelInfo的作用
    extraModelInfo = new ExtraModelInfo(projectOptions, project.getLogger());
    //处理Android SDK信息的类，在构造方法中会寻找SDK位置, 写在后面
    sdkHandler = new SdkHandler(project, getLogger());

    //TODO: AndroidBuilder类提供了android实际build的方法
    // 例如mergeManifest，处理resource等
    androidBuilder = new AndroidBuilder(
                project == project.getRootProject() ? project.getName() : project.getPath(),
                creator,
                new GradleProcessExecutor(project),
                new GradleJavaProcessExecutor(project),
                extraModelInfo,
                getLogger(),
                isVerbose());

    //TODO: DataBindingBuilder的作用
    dataBindingBuilder = new DataBindingBuilder();
    dataBindingBuilder.setPrintMachineReadableOutput(
            extraModelInfo.getErrorFormatMode() ==
                    ExtraModelInfo.ErrorFormatMode.MACHINE_PARSABLE);

    // 应用了JavaBasePlugin插件和JacocoPlugin插件
    // Apply the Java and Jacoco plugins.
    project.getPlugins().apply(JavaBasePlugin.class);
    project.getPlugins().apply(JacocoPlugin.class);

    // (TODO:)单单给assemble任务改了描述...之后是对assemble做了什么
    // assemble任务(大概)是来自JavaBasePlugin。
    project.getTasks()
            .getByName("assemble")
            .setDescription(
                    "Assembles all variants of all applications and secondary packages.");

    // call back on execution. This is called after the whole build is done (not
    // after the current project is done).
    // This is will be called for each (android) projects though, so this should support
    // being called 2+ times.
    project.getGradle()
        .addBuildListener(
            // 监听gradle build，仅仅实现了buildFinished()监听gradle执行完成，写在后面
            ...
            );

    project.getGradle()
            .getTaskGraph()
            .addTaskExecutionGraphListener(
            // TaskExecutionGraphListener监听TaskGraph创建，写在后面      
            );
    }
```

#### BasePlugin.apply() > configureProject() > SdkHandler() > findLocation()
寻找Android SDK

```java
private void findLocation(@NonNull Project project) {
    ...

    File rootDir = project.getRootDir();
    File localProperties = new File(rootDir, FN_LOCAL_PROPERTIES);
    Properties properties = new Properties();
    //从root_project/local.properties读出Properties到properties对象中
    ...
    //寻找sdk目录
    //local.properties的'sdk.dir'属性 > local.properties的'android.dir'属性
    // > ANDROID_HOME环境变量 > android.home环境变量
    //其中'android.dir'返回的isRegularSdk为false
    Pair<File, Boolean> sdkLocation = findSdkLocation(properties, rootDir);
    sdkFolder = sdkLocation.getFirst();
    isRegularSdk = sdkLocation.getSecond();
    //寻找ndk目录
    //local.properties的'ndk.dir'属性 > ANDROID_NDK_HOME环境变量
    // > 'sdk目录/ndk-bundle'
    ndkFolder = NdkHandler.findNdkDirectory(properties, rootDir);

    //从properties中读出cmake.dir，并赋值给this.cmakePathInLocalProp(File对象)。
    // Check if the user has specified a cmake directory in local properties and assign the
        // cmake folder.
        String cmakeProperty = properties.getProperty("cmake.dir");
        if (cmakeProperty != null) {
            // cmake.dir can be specified one of two ways
            // 1. cmake.dir="value"
            // 2. cmake.dir=value
            // Inorder to create a file, we just need the value without the quotes, so if the CMake
            // directory's value is within quotes, extract that value.
            Matcher m = PATTERN_FIND_QUOTES.matcher(cmakeProperty);
            if (m.find()) {
                cmakePathInLocalProp = new File(m.group(1));
            } else {
                cmakePathInLocalProp = new File(cmakeProperty);
            }
        }
}
```

#### BasePlugin.apply() > configureProject() > project.getGradle().addBuildListener() > BuildListener匿名类

```java
private final LibraryCache libraryCache = LibraryCache.getCache();

 @Override
public void buildFinished(BuildResult buildResult) {
    ExecutorSingleton.shutdown();
    sdkHandler.unload(); //TODO: (大概)释放加载的android SDK
    //TODO: 清理PreDexCache
    PreDexCache.getCache().clear(
        FileUtils.join(project.getRootProject().getBuildDir(),
            FD_INTERMEDIATES,
            "dex-cache",
            "cache.xml"),
        getLogger());
    Main.clearInternTables();//TODO: Main.clearInternTables()的作用
}
```

#### BasePlugin.apply() > configureProject() > roject.getGradle().getTaskGraph().addTaskExecutionGraphListener() >  graphPopulated()

```java
 taskGraph -> {
    for (Task task : taskGraph.getAllTasks()) {
        if (task instanceof TransformTask) {
            Transform transform = ((TransformTask) task).getTransform();

            // TODO: 如果有DexTransform任务，加载PreDexCache
            if (transform instanceof DexTransform) {
                PreDexCache.getCache().load(
                    FileUtils.join(project.getRootProject().getBuildDir(),
                        FD_INTERMEDIATES,
                        "dex-cache",
                        "cache.xml"));
                break;
            } 
        }
    }
}}
```

### BasePlugin.apply() > configureExtension()
plugin使用的Extension，并继续创建一些Manager为BasePlugin的私有成员。

```java
private void configureExtension() {

    //创建BuildType、ProductFlavor和SigningConfig三个NamedDomainObjectContainer
    final NamedDomainObjectContainer<BuildType> buildTypeContainer =
        project.container(BuildType.class,
            new BuildTypeFactory(instantiator, project, project.getLogger()));
    final NamedDomainObjectContainer<ProductFlavor> productFlavorContainer =
            project.container(ProductFlavor.class,
                new ProductFlavorFactory(
                    instantiator, project, project.getLogger(), extraModelInfo));
    final NamedDomainObjectContainer<SigningConfig> signingConfigContainer =
        project.container(SigningConfig.class, new SigningConfigFactory(instantiator));

    //创建BuildOutputs的NamedDomainObjectContainer，并将其加入extension。
    final NamedDomainObjectContainer<BaseVariantOutput> buildOutputs =
        project.container(BaseVariantOutput.class);
    project.getExtensions().add("buildOutputs", buildOutputs);

    //创建Extension，abstract方法
    extension = createExtension(...);

    //创建NdkHandler，处理ndk信息的类，写在后面。
    ndkHandler = new NdkHandler(
        project.getRootDir(),
        null, /* compileSkdVersion, this will be set in afterEvaluate */
        "gcc",
        "" /*toolchainVersion*/);

    //创建buildCache，TODO: 写在后面。
    @Nullable
    FileCache buildCache = BuildCacheUtils.createBuildCacheIfEnabled(project, projectOptions);

    //创建globalScope，GlobalScope是一种Context类，用于获取上下文相关的对象。
    GlobalScope globalScope =new GlobalScope(
        project,
        projectOptions,
        androidBuilder,
        extension,
        sdkHandler,
        ndkHandler,
        registry,
        buildCache);

    //创建VariantFactory，abstract方法。
    variantFactory = createVariantFactory(globalScope...);
    //创建TaskManager，abstract方法
    taskManager = createTaskManager(globalScope...);
    //创建VariantManager，管理Variant的类
    variantManager = new VariantManager(globalScope...);
    
    //TODO: ModelBuilder和NativeModelBuilder的作用；于AndroidBuilder的关系
    // Register a builder for the custom tooling model
    ModelBuilder modelBuilder = new ModelBuilder(
        ...
        new NativeLibraryFactoryImpl(ndkHandler),
        getProjectType(),
        AndroidProject.GENERATION_ORIGINAL);
    registry.register(modelBuilder);
    // Register a builder for the native tooling model
    NativeModelBuilder nativeModelBuilder = 
        new NativeModelBuilder(variantManager);
    registry.register(nativeModelBuilder);

    // 监听BuildType、ProductFlavor和SigningConfig的创建，并进行处理
    // map the whenObjectAdded callbacks on the containers.

    //variantManager::addSigningConfig会将SigningConfig放置到VariantManager的Map中
    signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);

    //用名debug的SigningConfig初始化buildType对象
    //buildType.init()通过名字判断，只对名为debug的buildType有效
    //variantManager.addBuildType，写在后面
    buildTypeContainer.whenObjectAdded(
        buildType -> {
            SigningConfig signingConfig =
                signingConfigContainer.findByName(BuilderConstants.DEBUG);
            buildType.init(signingConfig);
            variantManager.addBuildType(buildType);
        }
    );
    //variantManager.addProductFlavor写在后面
    productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);

    // map whenObjectRemoved on the containers to throw an exception.
    signingConfigContainer.whenObjectRemoved(
        new UnsupportedAction("Removing signingConfigs is not supported."));
    buildTypeContainer.whenObjectRemoved(
        new UnsupportedAction("Removing build types is not supported."));
    productFlavorContainer.whenObjectRemoved(
        new UnsupportedAction("Removing product flavors is not supported."));

    // create default Objects, signingConfig first as its used by the BuildTypes.
    //variantFactory.createDefaultComponents是个abstract方法
    //大部分实现会创建名为debug的signingConfig和名为debug、release的buildType
    variantFactory.createDefaultComponents(
        buildTypeContainer, productFlavorContainer, signingConfigContainer);
}
```

#### BasePlugin::apply() > configureExtension() > NdkHandler()

```java
public NdkHandler(
            @NonNull File projectDir,
            @Nullable String platformVersion,
            @NonNull String toolchainName,
            @NonNull String toolchainVersion,
            @Nullable Boolean useUnifiedHeaders) {
        //toolchainName is "gcc"
        this.toolchain = Toolchain.getByName(toolchainName);
        this.toolchainVersion = toolchainVersion;
        this.platformVersion = platformVersion;

        //寻找ndk目录
        //local.properties的'ndk.dir'属性 > ANDROID_NDK_HOME环境变量
        // > 'sdk目录/ndk-bundle'
        ndkDirectory = findNdkDirectory(projectDir);

        if (ndkDirectory == null || !ndkDirectory.exists()) {
            ndkInfo = null;
            revision = null;
        } else {
            //在'ndk目录/source.properties'文件中读取Pkg.Revision属性，来确定初始化ndk的revision。
            revision = findRevision(ndkDirectory);
            //根据revision初始化ndkInfo。
            if (revision == null) {
                ndkInfo = new DefaultNdkInfo(ndkDirectory);
            } else if (revision.getMajor() > LATEST_SUPPORTED_VERSION) {
                ndkInfo = new NdkR14Info(ndkDirectory);
            } else {
                switch (revision.getMajor()) {
                    case 14:
                        ndkInfo = new NdkR14Info(ndkDirectory);
                        break;
                    case 13:
                        ndkInfo = new NdkR13Info(ndkDirectory);
                        break;
                    case 12:
                        ndkInfo = new NdkR12Info(ndkDirectory);
                        break;
                    case 11:
                        ndkInfo = new NdkR11Info(ndkDirectory);
                        break;
                    default:
                        ndkInfo = new DefaultNdkInfo(ndkDirectory);
                }
            }
        }

        //TODO:useUnifiedHeaders的作用。
        // useUnifiedHeaders defaults to true for r15 and above.
        this.useUnifiedHeaders =
                useUnifiedHeaders != null
                        ? useUnifiedHeaders
                        : revision != null && revision.getMajor() > 14;

        if (this.useUnifiedHeaders && (revision == null || revision.getMajor() < 14)) {
            throw new InvalidUserDataException("Unified headers is not supported before NDK r14.");
        }
    }
```


#### BasePlugin::apply() > configureExtension() > NamedDomainObjectContainer::whenObjectAdded() > VariantManager::addBuildType()

```java
/**
 * Adds new BuildType, creating a BuildTypeData, and the associated source set,
 * and adding it to the map.
 * @param buildType the build type.
 */
public void addBuildType(@NonNull CoreBuildType buildType) {
    String name = buildType.getName();
    //检查BuildType的名字不是以androidTest、test开头，并且不等于lint;否则抛异常
    checkName(name, "BuildType");

    if (productFlavors.containsKey(name)) {
        throw new RuntimeException("BuildType names cannot collide with ProductFlavor names");
    }
    //创建和buildType同名的sourceSet
    DefaultAndroidSourceSet mainSourceSet = 
        (DefaultAndroidSourceSet) extension.getSourceSets().maybeCreate(name);
    //1. 创建名为test + buildType.name的sourceSet
    //如果sourceSet名字以UnitTest结尾，那么去掉这个结尾
    //2. 如果buildType是testBuildType，创建名为AndroidTest + buildType.name的sourceSet
    //如果sourceSet名字以AndroidTest，那么去掉这个结尾
    // TODO: testBuildType的含义。
    DefaultAndroidSourceSet unitTestSourceSet = null;
    DefaultAndroidSourceSet androidTestSourceSet = null;
    if (variantFactory.hasTestScope()) {
        if (buildType.getName().equals(extension.getTestBuildType())) {
                androidTestSourceSet =
                    (DefaultAndroidSourceSet)
                        extension.getSourceSets().maybeCreate(
                            computeSourceSetName(buildType.getName(),
                                ANDROID_TEST));
            }
        unitTestSourceSet = (DefaultAndroidSourceSet) extension
                .getSourceSets().maybeCreate(
                        computeSourceSetName(buildType.getName(), UNIT_TEST));
    }
    //创建BuildTypeData
    BuildTypeData buildTypeData = new BuildTypeData(
            buildType, project, mainSourceSet, androidTestSourceSet, unitTestSourceSet);
    //放进map中
    buildTypes.put(name, buildTypeData);
}
```

#### BasePlugin::apply() > configureExtension() > NamedDomainObjectContainer::whenObjectAdded() > VariantManager::addProductFlavor()

```java
/**
 * Adds a new ProductFlavor, creating a ProductFlavorData and associated source sets,
 * and adding it to the map.
 *
 * @param productFlavor the product flavor
 */
public void addProductFlavor(@NonNull CoreProductFlavor productFlavor) {
    String name = productFlavor.getName();
    //检查ProductFlavor的名字不是以androidTest、test开头，并且不等于lint;否则抛异常
    checkName(name, "ProductFlavor");

    if (buildTypes.containsKey(name)) {
        throw new RuntimeException("ProductFlavor names cannot collide with BuildType names");
    }
    //创建和productFlavor同名的sourceSet
    DefaultAndroidSourceSet mainSourceSet = 
        (DefaultAndroidSourceSet) extension.getSourceSets().maybeCreate(        productFlavor.getName());
    //创建名为test + productFlavor.name 和 androidTest + productFlavor.name的SS
    //如果sourceSet名字以UnitTest 和 AndroidTest结尾，那么去掉这个结尾
    DefaultAndroidSourceSet androidTestSourceSet = null;
    DefaultAndroidSourceSet unitTestSourceSet = null;
    if (variantFactory.hasTestScope()) {
        androidTestSourceSet = (DefaultAndroidSourceSet) extension
                .getSourceSets().maybeCreate(
                        computeSourceSetName(productFlavor.getName(), ANDROID_TEST));
        unitTestSourceSet = (DefaultAndroidSourceSet) extension
                .getSourceSets().maybeCreate(
                        computeSourceSetName(productFlavor.getName(), UNIT_TEST));
    }

    //创建ProductFlavorData
    ProductFlavorData<CoreProductFlavor> productFlavorData =
            new ProductFlavorData<>(
                    productFlavor,
                    mainSourceSet,
                    androidTestSourceSet,
                    unitTestSourceSet,
                    project);
    //放进map中
    productFlavors.put(productFlavor.getName(), productFlavorData);
}
```

### BasePlugin::apply() > createTasks()

```java
private void createTasks() {
    //TODO: 其中需要关注的task的作用
    //
    //在这个方法中创建和Variant无关的Task，和一些工具Task
    //这些task通过AndroidTaskRegistry的接口创建
    //AndroidTaskRegistry会创建Task，执行传入的Config方法，并将Task记录在AndroidTaskRegistry对象的Map中
    //
    //创建的Task有:
    //UNINSTALL_ALL、DEVICE_CHECK、CONNECTED_CHECK、MAIN_PREBUILD、ASSEMBLE_ANDROID_TEST (Task.class)；
    //EXTRACT_PROGUARD_FILES (ExtractProguardFiles.class)
    //LINT (Lint.class)
    //'sourceSets' (SourceSetsTask.class)
    //'compileLint' (LintCompile.class)
    //'cleanBuildCache' (CleanBuildCache.class)
    //
    //建立依赖关系:
    //EXTRACT_PROGUARD_FILES -> MAIN_PREBUILD
    //JavaBasePlugin.CHECK_TASK_NAME -> LINT
    threadRecorder.record(
        ExecutionType.TASK_MANAGER_CREATE_TASKS,
        project.getPath(),
        null,
        () ->
            taskManager.createTasksBeforeEvaluate(
                new TaskContainerAdaptor(project.getTasks())));


    //afterEvaluate后创建Android Tasks,写在后面
    project.afterEvaluate(
        project ->
            threadRecorder.record(
                ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                project.getPath(),
                null,
                () -> createAndroidTasks(false)));
    }
```

### BasePlugin::apply() > createTasks() > Project::afterEvaluate() > BasePlugin::createAndroidTasks()

```java
final void createAndroidTasks(boolean force) {
    //在这里会执行一些afterEvaluate的检查
    //1. 确保extension中compileSdkVersion和buildToolsVersion被配置
    //2. 确保没有apply JavaPlugin.class
    ...
    
    ndkHandler.setCompileSdkVersion(extension.getCompileSdkVersion());
    //调用SdkHandler::initTarget，将Sdk信息set到AndroidBuilder中，写在后面
    ensureTargetSetup();

    ...

    if (hasCreatedTasks) {
        return;
    }
    hasCreatedTasks = true;

    extension.disableWrite();

    ...
    // 创建SDK附带的本地repository, 并将这些maven库置于repositories list最前面
    // setup SDK repositories.
    sdkHandler.addLocalRepositories(project);
    //TODO: databinding
    taskManager.addDataBindingDependenciesIfNecessary(extension.getDataBinding());
    // 生成所有VariantData以及和Variant相关的task，写在后面
    variantManager.createAndroidTasks();
    // 生成Variant对象并将其加入extension，写在后面
    ApiObjectFactory apiObjectFactory =
            new ApiObjectFactory(
                    androidBuilder, extension, variantFactory, instantiator);
    for (BaseVariantData variantData : variantManager.getVariantDataList()) {
        apiObjectFactory.create(variantData);
    }

    //TODO: 看看生成JSON的过程是否需要了解。
    // Create and read external native build JSON files depending on what's happening right now.
    ...
}
```

#### BasePlugin::apply() > createTasks() > Project::afterEvaluate() > BasePlugin::createAndroidTasks() > BasePlugin::ensureTargetSetup() > SdkHandler::initTarget()

```java
 public void initTarget(
            @NonNull String targetHash,
            @NonNull Revision buildToolRevision,
            @NonNull Collection<LibraryRequest> usedLibraries,
            @NonNull AndroidBuilder androidBuilder,
            boolean useCachedVersion) {
        //检查targetHash和buildToolRevision不为空，否则抛异常。
        ...

        // 通过getSdkLoader()创建SdkLoader对象，写在后面。
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
```

### BasePlugin::apply() > createTasks() > Project::afterEvaluate() > BasePlugin::createAndroidTasks() > VariantManager::createAndroidTasks()

```java
/**
 * Variant/Task creation entry point.
 *
 * Not used by gradle-experimental.
 */
public void createAndroidTasks() {
    //检查在gradle脚本中Variant相关的设置是否合法
    // TODO: 以下两个为抽象方法，分类讨论(App,Library)
    variantFactory.validateModel(this);
    variantFactory.preVariantWork(project);

    final TaskFactory tasks = new TaskContainerAdaptor(project.getTasks());
    if (variantScopes.isEmpty()) {
        // 生成所有的VariantData, 写在后面
        populateVariantDataList();
    }
    // Create top level test tasks.
    // 写在后面
    taskManager.createTopLevelTestTasks(tasks, !productFlavors.isEmpty());

    for (final VariantScope variantScope : variantScopes) {
        // 根据每个VariantData生成task，写在后面
        createTasksForVariantData(tasks, variantData)
    }
    // 生成report类型的task，写在后面
    taskManager.createReportTasks(tasks, variantDataList);
}
```

#### BasePlugin::apply() > createTasks() > Project::afterEvaluate() > BasePlugin::createAndroidTasks() > VariantManager::createAndroidTasks() > VariantManager::populateVariantDataList() > VariantManager::createVariantDataForProductFlavors()

```java
/**
 * Create all variants.
 */
public void populateVariantDataList() {
    List<String> flavorDimensionList = extension.getFlavorDimensionList();

    if (productFlavors.isEmpty()) {
        //TODO: 注册transforms
        configureDependencies();
        createVariantDataForProductFlavors(Collections.emptyList());
    } else {
        // ensure that there is always a dimension
        if (flavorDimensionList == null || flavorDimensionList.isEmpty()) {
            ...
            //如果定义了productFlavors却没有定义flavorDimension，则抛出异常
        }

        // can only call this after we ensure all flavors have a dimension.
        configureDependencies();

        // Create iterable to get GradleProductFlavor from ProductFlavorData.
        Iterable<CoreProductFlavor> flavorDsl =
                Iterables.transform(
                        productFlavors.values(),
                        ProductFlavorData::getProductFlavor);

        // Get a list of all combinations of product flavors.
        List<ProductFlavorCombo<CoreProductFlavor>> flavorComboList =
                ProductFlavorCombo.createCombinations(
                        flavorDimensionList,
                        flavorDsl);
        //flavorDimensionList + flavorDsl生成flavorComboList
        //每个ProductFlavorCombo<CoreProductFlavor>对象代表一个包含所有dimension的ProductFlavor的组合。
        for (ProductFlavorCombo<CoreProductFlavor>  flavorCombo : flavorComboList) {
            //noinspection unchecked
            //对于创建VariantData对象列表
            createVariantDataForProductFlavors(
                    (List<ProductFlavor>) (List) flavorCombo.getFlavorList());
        }
    }
}
```

```java
private void createVariantDataForProductFlavors(
            @NonNull List<ProductFlavor> productFlavorList) {
        //TODO: variantFactory是抽象类,分实现讨论
        for (VariantType variantType : variantFactory.getVariantConfigurationTypes()) {
            createVariantDataForProductFlavorsAndVariantType(
                productFlavorList, variantType);
        }
    }
```

```java
/**
 * Creates VariantData for a specified list of product flavor.
 *
 * This will create VariantData for all build types of the given flavors.
 *
 * @param productFlavorList the flavor(s) to build.
 */
private void createVariantDataForProductFlavorsAndVariantType(
        @NonNull List<ProductFlavor> productFlavorList,
        @NonNull VariantType variantType) {

    BuildTypeData testBuildTypeData = null;
    if (extension instanceof TestedAndroidConfig) {
        //检查名为extension.testBuildType的buildType存在，否则抛异常
        //extension.testBuildType的默认值为'debug'
        //App和Library类型的Extension都implements了TestedAndroidConfig
        ...
        testBuildTypeData = buildTypes.get(testedExtension.getTestBuildType());
    }

    BaseVariantData variantForAndroidTest = null;
    //defaultConfigData是VariantManager创建时，由extension.defaultConfig创建的ProductFlavorData对象。
    //extension.defaultConfig一个特殊的ProductFlavor,其名字为'main'；对应的名字为'main'的sourceSet。
    CoreProductFlavor defaultConfig = defaultConfigData.getProductFlavor();
    //extension.variantFilter为Action，这个Action被delegated到VariantFilter对象
    //VariantFilter会被应用到每个Variant，决定是否被忽略。
    Action<com.android.build.api.variant.VariantFilter> variantFilterAction =
            extension.getVariantFilter();

    //TODO: 关于"android.injected.restrict.variant.name"配置
    final String restrictedProject = AndroidGradleOptions.getRestrictVariantProject(project);
    final boolean restrictVariants = restrictedProject != null;

    // compare the project name if the type is not a lib.
    final boolean projectMatch;
    final String restrictedVariantName;
    if (restrictVariants) {
        projectMatch = variantFactory.getVariantConfigurationType() != VariantType.LIBRARY &&
                project.getPath().equals(restrictedProject);
        restrictedVariantName = AndroidGradleOptions.getRestrictVariantName(project);
    } else {
        projectMatch = false;
        restrictedVariantName = null;
    }


    for (BuildTypeData buildTypeData : buildTypes.values()) {
        boolean ignore = false;

        //通过restrictVariant和variantFilter和确定是否ignore；restrictVariant优先级高于variantFilter
        if (restrictVariants || variantFilterAction != null) {
            ...
        }
        //不忽略的情况创建VariantData
        if (!ignore) {
            //创建VariantData，TODO:过程比较复杂，单开一篇
            BaseVariantData<?> variantData = createVariantData(
                    buildTypeData.getBuildType(),
                    productFlavorList,
                    variantType);
            variantScopes.add(variantData.getScope());

            GradleVariantConfiguration variantConfig = variantData.getVariantConfiguration();
            ...

            if (variantFactory.hasTestScope()) {

                if (buildTypeData == testBuildTypeData) {
                    //如果buildType是testBuildType，androidTest所需的variantData
                    variantForAndroidTest = variantData;
                }

                if (variantType != FEATURE) {
                    // There's nothing special about unit testing the feature variant, so
                    // there's no point creating the duplicate unit testing variant. This only
                    // causes tests to run twice when running "testDebug".
                    //创建unitTest的VariantData
                    TestVariantData unitTestVariantData =
                            createTestVariantData(variantData, UNIT_TEST);
                    variantScopes.add(variantData.getScope());
                }

            }
        }
    }

    if (variantForAndroidTest != null) {
        //创建AndroidTest的VariantData
        TestVariantData androidTestVariantData = createTestVariantData(
                variantForAndroidTest,
                ANDROID_TEST);
        variantDataList.add(androidTestVariantData);
    }
}
```

#### BasePlugin::apply() > createTasks() > Project::afterEvaluate() > BasePlugin::createAndroidTasks() > VariantManager::createAndroidTasks() > TaskManager::createTopLevelTestTasks() 
//TODO: test tasks
```java
public void createTopLevelTestTasks(final TaskFactory tasks, boolean hasFlavors) {
    //创建名为mockableAndroidJar的Task，类型为MockableAndroidJarTask。
    //mockableAndroidJar的Task的输入为AndroidTarget中的android.jar文件，
    //输出为build/generated/mockable-android-${apilevel}.jar
    //任务过程就是将android.jar中所有非java标准库的.class文件复制到输出文件中。
    //在复制过程中会通过asm对这些.class文件做一些修改：去掉所有class和method的final修饰；确保所有方法返回默认值而不是抛出"Stub!"异常。
    //TODO: mockable*.jar的用处
    createMockableJarTask(tasks);
    //TODO:这个是什么任务
    createAttrFromAndroidJarTask(tasks);

    final List<String> reportTasks = Lists.newArrayListWithExpectedSize(2);

    List<DeviceProvider> providers = getExtension().getDeviceProviders();

    // If more than one flavor, create a report aggregator task and make this the parent
    // task for all new connected tasks.  Otherwise, create a top level connectedAndroidTest
    // DefaultTask.

    //task名字为connectedAndroidTest
    AndroidTask<? extends DefaultTask> connectedAndroidTestTask;
    if (hasFlavors) {
        connectedAndroidTestTask = androidTasks.create(tasks,
                new AndroidReportTask.ConfigAction(
                        globalScope,
                        AndroidReportTask.ConfigAction.TaskKind.CONNECTED));
        reportTasks.add(connectedAndroidTestTask.getName());
    } else {
        connectedAndroidTestTask = androidTasks.create(tasks,
                CONNECTED_ANDROID_TEST,
                connectedTask -> {
                    connectedTask.setGroup(JavaBasePlugin.VERIFICATION_GROUP);
                    connectedTask.setDescription("Installs and runs instrumentation tests "
                            + "for all flavors on connected devices.");
        });
    }

    tasks.named(CONNECTED_CHECK, check -> check.dependsOn(connectedAndroidTestTask.getName()));

    AndroidTask<? extends DefaultTask> deviceAndroidTestTask;
    // if more than one provider tasks, either because of several flavors, or because of
    // more than one providers, then create an aggregate report tasks for all of them.
    if (providers.size() > 1 || hasFlavors) {
        deviceAndroidTestTask = androidTasks.create(tasks,
                new AndroidReportTask.ConfigAction(
                        globalScope,
                        AndroidReportTask.ConfigAction.TaskKind.DEVICE_PROVIDER));
        reportTasks.add(deviceAndroidTestTask.getName());
    } else {
        deviceAndroidTestTask = androidTasks.create(tasks,
                DEVICE_ANDROID_TEST,
                providerTask -> {
                    providerTask.setGroup(JavaBasePlugin.VERIFICATION_GROUP);
                    providerTask.setDescription("Installs and runs instrumentation tests "
                            + "using all Device Providers.");
                });
    }

    tasks.named(DEVICE_CHECK, check -> check.dependsOn(deviceAndroidTestTask.getName()));

    // Create top level unit test tasks.

    androidTasks.create(tasks, JavaPlugin.TEST_TASK_NAME, unitTestTask -> {
        unitTestTask.setGroup(JavaBasePlugin.VERIFICATION_GROUP);
        unitTestTask.setDescription("Run unit tests for all variants.");
    });
    tasks.named(JavaBasePlugin.CHECK_TASK_NAME,
            check -> check.dependsOn(JavaPlugin.TEST_TASK_NAME));

    // If gradle is launched with --continue, we want to run all tests and generate an
    // aggregate report (to help with the fact that we may have several build variants, or
    // or several device providers).
    // To do that, the report tasks must run even if one of their dependent tasks (flavor
    // or specific provider tasks) fails, when --continue is used, and the report task is
    // meant to run (== is in the task graph).
    // To do this, we make the children tasks ignore their errors (ie they won't fail and
    // stop the build).
    //TODO: move to mustRunAfter once is stable.
    if (!reportTasks.isEmpty() && project.getGradle().getStartParameter()
            .isContinueOnFailure()) {
        project.getGradle().getTaskGraph().whenReady(new Closure<Void>(this, this) {
            public void doCall(TaskExecutionGraph taskGraph) {
                for (String reportTask : reportTasks) {
                    if (taskGraph.hasTask(reportTask)) {
                        tasks.named(reportTask, new Action<Task>() {
                            @Override
                            public void execute(Task task) {
                                ((AndroidReportTask) task).setWillRun();
                            }
                        });
                    }
                }
            }
        });
    }
}
```

#### BasePlugin::apply() > createTasks() > Project::afterEvaluate() > BasePlugin::createAndroidTasks() > VariantManager::createAndroidTasks() > VariantManager::createTasksForVariantData()

```java
/**
 * Create tasks for the specified variantData.
 */
public void createTasksForVariantData(
        final TaskFactory tasks,
        final VariantScope variantScope) {

    final VariantType variantType = variantData.getType();
    final GradleVariantConfiguration variantConfig = variantScope.getVariantConfiguration();

    final BuildTypeData buildTypeData = buildTypes.get(
            variantData.getVariantConfiguration().getBuildType().getName());
    if (buildTypeData.getAssembleTask() == null) {
        //创建名为assemble'$BuildType'的Task
        buildTypeData.setAssembleTask(taskManager.createAssembleTask(tasks, buildTypeData));
    }

    // Add dependency of assemble task on assemble build type task.
    tasks.named("assemble", new Action<Task>() {
        @Override
        public void execute(Task task) {
            assert buildTypeData.getAssembleTask() != null;
            task.dependsOn(buildTypeData.getAssembleTask().getName());
        }
    });

    //创建基于Variant的assemble Tasks,写在后面
    createAssembleTaskForVariantData(tasks, variantData);

    if (variantType.isForTesting()) { //如果是Test类的Variant
        final BaseVariantData testedVariantData =
                    (BaseVariantData) ((TestVariantData) variantData).getTestedVariantData();

        // Add the container of dependencies.
        // The order of the libraries is important, in descending order:
        // variant-specific, build type (, multi-flavor, flavor1, flavor2, ..., defaultConfig.
        // variant-specific if the full combo of flavors+build type. Does not exist if no flavors.
        // multi-flavor is the combination of all flavor dimensions. Does not exist if <2 dimension.
        List<CoreProductFlavor> testProductFlavors = 
            variantConfig.getProductFlavors();
        List<DefaultAndroidSourceSet> testVariantSourceSets =
            Lists.newArrayListWithExpectedSize(4 + testProductFlavors.size());

        // 1. add the variant-specific if applicable.
        if (!testProductFlavors.isEmpty()) {
            testVariantSourceSets.add(
                    (DefaultAndroidSourceSet) variantConfig.getVariantSourceProvider());
        }

        // 2. the build type.
        DefaultAndroidSourceSet buildTypeConfigurationProvider =
                buildTypeData.getTestSourceSet(variantType);
        if (buildTypeConfigurationProvider != null) {
            testVariantSourceSets.add(buildTypeConfigurationProvider);
        }

        // 3. the multi-flavor combination
        if (testProductFlavors.size() > 1) {
            testVariantSourceSets.add(
                    (DefaultAndroidSourceSet) variantConfig.getMultiFlavorSourceProvider());
        }

        // 4. the flavors.
        for (CoreProductFlavor productFlavor : testProductFlavors) {
            testVariantSourceSets.add(
                    this.productFlavors
                            .get(productFlavor.getName())
                            .getTestSourceSet(variantType));
        }

        // now add the default config
        testVariantSourceSets.add(
            defaultConfigData.getTestSourceSet(variantType));

        // If the variant being tested is a library variant, VariantDependencies must be
        // computed after the tasks for the tested variant is created.  Therefore, the
        // VariantDependencies is computed here instead of when the VariantData was created.
        VariantDependencies.Builder builder =
            VariantDependencies.builder(
                    project, androidBuilder.getErrorReporter(), variantConfig)
                .setConsumeType(
                    getConsumeType(
                        testedVariantData.getVariantConfiguration().getType()))
                .addSourceSets(testVariantSourceSets)
                .setFlavorSelection(getFlavorSelection(variantConfig))
                .setBaseSplit(
                        variantType == VariantType.FEATURE
                                && extension.getBaseFeature());

        final VariantDependencies variantDep = builder.build();
        variantData.setVariantDependency(variantDep);

        if (variantType == VariantType.ANDROID_TEST &&
                testVariantConfig.isLegacyMultiDexMode()) {
            project.getDependencies().add(
                variantDep.getCompileConfiguration().getName(), COM_ANDROID_SUPPORT_MULTIDEX_INSTRUMENTATION);
            project.getDependencies().add(
                variantDep.getPackageConfiguration().getName(), COM_ANDROID_SUPPORT_MULTIDEX_INSTRUMENTATION);
        }

        if (testedVariantData.getVariantConfiguration().getRenderscriptSupportModeEnabled()) {
            project.getDependencies()
                .add(
                    variantDep.getCompileClasspath().getName(),
                    project.files(androidBuilder.getRenderScriptSupportJar()));
        }

        switch (variantType) {
            case ANDROID_TEST:
                taskManager.createAndroidTestVariantTasks(tasks, (TestVariantData) variantData);
                break;
            case UNIT_TEST:
                taskManager.createUnitTestVariantTasks(tasks, (TestVariantData) variantData);
                break;
            default:
                throw new IllegalArgumentException("Unknown test type " + variantType);
        }
    } else { //如果不是Test类的Variant
    //根据Variant创建构建过程的Tasks，写在后面
        taskManager.createTasksForVariantScope(tasks, variantScope);
    }
}
```

#### BasePlugin::apply() > createTasks() > Project::afterEvaluate() > BasePlugin::createAndroidTasks() > VariantManager::createAndroidTasks() > TaskManager.createReportTasks()

```java
public void createReportTasks(TaskFactory tasks, final List<VariantScope> variantScopes) {
    //创建androidDependencies任务
    androidTasks.create(
        tasks,
        "androidDependencies",
        DependencyReportTask.class,
        task -> {
            task.setDescription("Displays the Android dependencies of the project.");
            task.setVariants(variantScopes);
            task.setGroup(ANDROID_GROUP);
        });

    //创建signingReport任务
    androidTasks.create(
        tasks,
        "signingReport",
        SigningReportTask.class,
        task -> {
            task.setDescription("Displays the signing info for each variant.");
            task.setVariants(variantScopes);
            task.setGroup(ANDROID_GROUP);
        });
}

```

#### BasePlugin::apply() > createTasks() > Project::afterEvaluate() > BasePlugin::createAndroidTasks() > ApiObjectFactory.create()

```java
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

    if (variantFactory.hasTestScope()) {
        TestVariantData androidTestVariantData =
                ((TestedVariantData) variantData).getTestVariantData(ANDROID_TEST);

        if (androidTestVariantData != null) {
            TestVariantImpl androidTestVariant =
                instantiator.newInstance(
                    TestVariantImpl.class,
                    androidTestVariantData,
                    variantApi,
                    objectFactory,
                    androidBuilder,
                    readOnlyObjectProvider,
                    variantData
                        .getScope()
                        .getGlobalScope()
                        .getProject()
                        .container(VariantOutput.class));
            createVariantOutput(androidTestVariantData, androidTestVariant);

            ((TestedAndroidConfig) extension).getTestVariants().add(androidTestVariant);
            ((TestedVariant) variantApi).setTestVariant(androidTestVariant);
        }

        TestVariantData unitTestVariantData =
                ((TestedVariantData) variantData).getTestVariantData(UNIT_TEST);
        if (unitTestVariantData != null) {
            UnitTestVariantImpl unitTestVariant =
                instantiator.newInstance(
                    UnitTestVariantImpl.class,
                    unitTestVariantData,
                    variantApi,
                    objectFactory,
                    androidBuilder,
                    readOnlyObjectProvider,
                    variantData
                        .getScope()
                        .getGlobalScope()
                        .getProject()
                        .container(VariantOutput.class));

            ((TestedAndroidConfig) extension).getUnitTestVariants().add(unitTestVariant);
            ((TestedVariant) variantApi).setUnitTestVariant(unitTestVariant);
        }
    }

    variantData.variantOutputFactory =
        new VariantOutputFactory(
            (variantData.getType() == LIBRARY)
                    ? LibraryVariantOutputImpl.class
                    : ApkVariantOutputImpl.class,
            instantiator,
            extension,
            variantApi,
            variantData);
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

    // Only add the variant API object to the domain object set once it's been fully
    // initialized.
    extension.addVariant(variantApi);

    return variantApi;
}
```

