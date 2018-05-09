---
title: Gradle多项目实践
date: 2018-05-08 18:58:17
categories: Gradle
tags:
---

## 前言

上篇文章中，我们说到了 [Gradle多项目构建](https://linxiaotao.github.io/2018/04/29/Gradle%E5%A4%9A%E9%A1%B9%E7%9B%AE%E6%9E%84%E5%BB%BA/) 的一些知识点，但这些总归只是纸上谈兵，今天我们在实际项目中通过之前学到的知识去改造下项目的 Gradle 构建脚本，充分利用 Gradle 带来的好处。

虽然使用 Gradle 作为构建工具已经有一段时间了，但很多同学对它还是很陌生的，基本都是沿用 Android Studio 默认生成的配置，用的比较多的可能也只是 `dependencies` 配置，从不同的 `repositories` 去下载依赖，可能在需要实现一些自定义配置的时候，就去 google 后照搬代码，对其配置参数也是似懂非懂，出了问题不知道如何下手解决，所以这篇文章的主要目的是，实践 Gradle 在 Android 上的运用，对 Gradle 的使用有基本认识。

[项目源码](https://github.com/LinXiaoTao/GradleCaseProject)

## 改造开始

### Gradle 知识

我们先看下 Android Studio 默认帮我们生成的配置代码：

``` groovy
apply plugin: 'com.android.application'

android {
    // android 构建配置
}

dependencies {
 	// 依赖配置
}

```

先简单讲下上面各部分的配置含义，首先我们知道如果不先 `apply plugin: 'com.android.application'` 就不会有 `android` 配置选项，那么这个配置选项具体是什么内容呢？我们可以通过源码了解下，`com.android.application` 这个插件就是我们在根目录下的 *build.gradle* 中导入的：

``` groovy
buildscript {
    repositories {
        // 构建脚本依赖源地址
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.1.2'
    }
}
```

对 Gradle Plugin 有点了解的同学都知道，Gradle Plugin 需要定义到 *META-INF* 中，浏览 *gradle-3.1.2.jar*，*META-INF* 目录如下：

```
├── MANIFEST.MF
├── gradle-plugins
│   ├── android-library.properties
│   ├── android-reporting.properties
│   ├── android.properties
│   ├── com.android.application.properties
│   ├── com.android.atom.properties
│   ├── com.android.bundle.properties
│   ├── com.android.debug.structure.properties
│   ├── com.android.feature.properties
│   ├── com.android.instantapp.properties
│   ├── com.android.library.properties
│   ├── com.android.lint.properties
│   └── com.android.test.properties
```

这里对应的是提供的各个插件的配置，比如 *com.android.application.properties* 就表示 `com.android.application` 插件，其他也是一样，properties 的名称表示插件的名称。*com.android.application.properties* 的内容如下：

```
implementation-class=com.android.build.gradle.AppPlugin
```

`com.android.build.gradle.AppPlugin` 这个类就是 Application Plugin 的源码。阅读 AppPlugin 源码，我们可以看到这一段：

``` groovy
	@NonNull
    @Override
    protected BaseExtension createExtension(
            @NonNull Project project,
            @NonNull ProjectOptions projectOptions,
            @NonNull AndroidBuilder androidBuilder,
            @NonNull SdkHandler sdkHandler,
            @NonNull NamedDomainObjectContainer<BuildType> buildTypeContainer,
            @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavorContainer,
            @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigContainer,
            @NonNull NamedDomainObjectContainer<BaseVariantOutput> buildOutputs,
            @NonNull SourceSetManager sourceSetManager,
            @NonNull ExtraModelInfo extraModelInfo) {
        return project.getExtensions()
                .create(
                        "android",
                        AppExtension.class,
                        project,
                        projectOptions,
                        androidBuilder,
                        sdkHandler,
                        buildTypeContainer,
                        productFlavorContainer,
                        signingConfigContainer,
                        buildOutputs,
                        sourceSetManager,
                        extraModelInfo);
    }
```

首先我们先通过 API 文档了解下 [project.extensions](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html) 的定义：

```
定义：
	ExtensionContainer extensions (read-only)
说明：
	Allows adding DSL extensions to the project. Useful for plugin authors.
	允许向项目添加 DSL 扩展，对插件作者很有用
```

接着我们看下 [`ExtensionContainer.create`](https://docs.gradle.org/current/javadoc/org/gradle/api/plugins/ExtensionContainer.html#create-java.lang.String-java.lang.Class-java.lang.Object...-) 方法的定义：

```
定义：
	<T> T create(String name,Class<T> type,Object... constructionArguments)
说明：
	Creates and adds a new extension to this container. A new instance of the given type will be created using the given constructionArguments. The extension will be exposed as type unless the extension itself declares a preferred public type via the HasPublicType protocol. The new instance will have been dynamically made ExtensionAware, which means that you can cast it to ExtensionAware.
	向项目创建并且添加一个新的扩展。通过给定的 type 创建一个新的实例，使用给定的 constructionArguments 构造函数参数。可以通过 HasPublicType 接口声明对外开放的类型，否则整个扩展将对外开放。新的实例将动态生成为 
ExtensionAware，这意味着你可以将它转换成 ExtensionAware
```

现在我们可以知道，为什么在引入 `com.android.application` 之后才能使用 `android` 配置块，这也意味着，`android` 所能提供的配置依赖于 `AppExtension` 这个类。

这里我们简单就 `defaultConfig` 这个做下分析，先看下这个类在 `AppExtension` 中的配置，因为 `AppExtension` 直接调用超类 `TestedExtension` 的构造方法，而 `TestedExtension` 又会调用超类 `BaseExtension` 所以我们直接看到 `BaseExtension`：

``` java
private final DefaultConfig defaultConfig;

defaultConfig = objectFactory.newInstance(
                        DefaultConfig.class,
                        BuilderConstants.MAIN,
                        project,
                        objectFactory,
                        extraModelInfo.getDeprecationReporter(),
                        project.getLogger());
```

这里我们先不关心 `objectFactory.newInstance` 只需要知道，当我们在 `android` 配置块中，配置 `defaultConfig` 实质上也是，对 `DefaultConfig` 的配置，而 `DefaultConfig` 又继承于 `DefaultProductFlavor`，相关的配置同时也是 `DefaultProductFlavor` 的成员属性：

> DefaultProductFlavor 位于 `builder-3.1.2.jar`
>
> Android Gradle Plugin 相关代码分别位于 `gradle-3.1.2.jar`、`gradle-api-3.1.2.jar`、`gradle-core-3.1.2.jar`
>
> 如果想查看源码，则下载 sources 包，比如 [builder-3.1.2-sources.jar](https://dl.google.com/dl/android/maven2/com/android/tools/build/builder/3.1.2/builder-3.1.2-sources.jar)，其他 jar 下载方法类似
>
> 心好累，各种继承。。。

``` java
public class DefaultProductFlavor extends BaseConfigImpl implements ProductFlavor {
    private static final long serialVersionUID = 1L;

    @NonNull
    private final String mName;
    @Nullable
    private String mDimension;
    @Nullable
    private ApiVersion mMinSdkVersion;
    @Nullable
    private ApiVersion mTargetSdkVersion;
    @Nullable
    private Integer mMaxSdkVersion;
    @Nullable
    private Integer mRenderscriptTargetApi;
    @Nullable
    private Boolean mRenderscriptSupportModeEnabled;
    @Nullable
    private Boolean mRenderscriptSupportModeBlasEnabled;
    @Nullable
    private Boolean mRenderscriptNdkModeEnabled;
    @Nullable
    private Integer mVersionCode;
    @Nullable
    private String mVersionName;
    @Nullable
    private String mApplicationId;
    @Nullable
    private String mTestApplicationId;
    @Nullable
    private String mTestInstrumentationRunner;
    @NonNull
    private Map<String, String> mTestInstrumentationRunnerArguments = Maps.newHashMap();
    @Nullable
    private Boolean mTestHandleProfiling;
    @Nullable
    private Boolean mTestFunctionalTest;
    @Nullable
    private SigningConfig mSigningConfig;
    @Nullable
    private Set<String> mResourceConfiguration;
    @NonNull
    private DefaultVectorDrawablesOptions mVectorDrawablesOptions;
    @Nullable
    private Boolean mWearAppUnbundled;
}
```

### 重复配置处理

当我们项目不止只有一个 app module 时，那么在其他 module 中，例如下面这些配置都需要去配置下：

``` groovy
defaultConfig {
        minSdkVersion 21
        targetSdkVersion 27
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"

    }
```

这样就显得很繁琐又不灵活，比如现在很多项目都开始进行组件化，有的 module 有时候是 `com.android.application`，有的时候又是  `com.android.library` 这样又得根据条件写两套配置，如果其中一个更改了，还得去挨个 module 中更改。

而利用 Gradle 多项目构建，我们可以将这些重复配置都统一处理，还可以根据属性自动导入不同的 plugin。首先为了统一管理各种版本号，我们先新建一个 *constants.gradle* 将一些常量写入到 ext 中：

``` groovy
project.ext {
    compileSdkVersion = 27
    
    minSdkVersion = 21
    
    targetSdkVersion = 27
    
    versionCode = 1
    
    versionName = "1.0"
}
```

>  ext 实际上是 [ExtraPropertiesExtension](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html) 类型，它类似 map，可以存储 key/value 键值对。ext 也是扩展的一种，可以通过 `project.extensions.getByName('ext')` 获取实例，所有的 [`ExtensionAware`](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtensionAware.html) 都拥有一个 `ext`，而 Project 又是继承于  `ExtensionAware`

接着我们在各个 module 中创建一个 `gradle.properties` ，这个文件默认会被读取，在其中定义一个属性 `isApplication`，根据这个属性导入不同的 plugin：

``` groovy
// library module
isApplication=false

// application module
isApplication=true
```

接着创建 *init.gradle*，在 `subprojects` 中配置，这里的配置对所有的子项目都会生效：

``` groovy
subprojects {
    if (project.isApplication.toBoolean()) {
        // 当前 module 导入 application plugin
        project.plugins.apply("com.android.application")
    } else {
        project.plugins.apply("com.android.library")
    }
}
```

> 这里有点地方需要注意下，因为属性的获取都是通过 String 类型，所以我们需要明确使用 `toBoolean()` 去转化成布尔值

最后要想使得这些 gradle 文件生效，需要将它们 apply 进去，因为 Gradle 默认只会识别 `build.gradle`，还记得我们在 [Gradle多项目构建](https://linxiaotao.github.io/2018/04/29/Gradle%E5%A4%9A%E9%A1%B9%E7%9B%AE%E6%9E%84%E5%BB%BA/) 中说到，*Gradle 是边读取边执行的* ，这意味着你需要在使用到 `android` 构建块之前，将 plugin apply 进去，所以我们将这些 gradle 文件在根目录的 *build.gradle* 中 apply，因为默认根目录的优先级最高：

``` groovy
// constants.gradle 需要在 init.gradle，因为 init.gradle 后面会使用到 constants.gradle 中定义的值，理由如上
apply from: 'gradle/constants.gradle'
apply from: 'gradle/init.gradle'
```

我们将 module 下 *build.gradle* 中的 apply 代码删除，重新同步下，可以发现 `android` 配置块依然生效，这意味着，我们 apply plugin 操作成功了。

在处理完 plugin 之后，我们可以将 `android` 配置块中的重复配置也在 *init.gradle* 中去配置：

``` groovy
subprojects {
    if (project.isApplication.toBoolean()) {
        // 当前 module 导入 application plugin
        project.plugins.apply("com.android.application")
    } else {
        project.plugins.apply("com.android.library")
    }
    android {
        compileSdkVersion rootProject.ext['compileSdkVersion']
        defaultConfig {
            minSdkVersion rootProject.ext['minSdkVersion']
            targetSdkVersion rootProject.ext['targetSdkVersion']
            versionCode rootProject.ext['versionCode']
            versionName rootProject.ext['versionName']
            testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        }
        buildTypes {
            release {
                minifyEnabled false
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }
    }

}
```

是不是非常方便，这样统一的配置你只需要写一次，不只是 `android` 配置块可以这么处理，其他的配置也可以这么处理。

### 依赖管理

一般情况下，我们会在各个 module 下去编写所需的依赖配置，比如通过 `implementation 'com.android.support:appcompat-v7:27.1.1'` 去从 maven 中心仓库或者 jcenter 中下载依赖，但这样有个不好的地方就是依赖配置太分散了，只能到每个 module 目录中的 *build.gradle* 去管理，如果我们将这个写到统一的 Gradle 构建脚本中，是不是会更好点，我们同样创建个 *dependencies.gradle*，将共同的依赖写到 `subprojects` ，特有的依赖写到单独的项目配置中，同样这个文件需要在根目录的 *build.gradle* 中 apply：

``` groovy
subprojects {
    dependencies {
        testImplementation rootProject.ext.dependencies['junit']
        androidTestImplementation rootProject.ext.dependencies['runner']
        androidTestImplementation rootProject.ext.dependencies['espresso-core']
    }
}

project(':mylibrary') {
    dependencies {
        api fileTree(dir: 'libs', include: ['*.jar'])
        api rootProject.ext.dependencies['appcompat-v7']
        api rootProject.ext.dependencies['constraint-layout']
    }
}

project(':app') {
    dependencies {
        implementation fileTree(dir: 'libs', include: ['*.jar'])
        implementation project(":mylibrary")
    }
}
```

我们可以将共同的依赖写到 `subprojects` 块，单独项目的配置写到各自的配置块下。

现在我们再来解决另外一个常见的问题：依赖冲突，比如不同的依赖项各自又依赖于同一个库的不同版本，默认 [Gradle 会用这个重复库的最高版本](https://docs.gradle.org/current/userguide/troubleshooting_dependency_resolution.html#sub:version_conflicts)，但如果你配置了 `resolutionStrategy.failOnVersionConflict()` 即当出现版本冲突则失败，或者版本选择不合适，等等，就会出现依赖冲突问题。

我们通过个例子来证实默认的处理策略，我们在 app 模块中去依赖  `com.android.support:design:27.1.1` 和 `com.android.support:recyclerview-v7:26.1.0` ，而 `design` 会自动去依赖 `com.android.support:recyclerview-v7:27.1.1`，依赖关系我们可以通过 `./gradlew :app:dependencies` 这个 Task 去查看，这里我们只查看 *releaseCompileClasspath* 依赖配置中 `design` 和 `recyclerview` 的情况：

```
+--- project :mylibrary
|    \--- com.android.support:design:27.1.1
|         +--- com.android.support:support-v4:27.1.1
|         |    +--- com.android.support:support-compat:27.1.1 (*)
|         |    +--- com.android.support:support-media-compat:27.1.1
|         |    |    +--- com.android.support:support-annotations:27.1.1
|         |    |    \--- com.android.support:support-compat:27.1.1 (*)
|         |    +--- com.android.support:support-core-utils:27.1.1 (*)
|         |    +--- com.android.support:support-core-ui:27.1.1 (*)
|         |    \--- com.android.support:support-fragment:27.1.1 (*)
|         +--- com.android.support:appcompat-v7:27.1.1 (*)
|         +--- com.android.support:recyclerview-v7:27.1.1
|         |    +--- com.android.support:support-annotations:27.1.1
|         |    +--- com.android.support:support-compat:27.1.1 (*)
|         |    \--- com.android.support:support-core-ui:27.1.1 (*)
|         \--- com.android.support:transition:27.1.1
|              +--- com.android.support:support-annotations:27.1.1
|              \--- com.android.support:support-compat:27.1.1 (*)
\--- com.android.support:recyclerview-v7:26.1.0 -> 27.1.1 (*)
```

可以看到我们依赖的 26.1.0 已经被替换成 27.1.1，如果这是我们想要的结果，那就很理想，但是如果我们想要去定义依赖冲突的策略呢？同样的，Gradle 也提供了相应的 API。

还是上面的例子，如果我们不想要使用 `design` 中的 `recyclerview` 而想要我们单独依赖的 `recyclerview-v7:26.1.1` 这里我们有几种方式解决：

1. 不传递依赖

   ``` groovy
   api(rootProject.ext.dependencies['design']) {
               transitive = false
   }
   ```

2. 强制使用当前版本

   ``` groovy
   implementation(rootProject.ext.dependencies['recyclerview']) {
               force = true
   }
   ```

其他的解决方案可以参考[官方文档](https://docs.gradle.org/current/userguide/introduction_dependency_management.html)

这里我们使用 Gradle 提供的 [resolutionStrategy](https://docs.gradle.org/current/userguide/customizing_dependency_resolution_behavior.html) 方案： 

``` groovy
subprojects {
    configurations.all {
        resolutionStrategy {
            force rootProject.ext.dependencies['recyclerview']
        }
    }
}
```

`configurations` 表示当前项目的依赖配置容器，`configurations.all` 表示当前项目的所有依赖配置，上面的配置表示，遍历当前项目的所有依赖配置，设置其依赖解决策略为 **强制使用指定的版本**

## 改造结束

现在我们再来对比下，默认生成的构建配置和改造后的构建配置：

``` groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 27
    defaultConfig {
        applicationId "com.example.leo.gradlecaseproject"
        minSdkVersion 21
        targetSdkVersion 27
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:27.1.1'
    implementation 'com.android.support.constraint:constraint-layout:1.1.0'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
}

```

``` groovy
android {
    defaultConfig {
        applicationId "com.example.leo.gradlecaseproject"
    }
}
```

是不是简洁了不少，重复的配置项我们交给统一的配置文件去做，依赖配置也是如此，包括依赖冲突的解决策略，但是 Gradle 能实现的不仅仅如此，还有强大的自定义 Gradle Plugin 我们还没讲到。

## 总结

通过这篇文章，不仅仅是教会大家如何去编写简单的 Gradle 构建脚本，更重要的想让大家对 Gradle 有个初步的认识，知道 Gradle 的配置块是如何生成的，更多知识可以通过[官方文档](https://docs.gradle.org/current/userguide/userguide.html)去获取。