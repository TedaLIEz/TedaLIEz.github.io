---
title: APT插件实战
date: 2019-06-29 20:34:53
tags:
---

- [Annotation Processor](#Annotation-Processor)
  - [注解处理器的运行原理](#%E6%B3%A8%E8%A7%A3%E5%A4%84%E7%90%86%E5%99%A8%E7%9A%84%E8%BF%90%E8%A1%8C%E5%8E%9F%E7%90%86)
  - [实现一个简单的注解处理器](#%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84%E6%B3%A8%E8%A7%A3%E5%A4%84%E7%90%86%E5%99%A8)
    - [基本步骤](#%E5%9F%BA%E6%9C%AC%E6%AD%A5%E9%AA%A4)
      - [解决依赖](#%E8%A7%A3%E5%86%B3%E4%BE%9D%E8%B5%96)
      - [创建注解](#%E5%88%9B%E5%BB%BA%E6%B3%A8%E8%A7%A3)
      - [创建注解处理器](#%E5%88%9B%E5%BB%BA%E6%B3%A8%E8%A7%A3%E5%A4%84%E7%90%86%E5%99%A8)
        - [init()](#init)
        - [Element](#Element)
        - [Messager](#Messager)
        - [process()](#process)
        - [getSupportedAnnotationTypes()](#getSupportedAnnotationTypes)
        - [getSupportedSourceVersion()](#getSupportedSourceVersion)
      - [注解处理器方法实现](#%E6%B3%A8%E8%A7%A3%E5%A4%84%E7%90%86%E5%99%A8%E6%96%B9%E6%B3%95%E5%AE%9E%E7%8E%B0)
      - [向javac声明这个注解处理器](#%E5%90%91javac%E5%A3%B0%E6%98%8E%E8%BF%99%E4%B8%AA%E6%B3%A8%E8%A7%A3%E5%A4%84%E7%90%86%E5%99%A8)
    - [编译，体验成果](#%E7%BC%96%E8%AF%91%E4%BD%93%E9%AA%8C%E6%88%90%E6%9E%9C)
  - [注解处理器不能做(不建议)的事情](#%E6%B3%A8%E8%A7%A3%E5%A4%84%E7%90%86%E5%99%A8%E4%B8%8D%E8%83%BD%E5%81%9A%08%E4%B8%8D%E5%BB%BA%E8%AE%AE%E7%9A%84%E4%BA%8B%E6%83%85)
  - [注解处理器的特性](#%E6%B3%A8%E8%A7%A3%E5%A4%84%E7%90%86%E5%99%A8%E7%9A%84%E7%89%B9%E6%80%A7)

# Annotation Processor

大量框架(例如Room, Dagger2, DataBinding, Butterknife)都利用Java1.6引入的annotation processor(后文统称注解处理器)来实现自己的框架功能。

这篇文章简单介绍注解处理器

## 注解处理器的运行原理

整个代码生成的过程发生在编译期，javac会获知所有的注解处理器，同时在编译时处理所有的注解。
由于整个过程在编译期完成，你可以认为注解处理器是编译器的一部分，因此最后的编译产物是不会包含编译处理器的内容的。

## 实现一个简单的注解处理器
### 基本步骤
1. 创建一个注解module(Java Library)，这个module只包含注解(Annotation)
2. 创建一个处理器module(Java Library)，这个module包含相关注解的处理逻辑


#### 解决依赖
注解相关的module不需要任何依赖。
处理器module需要依赖我们自定义的注解和一些工具库。上文中提到过，这个module将会在编译器完成自己的任务，因此依赖此module的app不会打包这个module的内容。所以不用担心这个module的包体积问题

```groovy
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.google.guava:guava:21.0'
    implementation 'com.squareup:javapoet:1.8.0'
    implementation project(':annotation')
```

之后给我们的app添加上注解处理器的编译依赖，这里使用另外一个依赖关键字`annotationProcessor`

```groovy
    implementation project(':annotation')
    annotationProcessor project(':processor')
```

`annotationProcessor`表示我们只需要编译期有这个依赖，不需要打包进apk

#### 创建注解

这里复习一下Java中注解的相关概念

`@Target`:指明你的注解作用的对象，可以有许多类型
```java
public enum ElementType {
    TYPE, //If you want to annotate class, interface, enum..
    FIELD, //If you want to annotate field (includes enum constants)
    METHOD, //If you want to annotate method
    PARAMETER, //If you want to annotate parameter
    CONSTRUCTOR, //If you want to annotate constructor
    LOCAL_VARIABLE, //..
    ANNOTATION_TYPE, //..
    PACKAGE, //..
    TYPE_PARAMETER, //..(java 8)
    TYPE_USE; //..(java 8)

    private ElementType() {
    }
}
```

`@Retention`:指明注解如何存储，有三种存储方式
1. SOURCE---用于编译期，不会保存
2. CLASS---保存到最后的class文件，运行期不会保留
3. RUNTIME---保存到class文件，同时运行期也会保留(用于反射)


我们这个场景只需要编译期注解:

```java
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface NewIntent {
}
```

#### 创建注解处理器

在注解处理器模块添加一个新类
```java
package com.github.tedaliez.processor;

import java.util.Set;

import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.ProcessingEnvironment;
import javax.annotation.processing.RoundEnvironment;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.TypeElement;

public class NewIntentProcessor extends AbstractProcessor {

    private Types typeUtils;
    private Elements elementUtils;
    private Filer filer;
    private Messager messager;
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        typeUtils = processingEnv.getTypeUtils();
        elementUtils = processingEnv.getElementUtils();
        filer = processingEnv.getFiler();
        messager = processingEnv.getMessager();
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        return false;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return super.getSupportedAnnotationTypes();
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return super.getSupportedSourceVersion();
    }
}
```

##### init()
初始化的回调。在这个方法中我们创建了四个类型的引用:
1. Elements:Element(下文介绍)的工具类
2. Types:TypeMirror(下文介绍)的工具类
3. Filer:用来创建文件
4. Messager:用来当注解处理器出现编译错误时打印错误信息
##### Element
注解处理过程中我们会扫描我们的java源代码。每个代码都有一个确定的Element类型，换句话说，Elment代表了一个程序的语言层级的元素

```java
package com.example;	// PackageElement

public class Foo {		// TypeElement

	private int a;		// VariableElement
	private Foo other; 	// VariableElement

	public Foo () {} 	// ExecuteableElement

	public void setA ( 	// ExecuteableElement
	                 int newA	// TypeElement
	                 ) {}
}
```
但是你不能从Element中获取类的相关信息，例如类的父类关系。这类信息得从TypeMirror中获得

##### Messager
用来当注解处理器出现编译错误时打印错误信息，需要注意的是当有异常出现时，你的注解处理器也应该正常结束而非选择抛出异常。如果抛出异常结束，那么javac会打印你的堆栈，但你messager的信息则不会被打印出来


##### process()
注解处理器中最重要的方法，方法提供了所有被注解的元素。在这个函数中你可以来处理你的注解，我们在这里实现生成代码的逻辑

##### getSupportedAnnotationTypes()
返回注解处理器支持的注解类型，你可以认为这个方法的返回值是`process`方法的第一个参数值。这个方法也可以使用`@SupportedAnnotationTypes`注解来替代，二者等效

##### getSupportedSourceVersion()
指明支持代码生成的java版本。这个方法也可以使用`@SupportedSourceVersion`注解来替代，二者等效



#### 注解处理器方法实现

推荐使用JavaPoet来完成新的class文件的生成

#### 向javac声明这个注解处理器

在处理器模块代码根目录下(一般是src/main)，创建resources/META-INF/services/javax.annotation.processing.Processor文件，在这个文件中声明你的注解器类路径
```
com.github.tedaliez.processor.NewIntentProcessor
```
换行可以声明下一个注解器的路径，这里我们只有一个。


### 编译，体验成果

具体的代码可以参考https://github.com/TedaLIEz/AptExample

## 注解处理器不能做(不建议)的事情
1. 修改已有的类文件。注解处理器的初衷是希望用注解来生成一些新的类文件，修改已有的类文件可能会有办法，但属于特殊技巧
2. 受1的限制，注解处理器并不能很好地实现面向切面编程，因为我们很难侵入到一个方法执行过程中加入一段我们的代码。


## 注解处理器的特性
注解处理器有下面几个特性:
1. 提供编译期的错误检查(fail-fast).更容易暴露错误和调试，个人认为这是注解处理器解决问题的最大优势
2. 提供了类似C++的模版能力，比C++的模版有着更好的错误信息
3. 避开了运行期使用反射获取注解带来的性能问题


