---
title: Gradle 脚本基础 - 编写 Gradle 脚本 (Part1)
subtitle: Gradle Manual 读书笔记 （2.1）
catalog: true
header-img: /img/header_img/tools-note-header.jpg
tags:
  - Gradle
categories:
  - 读书笔记
---

## Gradle 脚本基础 

### Gradle API
* Gradle 脚本是基于 Groovy (或者 kotlin) 的 DSL (domain specific language), Gradle 脚本的默认编码是 utf-8。
* Gradle 为每个工程创建了一个 `Project` 对象，并将这个 `Project` 对象 delegate 到 Gradle 脚本；所以未在脚本中定义的*方法*和*变量*都会去 `Project` 的命名空间中去查找，所以 `Project` 类提供了最基础的 Gradle 脚本 API。
* 大部分 Gradle 脚本都会 delegate 到 `Project` 对象，但是也有例外。例如：settings.gradle 被 delegate 到 `Settings` 对象，init.gradle 被 delegate 到 `Gradle` 对象。
* `Project` 提供了 `configure(Object o, Closure action)`接口，来用更容易阅读的方式来修改对象。
* `Project` 提供了 `apply` 接口，来用其他 Gradle 脚本来修改对象 (最常见的就是修改 `Project` 对象)，代码示例：

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

* 为了方便，Gradle 在每个脚本中做了一些默认 import。

### 在 Gradle 脚本中使用变量   
在 Gradle 脚本中声明变量有两种方式：局部变量和 Ext 属性 (extra properties)。

* 局部变量就是单纯的 Groovy 变量，其可见范围也符合 Groovy 语法规则。
* 大部分 Gradle domain model 都支持 extra properties；extra properties 通过 model 的 ext 属性来添加和访问。
    - 当 extra property 被添加到对象中后，可以使用像对象属性一样的方式来访问(例如 {object referrence}.{property name})。
    - Ext 的功能是基于 ExtraPropertiesExtension，ExtraPropertiesExtension 实际上的接口像个 Map；但是基于Groovy酷炫的语法功能，Ext 可以像正常属性一样访问，所以尽量使用访问属性语法(而不是类 Map 函数调用)来访问 Ext 属性。
    - project 的 Ext在它的 subprojects 中依然可见。
* 当访问对象的属性时、无论是正常属性还是通过Ext添加的属性、如果该属性不存在那么 Gradle 会抛出异常。

### 在 Gradle 脚本中使用第三方库
* 如果想在 gradle 脚本执行时使用一些第三方库，可以使用 `buildscript { }` 代码块来引入第三方库。
* 将第三方库依赖添加到名为 "classpath" 的 configuration 上之后，第三方库中的 class 就可以在 gradle 脚本中被引用到。 
* 多工程环境下，某工程通过 `buildscript { }` 引入的第三方库在其子工程中也能使用。
* 每个 project 都会有名为 "buildEnvironment" 的任务，改任务会打印当前 project 的 gradle 脚本的所有第三方依赖 (和打印 java 工程依赖形式相同)。
* `buildscript { }` 的 Closure 被代理到 `ScriptHandler` 类对象上，更多使用方法可以参照其 api。

### 一些 Groovy 语法特性   
* 可遍历的容器加入了`each(Closure action)`函数，例如`configurations.runtime.each { File f -> println f }`。
* 属性访问会自动调用get和set方法。
* 函数调用不用加括号。
* list和map可以使用字面值：
    - list字面值：`['object1', 'object2']`。
    - map字面值：`[key1:'value1', key2: 'value2']`
* 每个 Closure 可以 delegate 一个对象，以便在 Closure 可以直接使用对象的属性和方法。

## Gradle 的生命周期

Gradle 的核心是建立起 task 的有向无环图，并按照顺序执行这些 task ；所以 Gradle 脚本的主要工作就是定义和配置 task。

一次 Gradle 执行可以被分为三个阶段

* 初始化阶段，确定 Project 结构并为每个 Project 生成 `Project` 对象(settings.gradle)。
* 配置阶段，对所有参与这次 Gradle 执行 Project 进行配置(build.gradle)。
* 执行阶段，确定 taskGraph，并依次执行 task。

### 定位 setting.gradle

初始化阶段 Gradle 会寻找 setting.gradle 脚本，在执行该脚本后创建 Project 对象(多个)。

* 寻找 setting.gradle 是为了确定当次执行是多工程还是单工程，寻找规则如下：
    - 在相邻目录中的名为 'master' 的目录中寻找(flat 结构，所以 flat 结构的 rootProject 目录需要被命名为 'master')。
    - 在父目录中寻找。
    - 如果没有找到，或者执行目录没有在 setting.gradle 中定义，当次执行为单工程；否则为多工程。
* 可以在命令行中加入 `-u` 参数，强制当次执行为单工程；但是如果执行目录中存在 setting.gradle，`-u` 参数被忽略。

### settings.gradle 和多工程结构

