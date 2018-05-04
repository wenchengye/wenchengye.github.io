---
title: 编写Gradle脚本Part2
subtitle: Gradle Manual读书笔记2.2
catalog: true
header-img: /img/header_img/tools-note-header.jpg
tags:
  - Grale
categories:
  - 读书笔记
---

## Task(稍稍)进阶

* 在执行task时会有以下几种输出
    - EXECUTED：正常执行。
    - UP-TO-DATE：由‘增量执行’特性造成，任务的输入输出都没有变化。
    - FROM-CACHE：由‘build cache’特性造成。
    - SKIPPED：任务被忽略。
    - NO-SOURCE：任务的输入不存在。
* 创建task实际上是通过project.task()方法和project.tasks.create()方法来进行的，project.tasks的类型是TaskContainer；创建task时task名称的字符串可以带引号也可以不带(groovy语法)。
* task被创建之后会作为project的属性，可以使用project.{taskName}的方式直接访问；也可以通过tproject.tasks(TaskContainer)来访问。
* 通过project.tasks.getByPath(String path)方法来访问其他工程的task；其中path的格式为`:{projectName}:{taskName}`。
* task对其他task的依赖，是通过`Task.dependsOn(Object... paths)`方法来设置的，可以依赖当前project的task或者其他project的task；设置task的依赖参数有多种形式：
    - 当前project的其他task的名字字符串。
    - 其他project的task的路径字符串(`:{projectName}:{taskName}`)。
    - 所依赖的task对象。
    - 返回task对象活task collection的closure，这个closure在evaluated之后才会执行。
* 可以通过task的description属性给task添加描述。
* 在创建任务时，通过将overwrite设置为true可以覆盖已有的同名任务；否则创建和已用任务同名任务时，会抛出异常。代码示例：
```groovy
task copy(type: Copy)

task copy(overwrite: true) {
    ...
}
```

### Task顺序

task顺序并不规定task之间的依赖关系，而是当一些task在一次gradle run中都要执行的时候，约定task之间的执行顺序。

* task顺序目前包括mustRunAfter和shouldRunAfter两种规则；其区别就是在shouldRunAfter规则在造成环 & 并行执行模式下，会被忽略。
* task循序通过`Task.mustRunAfter(Object...)`和`Task.shouldRunAfter(Object...)`两个方法设置；他们接受的参数形式和`Task.dependsOn(Object... paths)`一样。

### 忽略任务

* 通过`Task.onlyIf(Closure)`方法忽略任务，当Closure返回false时task会被忽略；Closure的执行时间是在task执行之前。
* 在task执行代码中抛出StopExecutionException；抛出StopExecutionException后当前task的后续代码会被忽略，**并执行下一个task**。
* 将task.enabled属性设置为false。

### Finalizer tasks

* 可以通过`Task.finalizedBy(Object...)`方法来定义类似finally语意的task；`Task.finalizedBy(Object...)`接受的参数形式和`Task.dependsOn(Object... paths)`一样。
* 当task A准备执行之前，A的Finalizer tasks会被加入TaskGraph。
* 即使task执行失败，其Finalizer tasks也会被执行；但是如果task没有被执行(例如upToDate)，那么Finalizer tasks不会被执行。
* 和代码中的finally语义相同，Finalizer tasks适合做一些必须被执行的清理工作。


## Task的增量执行

在执行task时，如果task的input和output都没有变化，那么task被认为是UP-TO-DATE的并不被执行；会进行增量执行的最低条件时，task至少有一个output。

### 使任务支持增量执行

* 使任务支持增量执行，需要做的就是定义一些getter方法并对这些getter方法进行(input & output)注解。
* (input & output)相关的注解分为三类：
    - 简单类型，包括任何实现了Serializable接口的类型；仅适用于输入@Input注解。
    - 文件类型，包括File、 FileCollection以及任何`Project.file(java.lang.Object)`和`Project.files(java.lang.Object[])`方法的合法输入类型；文件类型在输入输出注解中使用最为广泛，包括@InputFile、@InputFile、@InputFiles、@OutputFile、@OutputDirectory、@OutputDirectory等。
    - Nested类型，包含 (input & output)注解的自定义类型；仅适用于@Nested注解。
* 由于注解会被继承(superclass或interface)，所以子类的(input & output)注解会覆盖父类的；同时superclass中的(input & output)注解优先级比interface中的优先级高。
* 在output目录中的文件增加，依然会被认定为upToDate；`TaskOutputs.upToDateWhen(Closure)`允许自定义一些upToDate检查条件。


### 通过API修改输入输出

当不能修改task类的源码时，也就不能使用注解定义输入输出；所以Gradle还提供了一组修改输入输出的API。

* 通过Task.inputs(TaskInputs)和Task.outputs(TaskOutputs)来修改任务的输入输出；TaskInputs提供了property()、file()、dir()等方法，TaskOutputs提供了file()、dir()等方法。

### 定义输入输出的其他影响

* 如果任务A的某个输入被赋值为任务B的某个输出，那么任务A自动dependsOn任务B。
* 被(input & output)注解的属性，会检查其值的合法性，例如文件&目录是否存在。

## Gradle的生命周期

Gradle的核心是建立起task的有向无环图，并按照顺序执行这些task；所以gradle脚本的核心就是定义和配置任务。

* 一次Gradle执行可以被分为三个阶段
    - 初始化阶段，确定Project结构并为每个Project生成Project对象(settings.gradle)。
    - 配置阶段，对所有参与这次Gradle执行Project进行配置(build.gradle)。
    - 执行阶段，确定taskGraph，并依次执行task。

### 初始化阶段

初始化阶段Gradle会寻找setting.gradle文件并创建Project对象(多个)。

* 寻找setting.gradle是为了确定当次执行是多工程还是单工程，寻找规则如下：
    - 在相邻目录中的名为'master'的目录中寻找(flat结构，所以flat结构的rootProject目录最好命名为'master')。
    - 在父目录中寻找。
    - 如果没有找到，或者执行目录没有在setting.gradle中定义，当次执行为单工程；否则为多工程。
* 可以在命令行中加入`-u`参数，强制当次执行为单工程；但是如果执行目录中存在setting.gradle，`-u`参数被忽略。

### 监听脚本执行

* `Project.beforeEvaluate(Closure)`、`Project.afterEvaluate(Closure)`、`Gradle.beforeProject(Closure)`、`Gradle.afterProject(Closure)`等方法可以用于监听Project的evaluate过程。
* 通过`Project.tasks`和`Gradle.taskGraph`的监听方法，可以监听task的创建过程和执行过程。

### settings.gradle和多工程结构

* "Settings文件"用于定义多工程结构，其默认命名是settings.gradle；多工程构建时，root project目录下必须存在settings.gradle；正如build.gradle会delegate一个Project对象，settings.gradle会delegate一个Settings对象。
* 多工程结构被定一个为一个单根树形结构，根节点被称为rootProject；默认配置下，rootProject物理路径就是setting.gradle文件所在的目录，subProject物理路径结构的和Project树结构相同；这种默认配置可以在setting.gradle中修改。
* `Settings.getRootProject()`和`Settings.project(String)`方法会返回ProjectDescriptor对象，可以通过ProjectDescriptor对象来修改Project的物理路径、build脚本名字等。

## 依赖管理


    
