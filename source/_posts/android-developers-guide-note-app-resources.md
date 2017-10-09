---
title: Android Developer Guide中的Resource
subtitle: Android官方guide随笔 - Resource
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
date: 2017-09-04 16:27:21
---


## 展开点 
* [New Tools For Managing Screen Sizes](https://android-developers.googleblog.com/2011/07/new-tools-for-managing-screen-sizes.html)。
* [Holo Everywhere](https://android-developers.googleblog.com/2012/01/holo-everywhere.html)。
* [New Mode for Apps on Large Screens](https://android-developers.googleblog.com/2011/07/new-mode-for-apps-on-large-screens.html)。  
* Fragment.setRetainInstance()的原理。
* [Localization Checklist](https://developer.android.google.cn/guide/topics/resources/localization.html#strategies)。
* [App Translation Service](https://support.google.com/l10n/answer/6359997)。
* ICU4J。
* 在xml中`@`引用和`?`引用。
* library工程的R类是否与app的R类进行合并；在xml中资源的包名，例如@android:;library工程是否有单独的包名。

## Caution   

* 直接把resource文件放置在res/目录下，会导致编译错误。
* configuration后缀有固定的顺序，其次序不能随意安排。
* aapt负责生成R类。
* 在api13之后，orientation变化同时也会引发可用宽高变化；所以要自行处理orientation变化需要声明`android:configChanges="orientation|screenSize"`。

## Resources Overview

所有的资源文件存放在`res/`目录下，并通过类型和configuration后缀安置在不同的子目录中。Android将资源和代码分离，除了保持整洁的结构之外，还便于随着configuration的变化在运行来变更所使用的资源。

## Providing Resources

### 资源类型和组织方式

每一个资源文件(或资源item)都应该被放置在`res/`目录下的某个子目录里(对于item则是存在某个子目录的某个文件里)。不考虑config后缀，`res/`下的子目录是按照资源类型区分命名的。

#### 子目录一览

Resource子目录类型表:

|子目录名 | 用途            | 
|:--------|:----------------| 
|`animator/` | 属性动画的xml文件。 |
|`anim/` | view动画的xml文件。 |
|`color/` | color state list的xml文件。 |
|`drawable/` | 包含bitmap文件(.png、.g.png、.jgp、.gif等)；包含用xml描述的drawable文件(stat list、shape、帧动画等)。 |
|`mipmap/` | 应用图标。 |
|`layout/` | 布局xml。 |
|`menu/` | menu xml。 |
|`raw/` | 不会被解释为任何类型的资源，单纯视为文件 |
|`values/` | 存放包含各种value的xml文件。 |
|`xml/` | xml格式的配置文件，这些文件通常用于某些Android自带的framework中，如widget framework。 |
|`font/` | 包含字体文件(.ttf、.otf、.ttc)；包含xml描述的字体。|

#### `values/`

这些子目录中`values/`目录比较特殊，也容易让初学者困惑。在其他目录中，单个文件就代表单个资源；在`values/`目录中，单个xml文件往往包含了一组资源，在xml中的单个特定标签代表单个资源。
事实上`values/`下的xml文件是可以随意命名的，标签名才是资源类别的标志；同时不同种类的资源可以写在同一个xml文件中。但是为了清晰，我没习惯给`values/`下的xml文件合理命名，并按类型将资源存放与不同的xml文件中。

### 提供动态可选资源

#### 子目录及命名

使用动态可选资源，需要借助`res/`下的子目录。首先建立名称格式为`<resource_name>[-<config_qualifier>...]`格式的子目录，然后把可选资源以完全相同的名称命名并放置在不同后缀的子目录下。

注意，子目录的`<config_qualifier>`后缀可以有多个，后缀次序有严格定义(在后面表中的次序)，如果不安次序命名这个子目录会被系统忽略。

#### 后缀一览表

Android支持的config后缀表如下，表中的顺序和后缀出现在文件名中的顺序相同:

|后缀类型 |后缀示例 | 解释 |
|:--------|:------|:-----|
|MCC & MNC| mcc310、mcc310-mnc004 | Mobile Country Code(MCC) & Mobile Network Code(MNC)，从SIM卡中读出。|
|语言和地区 | en、en-rUs | 当前机器使用的语言(和地区)，格式为`<语言code>[-r<地区code>]` |
|布局方向 | ldrtl、ldltr | 布局是从右向左(ldrtl)还是从左向右(ldltr，这是默认行为)。|
|最小宽度 |sw<N>dp、sw600dp | 实际上是指屏幕宽度和高度中最小的dp数(并不仅是指宽度，所以也不随orientation变化)。 |
|可用宽度 | w<N>dp、w720dp | 当前可用的屏幕宽度，随orientation变化。 |
|可用高度 | h<N>dp、h720dp | 当前可用的屏幕高度，随orientation变化。 |
|屏幕尺寸 | small、normal、large、xlarge | 
|是否长屏幕 | long & notlong | |
|是否圆屏幕 | round & notround| |
|orientation | port & land | |
|UI mode | {car, desk, television, appliance, watch, vrheadset} |
|是否夜间模式 | night & notnight | |
|屏幕像素密度 | ldpi、mdpi | |
|触屏类型 |notouch & finger | | 
|是否有键盘 | keysexposed & keyshidden & keyssoft | |
|键盘输入方式 | nokeys & qwerty & 12key | |
|系统版本 | v3、v11、v17 | |

使用从右向左布局，需要把`<application>`标签的android:supportsRtl属性设置为true并且targetSdkVersion大于17。
对于不用屏幕尺寸的资源(small、xlarge)，如果没有完全匹配的资源系统会选相最匹配的资源；但是系统不会选择更大尺寸的资源，所以如果所有可选尺寸都大于当前尺寸，运行时会报错。

使用新版本系统才添加后缀，相当于自动为子目录添加了系统版本后缀，老版本系统会自动忽略这个子目录。例如可用宽度后缀来自api13，所以添加w600dp后缀的功效等于w600dp-v13。

#### 可选资源子目录命名规则   

* 同一个子目录可以包含多个后缀，用`-`符号分隔。
* 后缀必须按照*后缀一览表*中的次序排列。
* 子目录都需要直接存放在`res/`目录下。
* 子目录名不区分大小写，在编译时编译器会将子目录名统一变为小写。
* 同一个类后缀在一个子目录中只能出现一次。

#### 使用资源引用  

为了防止将重复资源拷贝到不同的子目录下(比如在rES、rFR用图片A，rCA用图片B)，可以使用资源引用来解决；单并非所用类型的资源都支持资源引用。

drawble资源以及string、color等value资源可以通过在`values／`目录下新建item来建立引用；layout资源可以通过`<merge>`标签和`<include>`标签来建立引用。

### 可选资源最佳实践

永远提供default资源(屏幕像素密度后缀是个例外)；低版本系统无法识别高版本加入的后缀，找不到匹配的资源会抛出异常。

### Android系统的资源匹配规则 

1. 剔除所有后缀和当前configuration有冲突的子目录(除了屏幕像素密度后缀)。
2. 在*后缀一览表*中，按优先级从高到低的顺序选中一类后缀。
3. 如果有子目录包含这类后缀，执行步骤4；否则返回步骤2。
4. 剔除所有不包含选中后缀的子目录(针对屏幕像素密度后缀则是选择最接近的)。
5. 如果只剩下一个子目录，则匹配资源确定；否则返回步骤2。


## Accessing Resources

aapt脚本会根据资源生成R类，每个资源类型对应一个子类，没个资源对应一个静态整数常量。 

## Handling Configuration Changes

当Configuration变化时，Activity会被销毁重建，以便根据新的Configuration加载可选资源。在SaveInstanceState以外，本章将介绍其他两个应对Configuration变化的技巧。
即便自行处理Configuration变化，也不意味着可以避免处理SaveInstanceState，因为还有进程回收这个问题。

### 在Configuration变化时保留对象

当重启Activity加载资源量较大，尤其涉及网络请求时，可以利用一个*被保留的*Fragment来协助保存数据。
创建一个不包含layout的Fragment(add by tag)，用它来保存状态数据，并对调用`Fragment. setRetainInstance(true)`。
注意不要让被保留的Fragment持有Activity的引用，否则在Activity销毁重建时会发生内存泄漏。

### 自行处理Configuration变化

* 在`<activity>`标签的android:configChanges声明想要自行处理的Configuration变化。
* 当被声明的Configuration发生变化时，Activity的onConfigurationChanged()被调用，当前的Configuration被作为参数传入。

## Localizing with Resources

使用`<xliff:g>`标签来标记string资源中不希望被翻译的部分：

>`<string name="countdown">`   
>&emsp;&emsp;`<xliff:g id="time" example="5 days>%1$s</xliff:g>until holiday`   
>`</string>`

使用App Translation Service来翻译string资源，在Android Studio中就有入口。

//TODO: in detail

### 测试本地化

使用adb更改语言 adb shell -> setprop persist.sys.locale [BCP-47 language tag;e.g. fr-CA];stop;sleep 5;start。

测试default资源的方式：将系统设置为你应用所不支持的configuration。

## Inline Complex XML Resources

有一些复杂的资源是需要依赖多个其他资源的，在这种情况下，就会产生很多资源文件。如果被依赖的资源不会用在别处，可以借助inline资源(`<aapt:attr>`标签)将多个资源放在同一文件中。
`<aapt:attr>`标签所包裹的区域就是一个inline的资源xml，在编译时会被生成为单独的资源文件；这个inline的资源会被用作`<aapt:attr>`标签父标签的属性，属性名由`<aapt:attr>`标签的name属性制定。

例如：
>`<target android:name="rotationGroup">`   
>&emsp;&emsp;`<aapt:attr name="android:animation" >`    
>&emsp;&emsp;&emsp;&emsp;`<objectAnimator`   
>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;`android:duration="6000"`   
>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;`android:propertyName="rotation"`   
>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;`android:valueFrom="0"`   
>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;`android:valueTo="360" />`   
>&emsp;&emsp;`</aapt:attr>`   
>`</target>`   







