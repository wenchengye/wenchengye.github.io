---
title: 使用 Task - 编写 Gradle 脚本 (Part2)
subtitle: Gradle Manual 读书笔记 （2.2）
catalog: true
header-img: /img/header_img/tools-note-header.jpg
tags:
  - Gradle
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

## 依赖管理


    
