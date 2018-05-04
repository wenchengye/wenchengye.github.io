---
title: 编写Gradle脚本Part1
subtitle: Gradle Manual读书笔记2.1
catalog: true
header-img: /img/header_img/tools-note-header.jpg
tags:
  - Grale
categories:
  - 读书笔记
---

## 编写Gradle脚本基础

* Gradle脚本是基于Groovy的DSL(domain specific language),Gradle脚本的默认编码是utf-8。
* Gradle为每个工程创建了一个Project对象，并将这个Project对象delegate到Gradle脚本；所以未在脚本中定义的*方法*和*变量*都回去Project空间中去查找。
* 在Gradle脚本中声明变量有两种方式：局部变量和Ext属性(extra properties)。
    - 局部变量就是单纯的Groovy变量，其可见范围也符合Groovy语法规则。
    - 大部分Gradle domain model都支持extra properties；extra properties通过model的ext属性来添加和访问。
        + 当extra property被添加到对象中后，可以使用像对象属性一样的方式来访问(例如{object referrence}.{property name})。
        + Ext的功能是基于ExtraPropertiesExtension，ExtraPropertiesExtension实际上的接口像个Map；但是基于Groovy酷炫的语法功能，Ext可以像正常属性一样访问，所以尽量使用访问属性语法(而不是类Map函数调用)来访问Ext属性。
        + project的Ext在它的subprojects中依然可见。
* 当访问对象的属性时、无论是正常属性还是通过Ext添加的属性、如果该属性不存在那么Gradle会抛出异常。
* Project提供了`configure(Object o, Closure action)`接口，来用更容易阅读的方式来修改对象。
* Project提供了`apply`接口，来用其他Gradle脚本来修改对象(最常见的就是修改Project对象)，代码示例：

```groovy
task configure {
    doLast {
        def pos = new java.text.FieldPosition(10)
        // Apply the script
        apply from: 'other.gradle', to: pos
        println pos.beginIndex
        println pos.endIndex
    }
}
```
other.gradle
```groovy
// Set properties.
beginIndex = 1
endIndex = 5
```

* 一些Groovy的酷炫语法
    - 可遍历的容器加入了`each(Closure action)`函数，例如`configurations.runtime.each { File f -> println f }`。
    - 属性访问会自动调用get和set方法。
    - 函数调用不用加括号。
    - list和map可以使用字面值：
        + list字面值：`['object1', 'object2']`。
        + map字面值：`[key1:'value1', key2: 'value2']`
    - 每个Closure可以delegate一个对象，以便在Closure可以直接使用对象的属性和方法。
* 为了方便，Gradle在每个脚本中做了一些默认import。

## 文件处理

* `Project.file(Object)`方法可以用来创建File对象，参数可以时相对路径、绝对路径或者File对象；`Project.file(Object)`方法的相对路径参照永远是project目录，而不是当前的working dir。
* 使用Copy类型的task来拷贝文件，`Copy.from(Object...)`接受`Project.files(Object...)`形式参数，`Copy.into(Object)`接受`Project.file(Object)`形式参数。
* 也可以使用`Project.copy(Closure closure)`方便来拷贝文件，但是Copy task可以提供dependsOn以及增量执行等便利。
* Sync类型task继承自Copy，拷贝后会删除into目录下的其他文件(不是拷贝过来的)。
* Gradle提供了Zip、Tar、Jar、War、Ear等一系列task类型来生成压缩文件；使用技巧dig源码为好。

### FileCollection

Gradle提供了FileCollection接口来描述文件的List；FileCollection继承了Iterable<File>接口。

* FileCollection可以通过`Project.files(Object...)`方法来创建；这个方法和`Project.file(Object)`一样是project的目录。
* FileCollection支持`+`(并集)和`-`(差集)操作符。
* `Project.files(Object...)`接受很多类型的参数：
    - 相对路径&绝对路径&文件对象，以及它们的collection对象。
    - FileCollection对象。
    - Task对象，使用task的output中的文件。
    - TaskOutputs对象，使用其中的文件。
    - 返回以上参数的Closure，在每次获得FileCollection内容时运行closure，获得了延迟性和可变性。

### FileTree

Gradle提供了FileTree用来描述某个目录的整个层级结构；FileTree继承了FileCollection接口。

* FileTree可以通过`Project.fileTree(java.util.Map)`方法来创建。
* `Project.fileTree(java.util.Map)`返回的`ConfigurableFileTree`继承了`PatternFilterable`，支持`include(String...)`和`exclude(String...)`方法，例如`tree.exclude '**/Abstract*'`。
* 另外`Project.fileTree(java.util.Map)`的FileTree对象应用了一些默认exclude，详见[默认exclude列表](http://ant.apache.org/manual/dirtasks.html#defaultexcludes)。
* `Project.zipTree(java.lang.Object)`和`Project.tarTree(java.lang.Object)`方法可以返回表示zip文件、tar文件的FileTree对象。

## Log

* Gradle的log从高到低分为6个等级：
    - Error。
    - Quiet，筛选运行参数-q。
    - Warning，筛选运行参数-w。
    - Lifecycle，运行Gradle时的默认筛选等级。
    - Info，筛选运行参数-i。
    - Debug，筛选运行参数-d。
* 通过运行参数-s可以打印Gradle运行的stacktrace。
* 标准输出默认被视为是Quiet等级的log；可以通过Project.logger属性&Scrpit.logger属性(Logger类型的对象)，来输出特定等级的log。
* 通过Project.logging属性&Script.logging属性，可以获得一个LoggingManager对象；使用LoggingManager可以修改标准输出&错误输出对应的log等级(默认标准输出为Quiet,错误输出为Error)。
* 通过Gradle.useLogger(Object)方法可以替换Gradle默认的输出log行为(这个方法具体都能干什么还是看源码为好)。



