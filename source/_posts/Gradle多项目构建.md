---
title: Gradle多项目构建
date: 2018-04-29 09:57:51
categories: Gradle
tags:
---

## 参考

[multi_project_builds](https://docs.gradle.org/current/userguide/multi_project_builds.html)

## 概述

在使用 Android Studio 作为 IDE 之后，Android 项目就开始使用 Gradle 作为构建脚本，Gradle 的优点就不用我多说了，使用 Groovy 作为开发语言，配合各种 Gradle 插件和 DSL 可以实现多样化的构建过程。

Gradle 能讲的知识点很多，本文主要讲的是 Gradle 在多项目构建上提供的一些便捷的功能，希望能给大家一些启发。

## 名词解释

* 构建脚本：本文所说的构建脚本指的是 Gradle 文件，以 `.gradle` 为后缀的文件
* 项目：在多项目构建中，有根项目和子项目。根项目的称呼是相对的，以执行 gradle 命令的目录为根项目，当前目录的子目录称为子项目

## Gradle 多项目构建

首先我们对 Gradle 多项目构建先做下了解，这里所涉及的知识点大部分来源于参考文档。

### 跨项目配置

Gradle 提供了在任何构建脚本中访问任何项目，比如可以使用 `allprojects` 来对所有项目进行配置：

``` groovy
allprojects {
    task hello {
        doLast {
            println "Hello Gradle"
        }
    }
}
```

这个例子我们对所有项目都创建了一个叫 "hello" 的 task，如果你只是想对当前项目的子项目进行配置：

``` groovy
subprojects {
    task hello {
        doLast {
            println "Hello Gradle"
        }
    }
}
```

当然你也可以针对单个项目进行配置：

``` groovy
project (':project') {
    task hello {
        doLast {
            println "Hello Gradle"
        }
    }
}
```

上面所说的操作可以在任何一个构建脚本上执行，所以你可以选择统一写到单独的构建脚本上，再通过 `apply from: "xxx.gradle"` 应用进来。

### 边读取边解释

可能有的同学会问，为什么上面要用 doLast，可以不用 doLast，直接写可以吗？

当前可以，但是执行的时机就不一样了，doLast 从字面意思来看，表示在最后执行，那么这个最后指的是什么之后呢。答案就是项目配置评测(evaluation)之后，简单来讲，当 Gradle 开始执行时，会先从根目录的 `settings.gradle` 中读取参与构建的项目，即只有将子项目 `include` 才能参与构建，接着 Gradle 会在每个项目的根目录下读取 `build.gradle` 如果存在的话，即 `build.gradle` 并不是必须的。默认情况下，Gradle 会先读取根项目的配置，即当你执行 Gradle 命令时所在目录的项目。接着按字母排序，读取子项目的配置，当项目配置评测完成之后，再执行对应的 task.doLast。

那有的同学又会问了，那如果直接写，执行的顺序是什么呢？是在评测之后，doLast 之前吗？这可能是写习惯应用程序的同学最常见的误区了，之前博主也是这么想的，后来经过同事的点拨，**Gradle 是边读取边解释**，有点口语化，但确实如此，回到上面的问题，如果直接写在 task 的代码，会在 Gradle 读取到这一行的时候执行，即可能先于其他还没评测的项目执行，我们可以通过一个例子来看下：

这是项目的目录结构：

```
├── build.gradle
├── config.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradle.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
├── sub1
│   └── build.gradle
└── sub2
    └── build.gradle
```

这里我们包含了两个子项目，分别是 `sub1` 和 `sub2`，在每个项目的 `build.gradle` 我们都加上 Log 打印：

``` groovy
println "$project.name init"

println "$project.name  end"
```

接着我们在根项目的 `build.gradle` 即最外层目录下，添加一个 task：

``` groovy
task hello {
    println "我直接运行"
    doLast {
        println "我运行在 doLast"
    }
}
```

记得在根目录下执行 `./gradlew -q hello`，参数 `-q` 只打印我们的 log，结果如下：

```
MyApplication init
我直接运行
MyApplication  end
sub1 init
sub1 end
sub2 init
sub2  end
我运行在 doLast
```

结果是不是和我们想的一样。接着我们在 sub1 的 `build.gradle` 中增加以下代码：

``` groovy
println "当前 sub2 是否存在 username 属性：" + rootProject.project(":sub2").findProperty('username')

rootProject.project(":sub2") {
    ext {
        username = "Leo"
    }
}

println "当前 sub2 是否存在 username 属性：" + rootProject.project(":sub2").findProperty('username')
```

上面我们说到，在任何一个构建脚本中，都可以去配置其他项目，所以我们在 sub1 中往 sub2 添加一个变量，然后在 sub2 中将它打印出来：

``` groovy
println "我的名字是 $project.ext.username"
```

执行下：

``` 
MyApplication init
我直接运行
MyApplication  end
sub1 init
当前 sub2 是否存在 username 属性：null
当前 sub2 是否存在 username 属性：Leo
sub1  end
sub2 init
我的名字是 Leo
sub2  end
我运行在 doLast
```

是不是非常有意思，要记住：**Gradle 边读取边解释，先评测项目配置，再执行相应的 Task.doLast**

### 执行规则

Gradle 执行时，从当前执行的目录开始查看项目结构，即当前目录为根项目，根据目录下的 `setting.gradle` 去评估子项目的配置，执行相应的 Task，我们同样来看个例子：

```
.
├── build.gradle
├── config.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradle.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
├── sub1
│   └── build.gradle
└── sub2
    ├── build.gradle
    ├── settings.gradle
    └── sub3
        └── build.gradle
```

我们在 sub2 目录下创建一个新的目录 sub3，其中的 `build.gradle` 如下：

``` groovy
println "rootProject is $rootProject.name"
```

代码非常简单，就是输出当前根项目的名称。我们先在最外层的目录下，执行 `./gradlew` 输出如下：

```
rootProject is MyApplication
```

记得将 sub3 `include` 到 `settings.gradle` 可以看到当前的根项目名称即为当前运行的目录，接着我们切换到 sub2 目录下执行同样的命令，输出如下：

```
rootProject is sub2
```

### 执行顺序

这里我们所说的执行顺序，包括两个方面，第一是项目的评测顺序，第二是各个项目 Task 的执行顺序。

上面我们提到了项目评测顺序是，先评测根项目，接着按字母顺序评测子项目。那我们如果想改变默认顺序，又不想修改名称呢。Gradle 提供了 `evaluationDependsOn` 用于声明某个项目的评测依赖于其他项目的评测，所以会在依赖项目评测完成之后进行。在上面的例子中，sub1 默认会在 sub2 之前执行，但是如果我们在 sub1 项目中增加如下配置：

``` groovy
println "$project.name init"

evaluationDependsOn(':sub2')

println "$project.name  end"
```

执行 `./gradlew -q` 输出如下：

``` 
sub1 init
sub2 init
sub2  end
sub1  end
```

结合我们之前说到的，Gradle 是边读取边解释的，那么 `sub1 end` 在最后输出就不难理解了。

Gradle 还提供了 `evaluationDependsOnChildren` 声明子项目先于根项目进行评测。

Task 也是类似的，Gradle 提供了 `dependsOn` 去声明某个 Task 依赖于其他 Task，所以会在依赖 Task 执行后执行。使用例子如下：

``` groovy
task hello(dependsOn: ':project:task') {
    
}
```

### 项目解耦

Gradle 允许任何项目去访问当前多项目构建中其他项目，虽然这很灵活，但使用不当却会导致项目耦合程度高。

例如，我们通过会在根项目中使用 `allprojects` 或者 `subprojects` 进行项目配置注入，但如果我们在子项目中去对其他项目进行配置注入，就会导致项目耦合。同时如果在子项目构建时，去更改其他项目的配置，这同样也会导致项目耦合，并且这两个操作都可能会影响到 **并行模式** 和 **按需配置** 的正确性。

为了更好的使用配置注入和其他优化选项，我们应该：

* 避免在子项目 `build.gradle` 引用其他子项目，更适合在根项目中进行配置注入
* 避免在构建时更改其他的项目的配置

### 多项目编译和测试

在 Java 插件的 `build` task 通常是用于对单个项目进行编译、测试和应用代码格式化检查等等。在多项目中构建中你可能想要将 task 作用于指定范围内的项目，那么 `buildNeeded` 和 `buildDependents` task 可以帮助你。

> 接下来的例子都是从官方文档中翻译而来的

比如在这个[例子](https://docs.gradle.org/current/userguide/multi_project_builds.html#javadependencies_2)中，`:services:personservice` 项目依赖于 `:api` 和 `:shared` 项目，同时 `:api` 项目也依赖于 `:shared`。

当我们执行 `./gradlew :api:build` 时，输出可能如下：

``` 
> gradle :api:build
> Task :shared:compileJava
> Task :shared:processResources
> Task :shared:classes
> Task :shared:jar
> Task :api:compileJava
> Task :api:processResources
> Task :api:classes
> Task :api:jar
> Task :api:assemble
> Task :api:compileTestJava
> Task :api:processTestResources
> Task :api:testClasses
> Task :api:test
> Task :api:check
> Task :api:build

BUILD SUCCESSFUL in 0s
9 actionable tasks: 9 executed
```

可以看到，当我们只执行 `:api` 项目的 `build` task，同时也会执行其依赖项目 `:shared` 部分的 task，如果我们确定对 `:api` 项目的修改不会影响 `:share` 项目，可以使用 `-a` 选项参数，这个参数可以让 Gradle 去缓存依赖项目生成的 jars，不重新去编译依赖项目，现在我们增加 `-a` 参数，`./gradlew -a :api:build`，输出可能如下：

```
> gradle -a :api:build
> Task :api:compileJava
> Task :api:processResources
> Task :api:classes
> Task :api:jar
> Task :api:assemble
> Task :api:compileTestJava
> Task :api:processTestResources
> Task :api:testClasses
> Task :api:test
> Task :api:check
> Task :api:build

BUILD SUCCESSFUL in 0s
6 actionable tasks: 6 executed
```

可以看到 `-a` 选项起作用了。

如果你刚刚从版本控制工具中更新了 `:api` 项目依赖的项目，你可能不仅仅想要只执行编译，可能想要去测试它们，那么 `buildNeeded` task 将测试所有依赖项目测试运行时的配置。执行 `./gradlew :api:buildNeeded`，可能输出如下：

```
> gradle :api:buildNeeded
> Task :shared:compileJava
> Task :shared:processResources
> Task :shared:classes
> Task :shared:jar
> Task :api:compileJava
> Task :api:processResources
> Task :api:classes
> Task :api:jar
> Task :api:assemble
> Task :api:compileTestJava
> Task :api:processTestResources
> Task :api:testClasses
> Task :api:test
> Task :api:check
> Task :api:build
> Task :shared:assemble
> Task :shared:compileTestJava
> Task :shared:processTestResources
> Task :shared:testClasses
> Task :shared:test
> Task :shared:check
> Task :shared:build
> Task :shared:buildNeeded
> Task :api:buildNeeded

BUILD SUCCESSFUL in 0s
12 actionable tasks: 12 executed
```

有时候你重构了 `:api` 的某些代码，想要测试依赖于 `:api` 项目的其他项目，那么可以使用 `buildDependents`，它可以测试编译依赖指定的项目的所有项目，运行 `./gradlew :api:buildDependents` 输出如下：

``` 
> gradle :api:buildDependents
> Task :shared:compileJava
> Task :shared:processResources
> Task :shared:classes
> Task :shared:jar
> Task :api:compileJava
> Task :api:processResources
> Task :api:classes
> Task :api:jar
> Task :api:assemble
> Task :api:compileTestJava
> Task :api:processTestResources
> Task :api:testClasses
> Task :api:test
> Task :api:check
> Task :api:build
> Task :services:personService:compileJava
> Task :services:personService:processResources
> Task :services:personService:classes
> Task :services:personService:jar
> Task :services:personService:assemble
> Task :services:personService:compileTestJava
> Task :services:personService:processTestResources
> Task :services:personService:testClasses
> Task :services:personService:test
> Task :services:personService:check
> Task :services:personService:build
> Task :services:personService:buildDependents
> Task :api:buildDependents

BUILD SUCCESSFUL in 0s
17 actionable tasks: 17 executed
```

最后，如果你在根项目执行的任何 task 都会导致所有项目中存在同名的 task 的执行。

### 属性和方法的继承

在根项目中声明的属性和方法都会继承到子项目中，这是配置注入的替代方式。而配置注入不支持方法，

### 其他选项

#### 并行模式

可以使用 `—parallel` 开启并行模式，这可以减少项目构建时间

#### 按需配置

可以使用 `--configure-on-demand` 开启按需配置，这同样可以减少构建配置时间

## 总结

在上面的篇幅我们着重讲解了 Gradle 对多项目构建的支持，包括跨项目配置，多项目的执行规则和执行顺序，配置注入等等。但理论知识毕竟只是纸上谈兵，下一篇文章会通过具体的项目配置，来讲解实际的使用。