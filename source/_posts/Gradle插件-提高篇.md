---
title: Gradle插件-提高篇
date: 2018-05-21 08:44:35
categories: Gradle
tags:
---

## 前言

在上一篇文章 [Gradle插件-基础篇](https://linxiaotao.github.io/2018/05/16/Gradle%E6%8F%92%E4%BB%B6-%E5%9F%BA%E7%A1%80%E7%AF%87/) 中，我们学习了 Plugin 的设计规范，并且通过一个非常简单的例子对自定义 Plugin 有了初步认识，在这篇文章中，我们来继续学习 Gradle Plugin 更为深入的知识点。

> 本文参考 [Gradle用户手册](https://guides.gradle.org/implementing-gradle-plugins/?_ga=2.31320828.258862271.1526633053-590693097.1523454314#writing-and-using-custom-task-types) 

## Gradle插件

> 本文涉及的所有源码都位于 [github](https://github.com/LinXiaoTao/GradleCaseProject)

### 简单扩展

当我们引入 Android Plugin 时，在 `android{}` 有一些常用的配置，比如：

``` groovy
android {
    compileSdkVersion 27
}
```

如果我们想在自定义的 Plugin 中也使用这样的配置，可以使用 Gradle 提供的扩展功能：`Extensions` ，在第一篇文章中我们自定义了一个简单的 `com.example.hello` 插件，我们现在就在这个基础上添加一个扩展功能，最终实现的配置如下：

``` groovy
hello {
    outFile 'hello'
}
```

向 outFile 指定的文件中写入一句 "Hello World"，如文件不存在则创建它。

为了创建扩展，我们先定义一个数据模型用于读取配置：

``` java
public class HelloModel {
    
    private String outFile;

    public String getOutFile() {
        return outFile;
    }

    public void setOutFile(String outFile) {
        this.outFile = outFile;
    }
}
```

接着使用 `ExtensionContainer` 创建一个扩展：

``` java
project.getExtensions().create("hello", HelloModel.class);
```

`getExtensions(String,Class,Object...)` 这个方法有三个参数，第一个参数为扩展的名称即在 ***gradle*** 文件中使用的配置块，第二个参数为扩展的模型，用于定义可配置的属性，在上面的例子上，只有一个简单的 `outFile` 表示输出文件，第三个参数为可变长参数，表示模型的构造参数，在这个例子中，我们使用默认的无参构造函数。

经过上面的步骤，我们已经创建好了扩展，接下来是读取扩展的属性值，还记得我们之前说的，**Gradle 是边读取边解释**，所以当我们 `apply plugin` 时，会创建好扩展，而扩展的属性值配置只能在 `apply plugin` 之后，所以我们在 plugin 中直接读取扩展属性值是不行的，必须在项目评估完成之后：

``` java 
project.afterEvaluate(project1 -> {
            HelloModel helloModel = project1.getExtensions().getByType(HelloModel.class);
            mLogger.quiet("hello : " + helloModel);
            if (helloModel.getOutFile() == null || helloModel.getOutFile().isEmpty()) {
                throw new GradleException("outFile 不能等于空");
            }
            FileOutputStream fileOutputStream = null;
            try {
                File outFile = project1.file(helloModel.getOutFile());
                if (!outFile.exists()) {
                    outFile.createNewFile();
                }
                fileOutputStream = new FileOutputStream(outFile);
                fileOutputStream.write("Hello World".getBytes());
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if (fileOutputStream != null) {
                    try {
                        fileOutputStream.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
```

逻辑比较简单，我们现在直接运行在 `.gradlew` 即可，可以看到在 app 目录下生成一个 ***hello*** 文件：

```
Hello World
```

### 嵌套扩展

如果要想实现如下 DSL 配置：

``` groovy
android {
    defaultConfig {
        applicationId "com.example.leo.gradlecaseproject"
    }
}
```

Gradle 也支持这种嵌套扩展，同样是上面的例子，我们现在想添加一个 `info` 配置块，将这部分的信息也写入到文件中，最终配置如下：

``` groovy
hello {
    outFile = 'hello'
    info {
        username = "leo"
        email = "linxiaotao1993@vip.qq.com"
    }
}
```

首先我们先创建一个 InfoModel，这里有个要注意的地方：用于实现扩展的类不能是 `final`，因为最终用于扩展的实现类就是继承于这个类。

``` java
static class InfoModel {
        private String username;
        private String email;

        @Inject
        public InfoModel() {
        }

        public String getUsername() {
            return username;
        }

        public void setUsername(String username) {
            this.username = username;
        }

        public String getEmail() {
            return email;
        }

        public void setEmail(String email) {
            this.email = email;
        }

        @Override
        public String toString() {
            return "InfoModel{" +
                    "username='" + username + '\'' +
                    ", email='" + email + '\'' +
                    '}';
        }
    }
```

上面在构造方法中添加 `@Inject` 注解是为了能用 Gradle 提供的 `ObjectFactor` 用于实现依赖注入功能。而在原来的 `HelloModel` 中，我们增加了 `private final InfoModel info;` 同时在构建函数中增加了 `ObjectFactory` 参数：

``` java
public HelloModel(ObjectFactory objectFactory) {
        info = objectFactory.newInstance(InfoModel.class);
}
```

但这样还不够，还需要当 `info` 配置块被调用时所传递的实例：

``` java
public void info(Action<InfoModel> action) {
        action.execute(info);
}
```

注意：这里的方法名和 DSL 中的使用的配置块名称一致。

最后我们在写入文件时，将这部分配置的信息也写入，同时在生成扩展时提供 `ObjectFactor` 实例：

``` java
project.getExtensions().create("hello", HelloModel.class, project.getObjects());

fileOutputStream.write(String.format(
                        Locale.getDefault(),
                        "Hello World\ncreate by %s %s",
                        helloModel.getInfo().getUsername(),
                        helloModel.getInfo().getEmail()).getBytes()
                      );
```

最后执行下就可以看到结果了。

> 可以看到 Gradle 可以很方便地使用依赖注入。

### 配置容器

在 Android 项目中，我们可以去配置不同的构建变体：

``` groovy
flavorDimensions "api", "mode"
    productFlavors {
        demo {
            dimension "mode"
        }
        full {
            dimension "mode"
        }
        minApi24 {
            dimension "api"
        }
        minApi23 {
            dimension "api"
        }
    }
```

如果我们想要实现类似 `productFlavors` 这种动态生成的 DSL 配置块，我们可以使用 `NamedDomainObjectContainer` 来实现，接着上面的例子，我们最终的配置如下：

``` groovy
hello {
    outFile = 'hello'
    info {
        username = "leo"
        email = "linxiaotao1993@vip.qq.com"
    }
    other {
        time {
            value = new SimpleDateFormat("yyyy-MM-dd").format(new Date())
        }
    }
}
```

我们先创建 `ValueModel` 用于表示容器中每一项的格式，注意：这个类必须有个字段为 `name` ，比如上面的例子中，`name` 就是我们配置的 `time`：

``` java
public class ValueModel {

    private final String name;
    private String value;

    public ValueModel(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "ValueModel{" +
                "name='" + name + '\'' +
                ", value='" + value + '\'' +
                '}';
    }

    public String getName() {
        return name;
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }
}
```

> 这个类不能写成内部类，静态内部类也不行

接着，我们在 `HelloModel` 中同样创建 `other` 配置块，需要注意的是，这里我们要使用 `NamedDomainObjectContainer` ：

``` java
public void other(Action<NamedDomainObjectContainer<ValueModel>> action) {
        action.execute(other);
}
```

而 `other` 则是这样子创建的：

``` java
other = project.container(ValueModel.class);
```

最后，同样的，我们将这块的配置也写入到 `hello` 文件中：

``` java
StringBuilder otherString = new StringBuilder();
                helloModel.getOther().forEach(valueModel ->
                        otherString.append(valueModel.getName())
                                .append("：")
                                .append(valueModel.getValue())
                                .append("\n"));
                fileOutputStream.write(String.format(
                        Locale.getDefault(),
                        "Hello World\ncreate by %s %s\n%s",
                        helloModel.getInfo().getUsername(),
                        helloModel.getInfo().getEmail(),
                        otherString.toString()).getBytes()
);
```

### 懒加载属性

在上面的例子中，我们都是通过使用原始数据类型去映射在构建脚本中的配置，这种方式会在配置完成之后直接映射，Gradle 4.0 提供了 `Provider` 和 `Property` 来提供懒加载属性，其中 `Provider` 表示不可变属性，`Property` 表示可变属性。这个 API 提供了两个好处：

1. 可以放心地连接不同的数据模型，而不用担心某个数据模型的值是否已经知道。举个例子，比如你想将扩展中的属性映射到 task 中的属性，但扩展的属性值只有在构建脚本配置后才能知道。
2. 避免在构建阶段进行资源密集型工作，比如当属性值时从文件中解析时

使用也比较简单，首先我们先声明一个`Property<String>` 然后使用 `ObjectFactor` 去创建：

``` java
private final Property<String> testProperty;

testProperty = project.getObjects().property(String.class);
```

> Gradle 会自动为 `Property` 生成 setter 方法，同时允许你使用 "=" 操作符来设置属性值

## 总结

在这篇文章中，我们主要讲的是，在自定义 Plugin 中经常用的到扩展功能，下一篇文章中，我们就开始进入 Plugin 的编写。