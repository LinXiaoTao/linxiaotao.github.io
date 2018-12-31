---
title: AppPlugin中的Task-第一篇
date: 2018-11-12 10:26:03
categories: Gradle
tags:
---

### 前言

在使用 Android Studio 以后，基本新的 Android 项目都是用 Gradle 作为构建工具，关于 Gradle 的介绍不在本文范畴，当新建一个 Android 项目时，默认只会有个 module 使用`com.android.application` 插件，Gradle 的核心在于 task，即从 java 源文件和资源文件编译成 apk 文件（编译过程），就是由一系列的 task 组成，task 可以相互依赖，比如最基本的 `Task.dependsOn()`，所以理解 Android 编译过程就是理解各个 task 的作用。

> 本文涉及的源码：
>
> * com.android.tools.build:gradle:3.1.4
> * gradle-api-4.4

### AppPlugin

`com.android.application` 的源码位于 `com.android.build.gradle.AppPlugin`，AppPlugin 继承于 BasePlugin，BasePlugin 封装了大部分通用的逻辑：

``` java
@Override                                                                                
public void apply(@NonNull Project project) {                                            
    
    // 省略初始化步骤
    
    if (!projectOptions.get(BooleanOption.ENABLE_NEW_DSL_AND_API)) {
        // 不使用新的 DSL API
        TaskInputHelper.enableBypass();                                                  
        
        // threadRecorder 用于记录执行时间
        
        // configureProject 配置项目
        threadRecorder.record(                                                           
                ExecutionType.BASE_PLUGIN_PROJECT_CONFIGURE,                             
                project.getPath(),                                                       
                null,                                                                    
                this::configureProject);                                                 
        
        // configureExtension 配置 Extension 后，我们才能使用 android {} 进行配置
        threadRecorder.record(                                                           
                ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,               
                project.getPath(),                                                       
                null,                                                                    
                this::configureExtension);                                               
        
        // createTasks 本文的重点，创建必须的 task
        threadRecorder.record(                                                           
                ExecutionType.BASE_PLUGIN_PROJECT_TASKS_CREATION,                        
                project.getPath(),                                                       
                null,                                                                    
                this::createTasks);                                                      
    } else {                                                                             
         // 省略                   
    }                                                                                    
}                                                                                        
```

``` java
private void createTasks() {
    
    // createTasksBeforeEvaluate
    threadRecorder.record(                                                   
            ExecutionType.TASK_MANAGER_CREATE_TASKS,                         
            project.getPath(),                                               
            null,                                                            
            () -> taskManager.createTasksBeforeEvaluate());                  
    
    // afterEvaluate 执行
    project.afterEvaluate(                                                   
            project ->                                                       
                    threadRecorder.record(                                   
                            ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,  
                            project.getPath(),                               
                            null,                                            
                            () -> createAndroidTasks(false)));               
}                                                                            
```

从 `createTasks()` 可以看到，`createAndroidTasks()` 是在 `afterEvaluate()` 中执行的，evaluate 是 Gradle  一个执行的过程

