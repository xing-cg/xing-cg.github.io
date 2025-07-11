---
title: Android_项目目录结构
categories:
  - - Android
tags: 
date: 2022/8/14
updated: 
comments: 
published:
---

# 项目目录结构

1. `.gradle`和`.idea`：这两个目录都是Android Studio自动生成的文件。无须关心，不要手动编辑。
2. app：项目中的代码、资源等内容几乎都是放置在这个目录下，开发工作也基本都是在这个目录下进行。
3. build：这个目录主要包含了一些在编译时自动生成的文件。
4. gradle：这个目录下包含了`gradle wrapper`的配置文件，使用gradle wrapper的方式不需要提前将gradle下载好，而是会自动根据本地的缓存情况决定是否需要联网下载gradle。Android Studio默认没有启用gradle wrapper的方式，如果需要打开，可以点击`File, Settings, Build, Execution, Deployment, Gradle`，进行配置更改。
5. `.gitignore`：这个文件用来将指定的目录或文件排除在版本控制之外。
6. `build.gradle`：项目全局的gradle构建脚本，文件内容通常不需要修改。
7. `gradle.properties`：项目全局的gradle配置文件，在这里配置的属性将会影响到项目中所有的gradle编译脚本。
8. gradlew和`gradlew.bat`：这两个文件是用来在命令行界面中执行gradle命令的，其中gradlew是在Linux或Mac系统使用，`gradlew.bat`是在Windows系统中使用。
9. `项目名.iml`：iml文件是所有IntelliJ IDEA项目都会自动生成的一个文件（Android Studio是基于IntelliJ IDEA开发的），用于标识这是一个IntelliJ IDEA项目，我们不需要修改这个文件中的任何内容。
10. `local.properties`：这个文件用于指定本机中的Android SDK路径，通常内容都是自动生成的，并不需要修改。除非本机中的Android SDK位置发生了变化，那么就将这个文件中的路径改成新的位置即可。
11. `settings.gradle`：用于指定项目中所有引入的模块。由于此时项目中只有一个app模块，因此文件中只引入了app这一个模块。通常情况下，模块的引入是自动完成的，很少去手动修改此文件。

发现，基本上除了app目录之外，大多数目录、文件都是自动生成的。下面对app内容进行更为详细的分析。

1. build：这个目录和外层的build目录类似，主要也是包含了一些在编译时自动生成的文件，不过它里面的内容更多更杂，不需要过多关心。
2. libs：如果你的项目中使用到了第三方jar包，就需要把这些jar包都放在libs目录下，放在这个目录下的jar包都会被自动添加到构建路径里去。
3. androidTest：此处是用来编写Android Test测试用例的，可以对项目进行一些自动化测试。
4. java：java目录是放置所有Java代码的地方，展开该目录，将看到刚才创建的HelloWorldActivity文件就在里面。
5. res：这个目录下的内容有点多。简单点说，就是在项目中使用到的所有图片、布局、字符串等资源都要存放在这个目录下。当然这个目录下还有很多子目录，图片放在drawable目录下，布局放在layout目录下，字符串放在values目录下，所以不用担心会把整个res目录弄得乱糟糟的。

6. `AndroidManifest.xml`：这是整个Android项目的配置文件，在程序中定义的所有四大组件都需要在这个文件里注册，另外还可以在这个文件中给应用程序添加权限声明。
7. test：此处是用来编写Unit Test测试用例的，是对项目进行自动化测试的另一种方式。
8. `.gitignore`：这个文件用于将app模块内的指定的目录或文件排除在版本控制之外，作用和外层的`.gitignore`文件类似。
9. `app.iml`：IntelliJ IDEA项目自动生成的文件，不需要关心或修改这个文件中的内容。
10. `build.gradle`：这是app模块的gradle构建脚本，这个文件中会指定很多项目构建相关的配置。
11. `proguard-rules.pro`：这个文件用于指定项目代码的混淆规则，当代码开发完成后打成安装包文件，如果不希望代码被别人破解，通常会将代码进行混淆，从而让破解者难以阅读。

# AndroidManifest.xml

```xml
<activity anroid:name=".HelloWorldActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"></action>
        <category android:name="android.intent.category.LAUNCHER"></category>
    </intent-filter>
</activity>
```

