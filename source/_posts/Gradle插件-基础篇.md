---
title: Gradle插件-设计
date: 2018-05-16 09:20:58
categories:
tags: Gradle
---

## 前言

本文是 Gradle 系列的第三篇，前两篇都是关于 Gradle 多项目构建，有兴趣的同学可以去翻看下。Gradle 系列作者会一直更新下去，这些知识大部分都来自于 [Gradle 用户手册](https://docs.gradle.org/current/userguide/userguide.html)，但我并不想写成翻译类型的文章，从最基础的知识开始深入，因为这样前期枯燥的理论知识会让人感到厌倦，所以从用户最常用的知识入手，再穿插必要的基础知识，最终达到知识的融会贯通。因为作者从事 Android 开发，所以会更多提及 Android 中关于 Gradle 的知识。

## Gradle插件

Gradle 是非常强大的构建工具，所有的知识都围绕着 project 和 task 这两个知识点。project 在前面的文章中我们已经简单介绍了，task 现在是穿插着讲，后续会考虑单独写成文章。而我们今天讲的 plugin 对于大部分的 Gradle 使用者来说，可能是一个熟悉又陌生的知识点，熟悉是因为用的太频繁了，比如 Android 项目中，我们都会使用以下配置：

``` groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion xxx
    buildToolsVersion xxx
    defaultConfig {
        
    }
}
```

但另外一方面，我们对 plugin 的实现又似懂非懂，使用 Android Application Plugin 后，`android {}` 配置块是如何生成的，又有哪些配置可以用。现在有很多自定义的 Gradle Plugin，比如使用 AOP 实现的代码插桩，打点等等功能的库，那么这些库又是如何通过 Plugin 功能实现的。

接下来，我们将基于 [Gradle 用户手册](https://docs.gradle.org/current/userguide/userguide.html) 中关于 Plugin 的知识，从 Plugin 的理论知识到自定义实现 Plugin 写一系列文章。

> 友情提示：如果想深入理解 Gradle 知识，最好还是从 Gradle 官方文档入手，比如 [Gradle 用户手册](https://docs.gradle.org/current/userguide/userguide.html)，[API 文档](https://docs.gradle.org/current/javadoc/) 等等，如果只是想快速了解，那么可以继续往下读。

### 设计

Gradle Plugin 的设计应该符合以下准则：

#### 架构

* 可复用的逻辑应该写成二进制形式

  Gradle Plugin 有两种形式：脚本插件和二进制插件。脚本插件就是普通的构建脚本，虽然脚本插件也可以组织构建逻辑，但是它不能进行测试，也不能定义可复用的类型，难于长期维护。

* 考虑性能影响

  虽然在设计 Gradle Plugin 时，可以实现你能想到的任何逻辑，但是应该考虑对构建时间的影响。

* 约定大于配置

  Gradle Plugin 的设计应该秉承**约定大于配置**的准则，尽量减少用户陷入繁琐的配置中，又提供配置入口给具有自定义需求的用户。比如 Android Application Plugin 的 ***sourceSets*** 配置，如果不进行配置，会使用默认的目录，可以使用  `./gradlew :app:sourceSets` 打印默认使用的目录。

* 功能与约定

  为了让用户能够更灵活地使用你设计的 Plugin，你应该提供更为灵活的使用方式，其中一个方式就是，将实现功能与通用配置分离开来。比如： [Java Base Plugin](https://docs.gradle.org/current/userguide/standard_plugins.html?_ga=2.84655903.747719590.1526541985-590693097.1523454314#sec:base_plugins) 和 [Java Plugin](https://docs.gradle.org/current/userguide/java_plugin.html?_ga=2.72588053.747719590.1526541985-590693097.1523454314)：

  * Java Base Plugin：提供了 ***sourceSets*** 配置属性，用于定义源文件的目录，但它并不会被用于任何构建任务
  * Java Plugin：则是在 Java Base Plugin 提供的 ***sourceSets*** 配置上读取源文件，同时定义了具体的构建任务

  如果用户想实现更深层次的定制，则可以继承于 Java Base Plugin 实现自定义构建任务。

  在实现自定义 Plugin 可以这样实现：

  假设 `BasePlugin` 实现了通用配置：

  ``` groovy
  public class BasePlugin implements Plugin<Project> {
      public void apply(Project project) {
          
      }
  }
  ```

  `MyPlugin` 则是基于 `BasePlugin` 实现了特定功能：

  ``` groovy
  public class MyPlugin implements Plugin<Project> {
      public void apply(Project project) {
          proejct.getPlugins().apply(BasePlugin.class);
      }
  }
  ```

  #### 技术实现

* 倾向于使用静态类型语言来实现

  Gradle Plugin 的实现语言可以是任何 JVM 语言（即编译产物能在 JVM 上运行），可以是 Java、Kotlin、Groovy 等等。

  但建议使用 Java 或 Kotlin 等静态类型语言来实现，以降低出现不兼容的可能。

* 使用 Gradle 公开 API 实现

  当实现 Gradle Plugin，需要指定使用的 Gradle API 版本，例如以下配置：

  ``` groovy
  dependencies {
      compile gradleAPi()
  }
  ```

  但因为 Gradle 的历史原因，公开和内部的 API 并没有分离开来，为了确保与各个 Gradle 版本的兼容性，请尽量使用公开的 API

  > 能在 [DSL 文档](https://docs.gradle.org/current/dsl/?_ga=2.131010252.258862271.1526633053-590693097.1523454314) 和  [DOC 文档](https://docs.gradle.org/current/javadoc/?_ga=2.131010252.258862271.1526633053-590693097.1523454314) 中找到的，即为公开的 API

* 最大程度地减少外部库的依赖

  当我们在编写 Plugin 时，可能会习惯性地去引入某个功能齐全的外部库，比如：Guave，但是这同时也会带来一些问题，可以使用 `buildEnvironment` task 查看构建环境依赖关系，包括 Plugin：

  ```
  classpath
  +--- com.android.tools.build:gradle:2.3.3
  |    \--- com.android.tools.build:gradle-core:2.3.3
  |         +--- com.android.tools.build:builder:2.3.3
  ```

  因为 Gradle Plugin 并没有独立的类加载器运行环境，所以这些依赖可能会与其他 Plugin 的依赖发生冲突。

### 例子

上面是自定义 Gradle Plugin 的一些规范和注意事项，下面我们通过一个比较简单的自定义 Plugin 来实践下。

这个 Plugin 的功能很简单，创建一个 task，获取配置的属性，再打印出来。

上面我们说到，Plugin 可以分为脚本插件和二进制插件两种，脚本插件就是 Gradle 文件，而二进制文件的可以像普通依赖一样，从中央仓库上下载，还有一种方式是，将源码存放到当前项目下的 ***buildSrc*** module，当应用插件时，会从中查找。为了方便起见，我们使用后面那种方式。

* 首先，我们创建一个名为 ***buildSrc*** 的 module，记得在 `setting.gradle` 中配置，在该 module 的根目录下创建 `build.gradle`，因为我们打算用 Java 实现，所以先引入 Java Plugin，同时依赖 Gradle API。

  > Gradle 同时也提供了一个 Plugin 用于简化这些步骤，具体可以查看 [java-gradle-plugin](https://docs.gradle.org/4.7/userguide/java_gradle_plugin.html?_ga=2.132564685.258862271.1526633053-590693097.1523454314)

  ``` groovy
  apply plugin: 'java-library'
  
  dependencies {
      implementation fileTree(dir: 'libs', include: ['*.jar'])
      implementation gradleApi()
  }
  
  sourceCompatibility = "1.8"
  targetCompatibility = "1.8"
  ```

* 接着，创建一个 Task 类型，命名为 `HelloWorld`：

  ``` java
  public class HelloWorld extends DefaultTask {
      
      public String userName;
  
      @TaskAction
      public void run() {
          System.out.println("Hello World，" + userName);
      }
  }
  ```

  一般继承于 `DefaultTask` 也可以选择其他 Task 基类，`@TaskAction` 是必须的，用于注解方法为 Task 运行时执行的代码，`userName` 是可选配置。

* 其实上一步做完，我们已经可以在其他 module 下的 gradle 文件中直接引用这个 Task 类型，比如：

  ``` groovy
  import com.example.gradle.HelloWorld
  
  task hello(type: HelloWorld) {
      userName = "leo"
  }
  ```

  执行 `./gradlew :app:hello` 输出 "Hello World，leo"

  > 注意：上面这样做的前提是，这部分的代码位于 ***buildSrc*** module，这也是约定配置，符合**约定大于配置**规范

  在我们这个例子中，我们通过 Plugin 去动态创建一个 Task：

  ``` java
  public final class HelloWorldPlugin implements Plugin<Project> {
      @Override
      public void apply(Project project) {
          project.getTasks().create("pluginHello", HelloWorld.class, helloWorld ->
                  helloWorld.userName = "leo");
      }
  }
  ```

  实现自定义 Plugin 需要实现于 Plugin，接着在 `apply()` 中添加一个名为 "pluginHello"，类型为 `HelloWorld` 的 Task。

* 至此，我们已经写好了 Plugin 的逻辑代码，要想在当前项目中的其他 module 中引用 ***buildSrc*** 中的 Plugin，需要给 `HelloWorldPlugin` 指定一个 ID，注：直接写在构建脚本中的插件则不需要。具体配置如下：

  在 `src/main/resources/META-INF/gradle-plugins/` 目录下创建一个配置文件，命名规则为：***[PluginID].properties***，在我们这个例子中为：`com.example.hello.properties`：

  ``` properties
  implementation-class=com.example.gradle.HelloWorldPlugin
  ```

  `implementation-class` 表示 Plugin 类

  > Plugin ID 应该符合以下规定：
  >
  > * 可以包含任何字母、字符，'.'、'-'
  > * 必须至少包含一个 '.' 分隔命名空间和插件名称
  > * 按照惯例，使用域名反向小写作为命名空间
  > * 按照惯例，命名空间只使用小写字母
  > * `org.gradle` 和 `com.gradleware` 命名空间不能被使用
  > * 不能使用 '.' 作为开始或结束字符
  > * 不能使用连续的 '.' 字符

* 至此，我们已经写好了一个简单自定义 Plugin，我们在 app module 下的 `build.gradle` 引入这个它：

  ```
  apply plugin: 'com.example.hello'
  ```

  执行 `./gradlew :app:pluginHello` 输出 "Hello World，leo"

  > 上面我们提到了 java-gradle-plugin，如果使用这个 Plugin 的话，可以无需手动配置 properties，只需在 gradle 中配置如下即可：
  >
  > ``` groovy
  > gradlePlugin {
  >     plugins {
  >         helloPlugin {
  >             id = 'com.example.hello'
  >             implementationClass = 'com.example.gradle.HelloWorldPlugin'
  >         }
  >     }
  > }
  > ```

### 插件源码

在上面的例子中，我们将代码写在了 ***buildSrc*** module 中，那么除此之外还有：

* 构建脚本

  可以将 Plugin 的源码直接包含在构建脚本中，这样子可以方便地在当前构建脚本直接使用该 Plugin，但是无法在其他构建脚本中使用。

  ***build.gradle***

  ``` groovy
  class TestPlugin implements Plugin<Project> {
      void apply(Project project) {
          project.tasks.create('pluginTest') {
              doLast {
                  println "This is Test"
              }
          }
      }
  }
  
  apply plugin: TestPlugin
  ```

  > 需要注意：上面的书写顺序，还记得我们之前说的**边读取边解释**

* 单独的项目

  你可以为 Gradle Plugin 创建一个单独的项目，这种方式和 ***buildSrc*** 类似，并且可以生成 JAR 上传到中央仓库提供给其他开发者使用，而 ***buildSrc*** 只能在当前项目中使用。

  ### 结束语

由于篇幅的限制，基础篇我们就讲到这里，后续的文章还会更深入地讲解 Gradle Plugin 知识。