> Gradle 编译的三个阶段：Initialization、Configuration、Execution。具体可以阅读 [build_lifecycle](https://docs.gradle.org/current/userguide/build_lifecycle.html)

`createAndroidTasks()` 这个方法较长，但是我们关心的是 `VariantManager.createAndroidTasks()`

``` java
public void createAndroidTasks() {                                                    
    variantFactory.validateModel(this);                                               
    variantFactory.preVariantWork(project);                                           
                                                                                      
    if (variantScopes.isEmpty()) {
        // 创建 variantScopes
        recorder.record(                                                              
                ExecutionType.VARIANT_MANAGER_CREATE_VARIANTS,                        
                project.getPath(),                                                    
                null /*variantName*/,                                                 
                this::populateVariantDataList);                                       
    }                                                                                 
                                                                                      
    // Create top level test tasks.                                                   
    recorder.record(                                                                  
            ExecutionType.VARIANT_MANAGER_CREATE_TESTS_TASKS,                         
            project.getPath(),                                                        
            null /*variantName*/,                                                     
            () -> taskManager.createTopLevelTestTasks(!productFlavors.isEmpty()));    
                                                                                      
                                                                                      
                                                                                      
    for (final VariantScope variantScope : variantScopes) {
        // 为每个 variant 生成各自的 task
        recorder.record(                                                              
                ExecutionType.VARIANT_MANAGER_CREATE_TASKS_FOR_VARIANT,               
                project.getPath(),                                                    
                variantScope.getFullVariantName(),                                    
                () -> createTasksForVariantData(variantScope));                       
    }                                                                                 
                                                                                      
    taskManager.createReportTasks(variantScopes);                                     
}                                                                                     
```

调用 `populateVariantDataList()` 来生成 VariantScopes，它是由 ProductFlavors、BuildType、VariantType 组合

``` java
public void createTasksForVariantData(final VariantScope variantScope) {                                              
    final BaseVariantData variantData = variantScope.getVariantData();                                                
    final VariantType variantType = variantData.getType();                                                            
                                                                                                                      
    final GradleVariantConfiguration variantConfig = variantScope.getVariantConfiguration();                          
                                                                                                                      
    final BuildTypeData buildTypeData = buildTypes.get(variantConfig.getBuildType().getName());                       
    if (buildTypeData.getAssembleTask() == null) {
        // 创建 BuildType 的 assemble task
        // 比如 assembleDebug 和 assembleRelease
        buildTypeData.setAssembleTask(taskManager.createAssembleTask(buildTypeData));                                 
    }                                                                                                                 
                                                                                                                      
    // Add dependency of assemble task on assemble build type task.
    // assemble 依赖于 assembleDebug 和 assembleRelease
    taskManager                                                                                                       
            .getTaskFactory()                                                                                         
            .configure(                                                                                               
                    "assemble",                                                                                       
                    task -> {                                                                                         
                        assert buildTypeData.getAssembleTask() != null;                                               
                        task.dependsOn(buildTypeData.getAssembleTask().getName());                                    
                    });                                                                                               
    
    // 根据 variant 创建 assemble task
    createAssembleTaskForVariantData(variantData);                                                                    
    if (variantType.isForTesting()) {                                                                                 
         // 省略代码                                                                       
    } else {
        // 根据 variant 创建 task
        taskManager.createTasksForVariantScope(variantScope);                                                         
    }                                                                                                                 
}                                                                                                                     
```

assemble task 用于将源文件编译为最终产物，比如 jar、aar、apk 等，在 `createTasksForVariantData()` 中会生成对应 BuildType 的 assemble task，比如 `assembleDebug` 和 `assembleRelease` ，接着设置 `assemble` 依赖于这两个 task，最后根据 variant 生成对应的 assemble task，比如 `assembleTestDebug`

``` java
private void createAssembleTaskForVariantData(final BaseVariantData variantData) {                      
    final VariantScope variantScope = variantData.getScope();                                           
    if (variantData.getType().isForTesting()) {                                                         
        variantScope.setAssembleTask(taskManager.createAssembleTask(variantData));                      
    } else {                                                                                            
        BuildTypeData buildTypeData =                                                                   
                buildTypes.get(variantData.getVariantConfiguration().getBuildType().getName());         
                                                                                                        
        Preconditions.checkNotNull(buildTypeData.getAssembleTask());                                    
                                                                                                        
        if (productFlavors.isEmpty()) {                                                                 
            // Reuse assemble task for build type if there is no product flavor.                        
            variantScope.setAssembleTask(buildTypeData.getAssembleTask());                              
                                                                                                        
            variantData.addTask(                                                                        
                    TaskContainer.TaskKind.ASSEMBLE, buildTypeData.getAssembleTask());                  
        } else {
            // 根据 variant 创建对应的 assemble task，比如 assembleTesDebug
            variantScope.setAssembleTask(taskManager.createAssembleTask(variantData));                  
                                                                                                        
            // setup the task dependencies                                                              
            // build type
            // build type assemble task 依赖于 具体的 assemble task
            buildTypeData.getAssembleTask().dependsOn(variantScope.getAssembleTask());                  
                                                                                                        
            // each flavor                                                                              
            GradleVariantConfiguration variantConfig = variantData.getVariantConfiguration();           
            for (CoreProductFlavor flavor : variantConfig.getProductFlavors()) {                        
                ProductFlavorData productFlavorData = productFlavors.get(flavor.getName());             
                                                                                                        
                DefaultTask flavorAssembleTask = productFlavorData.getAssembleTask();                   
                if (flavorAssembleTask == null) {
                    // 创建 flavor 的 assemble task
                    // 比如 assembleTes
                    flavorAssembleTask = taskManager.createAssembleTask(productFlavorData);             
                    productFlavorData.setAssembleTask(flavorAssembleTask);                              
                }
                // flavor assemble task 依赖于具体的 assemble task
                flavorAssembleTask.dependsOn(variantScope.getAssembleTask());                           
            }                                                                                           
                                                                                                        
            // assembleTask for this flavor(dimension), created on demand if needed.                    
            if (variantConfig.getProductFlavors().size() > 1) {                                         
                final String name = StringHelper.capitalize(variantConfig.getFlavorName());             
                final String variantAssembleTaskName =                                                  
                        StringHelper.appendCapitalized("assemble", name);                               
                if (!taskManager.getTaskFactory().containsKey(variantAssembleTaskName)) {               
                    Task task = taskManager.getTaskFactory().create(variantAssembleTaskName);           
                    task.setDescription("Assembles all builds for flavor combination: " + name);        
                    task.setGroup("Build");                                                             
                    task.dependsOn(variantScope.getAssembleTask().getName());                           
                }                                                                                       
                taskManager                                                                             
                        .getTaskFactory()                                                               
                        .configure(                                                                     
                                "assemble", task1 -> task1.dependsOn(variantAssembleTaskName));         
            }                                                                                           
        }                                                                                               
    }                                                                                                   
}                                                                                                       
```

上面会生成包括 BuildType、Flavor、BuildType + Flavor + VariantType 的 assemble task，同时 task 之间存在依赖关系：`assemble > buildType/flavor > 具体 ` 

![assemble-task-demo](https://leo-doc-img.oss-cn-hangzhou.aliyuncs.com/doc-img/assemble-task-demo.png?x-oss-process=style/doc-img)

除了上面所说的 assemble task，还将生成以下 task：

| task                                            | impl                                | dependsOn                                                    |
| ----------------------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| assemble{variant}                               | DefaultTask                         | compile{variant}Sources，package{variant}                    |
| pre{variant}Build                               | AppPreBuildTask                     | preBuild，extractProguardFiles(可能)                         |
| extractProguardFiles                            | ExtractProguardFiles                |                                                              |
| generate{variant}Sources                        | DefaultTask                         | compile{variant}Renderscript，generate{variant}BuildConfig，bundle{variant}Resources，compile{variant}Aidl |
| generate{variant}Resources                      | DefaultTask                         | generate{variant}ResValues，compile{variant}Renderscript     |
| generate{variant}Assets                         | DefaultTask                         | compile{variant}Shaders                                      |
| compile{variant}Sources                         | DefaultTask                         | compile{variant}Ndk，compile{variant}JavaWithJavac           |
| check{variant}Manifest                          | CheckManifest                       | pre{variant}Build                                            |
| write{variant}ApplicationId                     | ApplicationIdIdWriteTask            |                                                              |
| mainApkListPersistence{variant}                 | MainApkListPersistence              |                                                              |
| reportBuildArtifacts{variant}                   | BuildArtifactReportTask             |                                                              |
| create{variant}CompatibleScreenManifests        | CompatibleScreensManifest           |                                                              |
| process{variant}Manifest                        | MergeManifests                      | check{variant}Manifest                                       |
| generate{variant}ResValues                      | GenerateResValues                   |                                                              |
| compile{variant}Renderscript                    | RenderscriptCompile                 | pre{variant}Build                                            |
| merge{variant}Resources                         | MergeResources                      | generate{variant}Resources                                   |
| merge{variant}Shaders                           | MergeSourceSetFolders               |                                                              |
| compile{variant}Shaders                         | ShaderCompile                       | merge{variant}Shaders                                        |
| merge{variant}Assets                            | MergeSourceSetFolders               | generate{variant}Assets                                      |
| generate{variant}BuildConfig                    | GenerateBuildConfig                 | check{variant}Manifest                                       |
| splitsDiscoveryTask{variant}                    | SplitsDiscovery                     |                                                              |
| process{variant}JavaRes                         | Sync                                | pre{variant}Build                                            |
| process{variant}Resources                       | LinkApplicationAndroidResourcesTask |                                                              |
| bundle{variant}Resources                        | LinkAndroidResForBundleTask         |                                                              |
| compile{variant}Aidl                            | AidlCompile                         | pre{variant}Build                                            |
| compile{variant}Ndk                             | NdkCompile                          | pre{variant}Build                                            |
| merge{variant}JniLibFolders                     | MergeSourceSetFolders               | generate{variant}Assets                                      |
| javaPreCompile{variant}                         | JavaPreCompileTask                  | pre{variant}Build                                            |
| compile{variant}JavaWithJavac                   | AndroidJavaCompile                  | generate{variant}Sources                                     |
| bundleAppClasses{variant}                       | Jar                                 |                                                              |
| transformResourcesWithMergeJavaResFor{variant}  | TransformTask                       |                                                              |
| transformClassesWithMultidexlistFor{variant}    | TransformTask                       |                                                              |
| transformClassesWithDexBuilderFor{variant}      | TransformTask                       |                                                              |
| transformDexArchiveWithDexMergerFor{variant}    | TransformTask                       | transformClassesWithMultidexlistFor{variant}                 |
| preparePUBLISHED_JAVA_RES{variant}ForPublishing | PipelineToPublicationTask           |                                                              |
| package{variant}                                | PackageApplication                  | merge{variant}Assets，process{variant}Resources，validateSigning{variant}，compile{variant}JavaWithJavac |
| validateSigning{variant}                        | ValidateSigningTask                 |                                                              |
| install{variant}                                | InstallVariantTask                  | assemble{variant}                                            |
| uninstall{variant}                              | UninstallTask                       |                                                              |
| lint{variant}                                   | LintPerVariantTask                  |                                                              |

以 `assembleProdRelease` 命令，依次执行的 task：

> prod 是 product flavor

```
preBuild
extractProguardFiles
preProdReleaseBuild
compileProdReleaseRenderscript
checkProdReleaseManifest
generateProdReleaseBuildConfig
prepareLintJar
mainApkListPersistenceProdRelease
generateProdReleaseResValues
generateProdReleaseResources
compileProdReleaseAidl
createProdReleaseCompatibleScreenManifests
processProdReleaseManifest
mergeProdReleaseResources  // 合并资源
splitsDiscoveryTaskProdRelease
processProdReleaseResources
generateProdReleaseSources
javaPreCompileProdRelease
compileProdReleaseJavaWithJavac // javac
compileProdReleaseNdk
compileProdReleaseSources
mergeProdReleaseShaders
compileProdReleaseShaders
generateProdReleaseAssets
mergeProdReleaseAssets
transformClassesWithComponentCodeForProdRelease 
processProdReleaseJavaRes
transformResourcesWithMergeJavaResForProdRelease
transformClassesAndResourcesWithProguardForProdRelease // 混淆
transformClassesWithMultidexlistForProdRelease
transformClassesWithDexForProdRelease				// dex
transformClassesWithShrinkResForProdRelease
mergeProdReleaseJniLibFolders
transformNativeLibsWithMergeJniLibsForProdRelease
validateSigningProdRelease
packageProdRelease
```

接下来分析下比较重要的几个 task

> 接下来的分析，因为对应的源码比较复杂，所以只会简短介绍结果，没有详细过程，推荐 debug 整个流程

### CompileJavaWithJavac

首先找到执行的 AndroidJavaCompile

``` java
// AndroidJavaCompile.java
@Override                                                                                       
protected void compile(IncrementalTaskInputs inputs) {                                          
    // compileSdkVersion >= 24，需要使用 jdk1.8                                                     
    if (isPostN()) {                                                                            
        if (!JavaVersion.current().isJava8Compatible()) {                                       
            throw new RuntimeException("compileSdkVersion '" + compileSdkVersion + "' requires "
                    + "JDK 1.8 or later to compile.");                                          
        }                                                                                       
    }                                                                                           
    
    // 处理注解处理器，在之前 task 会将使用 annotationProcessor 标示的注解处理器写入到 annotationProcessors.json 中
    processAnalytics();                                                                         
                                                                                                
    // Create directory for output of annotation processor.                                     
    FileUtils.mkdirs(annotationProcessorOutputFolder);                                          
                                                                                                
    mInstantRunBuildContext.startRecording(InstantRunBuildContext.TaskType.JAVAC);
    // 调用 JavaCompile.compile
    super.compile(inputs);                                                                      
    mInstantRunBuildContext.stopRecording(InstantRunBuildContext.TaskType.JAVAC);               
}                                                                                               
```

``` java
// JavaCompile.java
@TaskAction                                                                                                                                                                                                                                                                                                           
protected void compile(IncrementalTaskInputs inputs) {
    // 是否为增量编译
    if (!this.compileOptions.isIncremental()) {
        this.compile();                                                                                                                                                                                                                                                                                               
    } else {                                                                                                                                                                                                                                                                                                          
        DefaultJavaCompileSpec spec = this.createSpec();                                                                                                                                                                                                                                                              
        CompileCaches compileCaches = this.createCompileCaches();                                                                                                                                                                                                                                                     
        IncrementalCompilerFactory factory = new IncrementalCompilerFactory(this.getFileOperations(), this.getStreamHasher(), this.getCachingFileHasher(), this.getPath(), this.createCompiler(spec), this.source, compileCaches, (IncrementalTaskInputsInternal)inputs, this.getEffectiveAnnotationProcessorPath()); 
        Compiler<JavaCompileSpec> compiler = factory.createCompiler();                                                                                                                                                                                                                                                
        this.performCompilation(spec, compiler);                                                                                                                                                                                                                                                                      
    }                                                                                                                                                                                                                                                                                                                 
}                                                                                                                                                                                                                                                                                                                     
```

不管是增量编译还是全量编译，最后都会调用 `performCompilation`

``` java
private void performCompilation(JavaCompileSpec spec, Compiler<JavaCompileSpec> compiler) {
	// 执行编译
    WorkResult result = compiler.execute(spec);                                             
    this.setDidWork(result.getDidWork());                                                   
}                                                                                           
```

Compiler 采用**装饰者设计模式**，最外层是 `CleaningJavaCompilerSupport` 会先删除编译目录，再调用 `JdkJavaCompiler` 执行 `javac` 过程，具体为，创建一个 `JavacTaskImpl` 然后 JavacTask 会调用 `com.sun.tools.javac.main.Main` 进行处理，其中创建 JavacTask 的任务是由 `JavacTool` 完成