这段代码表示对HelloWorldActivity这个活动进行注册， 没有在AndroidManifest.xml里注册的活动是不能使用的。其中intent-filter里的两行代码非常重要，`<action android:name="android.intent.action.MAIN"/>`和	`<category android:name="android.intent.category.LAUNCHER"/>`表示HelloWorldActivity是这个项目的主活动，在手机上点击应用图标， 首先启动的就是这个活动。

Activity是Android应用程序的门面，凡是在应用中看得到的东西，都是放在活动中的。因此看到的界面，其实就是HelloWorldActivity这个活动。打开HelloWorldActivity，代码如下所示：

```java
public class HelloWorldActivity extends AppCompatActivity{
    @Override
    protected void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        setContentView(R.layout.hello_world_layout);
    }
}
```

可以看到，HelloWorldActivity是继承自AppCompatActivity的，这是一种向下兼容的Activity，可以将Activity在各个系统版本中增加的特性和功能最低兼容到Android2.1系统。

Activity是Android系统提供的一个活动基类，我们项目中所有的活动都必须继承它或者它的子类才能拥有活动的特性（AppCompatActivity是Activity的子类）。然后可以看到HelloWorldActivity中有一个`onCreate()`方法，这个方法是一个活动被创建时必定要执行的方法，其中只有两行代码，并且没有`Hello World!`的字样。那么显示的`Hello World!`是在哪里定义的呢？

其实Android程序的设计讲究逻辑和视图分离，因此是**不推荐在活动中直接编写界面的，更加通用的一种做法是，在布局文件中编写界面，然后在活动中引入进来**。可以看到，在onCreate方法的第二行调用了setContentView方法，就是这个方法给当前的活动引入了一个`hello_world_layout`布局，`Hello World!`就是在这里定义的。

布局文件都是定义在`res/layout`目录下的，展开layout目录，会看到`hello_world_layout.xml`这个文件。打开该文件并切换到Text视图，代码就出来了：

# 项目中的资源

1. 所有以drawable开头的文件夹都是用来放图片的；
2. 所有以mipmap开头的文件夹都是用来放应用图标的；
3. 所有以values开头的文件夹都是用来放字符串、样式、颜色等配置的；
4. layout文件夹是用来放布局文件的。

mipmap开头的文件夹有很多，主要是为了兼容各种设备。drawable文件夹也是相同的道理，虽然Android Studio没有自动生成，但应该自己创建drawable-hdpi、drawable-xhdpi、drawable-xxhdpi等文件夹。在制作程序的时候最好能够给同一张图片提供几个不同分辨率的版本，分别放在对应的文件夹下。当程序运行的时候，会自动根据当前运行设备分辨率的高低选择加载。如果只有一份图片，那就都放在drawable-xxhdpi文件夹下。

## values

打开`res/values/strings.xml`文件，内容如下所示：

```xml
<resources>
    <string name="app_name">HelloWorld</string>
</resources>
```

这里定义了一个应用程序名的字符串，有以下两种方式来引用它。

* 在代码中通过`R.string.app_name`可以获得该字符串的引用
* 在XML中通过`@string/app_name`可以获得该字符串的引用

其中string部分可以替换，如果引用的是图片资源可以替换成drawable，如果引用的是应用图标可以替换成mipmap，如果引用的是布局文件可以替换成layout

打开`AndroidManifest.xml`文件，找到如下代码：

```xml
<application>
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:supportRtl="true"
    android:theme="@style/AppTheme"
    ...
</application>
```

其中，HelloWorld项目的应用图标是通过`android:icon`来指定的，应用的名称则是通过`android:label`属性指定的。

# build.gradle文件

Android Studio采用Gradle来构建项目。Gradle使用一种基于Groovy的领域特定预言（DSL）来声明项目设置，摒弃了传统基于XML（如Ant和Maven）的烦琐配置。

HelloWorld项目中有两个`build.gradle`文件，一个是在最外层目录下，一个是在app目录下。

# Android系统源码目录

## 整体结构

各个版本的源码目录基本是类似的，如果是编译后的源码目录，会多一个out文件夹，用来存储编译产生的文件。