* "Settings文件"用于定义多工程结构，其默认命名是 settings.gradle；多工程构建时，root project 目录下必须存在 settings.gradle；正如 build.gradle 会 delegate 一个 `Project` 对象，settings.gradle 会 delegate 一个 `Settings` 对象。
* 多工程结构被定为单根树形结构，根节点被称为 rootProject；默认配置下，rootProject 物理路径就是 setting.gradle 文件所在的目录，subProject 物理路径结构的和 Project 树结构相同；这种默认配置可以在 setting.gradle 中修改。
* `Settings.getRootProject()` 和 `Settings.project(String)` 方法会返回 `ProjectDescriptor` 对象，可以通过 `ProjectDescriptor` 对象来修改 Project 的物理路径、build 脚本名字等。

### 监听生命周期

* `Project.beforeEvaluate(Closure)`、`Project.afterEvaluate(Closure)`、`Gradle.beforeProject(Closure)`、`Gradle.afterProject(Closure)`等方法可以用于监听 Project 的 evaluate 过程。
* 通过 `Project.tasks` 和 `Gradle.taskGraph` 的监听方法，可以监听 task 的创建过程和执行过程。

## 多工程构建

多工程构建是 Gradle 的优势之一，Gradle 的多工程结构以单根树形的方式组织。 


## 文件处理

### `Project.file()`

* `Project.file(Object)`方法可以用来创建 `File` 对象，参数可以是相对路径、绝对路径、 `File` 对象、`Path` 对象。
* `Project.file(Object)` 方法的相对路径参照永远是 project 目录，而不是当前的 working dir。相对的 `new File(relative path)` 参照是 working dir，所以使用 `Project.file(Object)` 方法会获得更好的一致性，文件路径描述不会随命令行路径变化。

### `FileCollection`

Gradle 提供了 `FileCollection` 接口来描述文件的集合；`FileCollection` 继承了 `Iterable<File>` 接口。

* `FileCollection` 可以通过 `Project.files(Object...)` 方法来创建；这个方法和`Project.file(Object)` 一样是以 project 的目录为参照目录。
* `FileCollection`支持 `+` (并集) 和 `-` (差集) 操作符。
* `Project.files(Object...)`接受很多类型的参数：
    - 相对路径&绝对路径&文件对象，以及它们的 Collection 对象。
    - `FileCollection`对象。
    - `Task`对象，使用task的output中的文件。
    - `TaskOutputs`对象，使用其中的文件。
    - 返回以上参数的 `Closure`，在每次获得 `FileCollection` 内容时运行 `Closure`，获得了延迟性和可变性。

### `FileTree`

Gradle 提供了 `FileTree` 用来描述某个目录的整个层级结构；`FileTree` 继承了 `FileCollection` 接口。
 
* FileTree可以通过 `Project.fileTree(java.util.Map)` 方法来创建。
* 在作为 `Copy` 的参数时，`FileCollection` 不会保留目录层级结构，造成"平铺"的效果，而 `FileTree` 则会保留目录层级结构。
* `Project.fileTree(java.util.Map)` 返回的 `ConfigurableFileTree` 继承了 `PatternFilterable`，支持 `include(String...)` 和 `exclude(String...)` 方法，例如 `tree.exclude '**/Abstract*'`。
* 另外 `Project.fileTree(java.util.Map)` 的 `FileTree` 对象应用了一些默认exclude，详见[默认exclude列表](http://ant.apache.org/manual/dirtasks.html#defaultexcludes)。
* `Project.zipTree(java.lang.Object)` 和 `Project.tarTree(java.lang.Object)` 方法可以返回表示 zip 文件、tar 文件的 `FileTree` 对象。

### Util Tasks

* 使用 `Copy` 类型的task来拷贝文件，`Copy.from(Object...)` 接受 `Project.files(Object...)` 形式参数，`Copy.into(Object)` 接受 `Project.file(Object)` 形式参数。
* 也可以使用 `Project.copy(Closure closure)` 方便来拷贝文件，但是 `Copy` task 可以提供dependsOn以及增量执行等便利。
* `Sync` 类型 task 继 承自`Copy`，拷贝后会删除 "into" 目录下的其他文件(不是拷贝过来的)。
* Gradle 提供了`Zip`、`Tar`、`Jar`、`War`、`Ear`等一系列task类型来生成压缩文件；使用技巧dig源码为好。

## 使用 Log

* Gradle 的 log 从高到低分为6个等级：
    - Error。
    - Quiet，筛选运行参数 -q。
    - Warning，筛选运行参数 -w。
    - Lifecycle，运行 Gradle 时的默认筛选等级。
    - Info，筛选运行参数 -i。
    - Debug，筛选运行参数 -d。
* 通过运行参数 -s 可以打印 Gradle 运行的 stacktrace。
* 标准输出默认被视为是 Quiet 等级的 log；可以通过 `Project.logger` 属性或 `Scrpit.logger` 属性(`Logger`类型的对象)，来输出特定等级的log。
* 通过 `Project.logging` 属性或 `Script.logging` 属性，可以获得一个 `LoggingManager` 对象；使用 `LoggingManager` 可以修改标准输出&错误输出对应的 log 等级(默认标准输出为 Quiet,错误输出为 Error)。
* 通过 `Gradle.useLogger(Object)` 方法可以替换 Gradle 默认的输出 log 行为(这个方法具体都能干什么还是看源码为好)。



