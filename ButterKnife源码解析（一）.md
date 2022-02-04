---
title: ButterKnife源码解析（一）
date: 2022-02-03 19:26:10
tags:
---




#         ButterKnife 8.4.0 源码分析（一）



## 前言

本文是根据ButterKnife的历史版本 8.4.0进行分析的。

ButterKnife 用到了编译时技术(APT Annotation Processing Tool)，再讲解源码之前我们先看看这部分内容。

## 编译时技术(APT技术)

讲解编译时技术前，我们需要先了解下代码的生命周期。

![](BK源码生命周期.png)
如图所示，代码的生命周期分为源码期、编译期、运行期。

- 源码期 当我们在写以.java结尾的文件的时期。
- 编译期 则指的是是指把源码交给编译器编译成计算机可以执行的文件的过程。在Java中也就是把Java代码编成class文件的过程.。
- 运行期 则指的是java代码的运行过程。如我们的app运行在手机中的时期。

APT，就是Annotation Processing Tool 的简称，就是可以在代码编译期间对注解进行处理，并且生成java文件，减少手动的代码输入。比如在ButterKnife中，我们通过自定义注解处理器来对@BindView注解以及注解的的元素进行处理，最终生成XXXActivity$$ViewBinder.class文件，来加少我们使用者findViewById等手动代码的输入。

## Java注解处理器

注解处理器（Annotation Processor）是**javac**的一个工具，它用来在编译时扫描和处理注解（Annotation）。你可以对自定义注解，并注册相应的注解处理器。一个注解的注解处理器，以Java代码（或者编译过的字节码）作为输入，生成文件（通常是`.java`文件）作为输出。这具体的含义什么呢？你可以生成Java代码！这些生成的Java代码是在生成的.java文件中，但你不能修改已经存在的Java类，例如向已有的类中添加方法。这些生成的Java文件，会同其他普通的手动编写的Java源代码一样被**javac**编译。

我们首先看一下处理器的API。每一个处理器都是继承于`AbstractProcessor`，如下所示：

```java
public class MyProcessor extends AbstractProcessor {
    private Filer mFiler; //文件相关的辅助类
    private Elements mElementUtils; //元素相关的辅助类
    private Messager mMessager; //日志相关的辅助类

    //用来指定你使用的 java 版本。通常你应该返回 SourceVersion.latestSupported()
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    //会被处理器调用，可以在这里获取Filer，Elements，Messager等辅助类
    @Override
    public synchronized void init(ProcessingEnvironment env) {
        super.init(env);
        elementUtils = env.getElementUtils();
        typeUtils = env.getTypeUtils();
        filer = env.getFiler();
    }


    //这个方法返回stirng类型的set集合，集合里包含了你需要处理的注解
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotataions = new LinkedHashSet<String>();
        annotataions.add("com.example.MyAnnotation");
        return annotataions;
    }


   //核心方法，这个一般的流程就是先扫描查找注解，再生成 java 文件
   //这2个步骤设计的知识点细节很多。

    @Override
    public boolean process(Set<? extends TypeElement> annoations,
            RoundEnvironment env) {
        return false;
    }
}
```

### 注册你的处理器

要像jvm调用你写的处理器，你必须先注册，让他知道。怎么让它知道呢，其实很简单，google 为我们提供了一个库，简单的一个注解就可以。 

首先是依赖

```java
compile 'com.google.auto.service:auto-service:1.0-rc2'
```



```
@AutoService(Processor.class)
public class BindViewProcessor extends AbstractProcessor {
  //...省略非关键代码
｝
```

接着只需要在你的注解处理器上加上@AutoService(Processor.class)即可

### 基本概念

- Elements：一个用来处理Element的工具类

  我们看一下Element这个类。在注解处理器中，我们扫描 java 源文件，源代码中的每一部分都是Element的一个特定类型。换句话说：Element代表程序中的元素，比如说 包，类，方法。每一个元素代表一个静态的，语言级别的结构. 

  比如：

  ```java
  package com.example;    // PackageElement 包元素
  
  public class Foo {        // TypeElement 类元素
  
      private int a;      // VariableElement 变量元素
      private Foo other;  // VariableElement 变量元素
  
      public Foo () {}    // ExecuteableElement 方法元素
  
      public void setA (  // ExecuteableElement 方法元素
                       int newA   // VariableElement 变量元素
                       ) {}
  }
  ```

  一句话概括：Element代表源代码，TypeElement代表源代码中的元素类型，例如类。然后，TypeElement并不包含类的相关信息。你可以从TypeElement获取类的名称，但你不能获取类的信息，比如说父类。这些信息可以通过TypeMirror获取。你可以通过调用element.asType()来获取一个Element的TypeMirror。

- Types：一个用来处理TypeMirror的工具类

- Filer：你可以使用这个类来创建.java文件

- Messager : 日志相关的辅助类

到此为止，一个基本的注解处理器就写好了。

下一章节我们再看看ButterKnife的内部实现吧。