| Android源码根目录 | 描述                                                 |
| ----------------- | ---------------------------------------------------- |
| art               | ART运行环境                                          |
| bionic            | 系统C库                                              |
| bootable          | 启动引导相关代码                                     |
| build             | 存放系统编译规则及generic等基础开发包配置            |
| cts               | Android兼容性测试套件标准                            |
| dalvik            | Dalvik虚拟机                                         |
| developers        | 开发者目录                                           |
| development       | 与应用程序开发相关                                   |
| device            | 设备相关配置                                         |
| external          | 开源模组相关文件                                     |
| frameworks        | 应用程序框架，Android系统核心部分，由Java和`C++`编写 |
| hardware          | 主要是硬件抽象层的代码                               |
| libcore           | 核心库相关文件                                       |
| libnativehelper   | 动态库，实现JNI库的基础                              |
| out               | 编译完成后代码在此目录输出                           |
| packages          | 应用程序包                                           |
| pdk               | Plug Development Kit的缩写，本地开发套件             |
| platform_testing  | 平台测试                                             |
| prebuilts         | X86和ARM架构下预编译的一些资源                       |
| sdk               | SDK和模拟器                                          |
| system            | 底层文件系统库、应用和组件                           |
| toolchain         | 工具链文件                                           |
| tools             | 工具文件                                             |
| Makefile          | 全局Makefile文件，用来定义编译规则                   |

## packages目录结构

应用层位于整个Android系统的最上层，开发者开发的应用程序以及系统内置的应用程序都在应用层。源码根目录中的packages目录对应着系统应用层，它的目录结构如表所示。

| packages目录 | 描述           |
| ------------ | -------------- |
| apps         | 核心应用程序   |
| experimental | 第三方应用程序 |
| inputmethods | 输入法目录     |
| providers    | 内容提供者目录 |
| screensavers | 屏幕保护       |
| services     | 通信服务       |
| wallpapers   | 墙纸           |

## 应用框架层部分

应用框架层是系统的核心部分，一方面向上提供接口给应用层调用，另一方面向下与`C/C++`程序库及硬件抽象层等进行衔接。应用框架层的主要实现代码在`frameworks/base`和`frameworks/av`目录下，其中`frameworks/base`目录结构如表所示。

| frameworks/base目录 | 描述                    |
| ------------------- | ----------------------- |
| api                 | 定义API                 |
| cmds                | 重要命令：am、app_proce |
| core                | 核心库                  |
| data                | 字体和声音等数据文件    |
| graphics            | 与图形图像相关          |
| keystore            | 与数据签名证书相关      |
| libs                | 库                      |
| location            | 地理位置相关库          |
| media               | 多媒体相关库            |
| native              | 本地库                  |
| nfc-extras          | 与NFC相关               |
| obex                | 蓝牙传输                |
| opengl              | 2D/3D图形API            |
| packages            | 设置、TTS、VPN程序      |
| sax                 | XML解析器               |
| services            | 系统服务                |
| telephony           | 电话通信管理            |
| test-runner         | 测试工具相关            |
| tests               | 与测试相关              |
| tools               | 工具                    |
| wifi                | WiFi无线网络            |

## `C/C++`程序库部分

系统运行库层（Native）中的`C/C++`程序库的类型繁多，功能强大，`C/C++`程序库并不完全在一个目录中，这里给出几个常用且比较重要的`C/C++`程序库所在的目录位置，如表所示。

| 目录位置                                    | 描述                                             |
| ------------------------------------------- | ------------------------------------------------ |
| bionic                                      | Google开发的系统C库，以BSD许可形式开源           |
| `frameworks/av/media`                       | 系统媒体库                                       |
| `frameworks/native/opengl`                  | 第三方图形渲染库                                 |
| `frameworks/native/services/surfaceflinger` | 图形显示库，主要负责图形的渲染、叠加和绘制等功能 |
| `external/sqlite`                           | 轻量级关系型数据库SQLite的`C++`实现              |

Android运行时库的代码在`art/`目录中；硬件抽象层的代码在`hardware/`目录中，这是手机厂商改动最大的部分，根据手机终端所采用的硬件平台不同会有不同的实现。

# 源码阅读

系统源码的阅读有很多种方式，总的来说分为两种：一种是在线阅读，另一种是下载源码到本地用软件工具阅读。

1. 在线阅读
   1. Android 在线阅读源码的网站有很多，比如www.grepcode.com、www.androidxref.com，www.androidos.net.cn等，这里推荐使用www.androidxref.com进行在线阅读。
2. 使用本地工具，本地阅读源码可以采用Android Studio、Eclipse、Sublime和Source Insight等软件。
