---
title: Android_应用基础知识
typora-root-url: ../..
categories:
  - [Android]
tags:
  - null 
date: 2022/7/25
update:
comments:
published:
---

# Android应用

可以使用Kotlin、Java和`C++`语言编写Android应用。Android SDK工具会将代码连同任何数据和资源文件编译成一个APK（Android package, Android软件包），即带有`.apk`后缀的归档文件；或者Android App Bundle。

* 一个APK文件包含Android应用在运行时的必需内容，它也是Android设备用来安装应用的文件。
* 一个AAB文件包含一个Android应用程序项目，包括一些在运行时不需要的附加元数据。ABB是一种发布格式，是不可安装到Android设备上的，它推迟(defer)APK的生成、签名到更晚的阶段。在Google Play分发应用程序时，Google Play的服务器才生成优化后的APK，仅包含特定设备安装应用所需要的资源和代码。

每个 Android 应用都处于各自的安全沙盒中，并受以下 Android 安全功能的保护：

- Android 操作系统是一种多用户 Linux 系统，其中每个应用都是一个不同的用户；
- 默认情况下，系统会为每个应用分配一个唯一的 Linux 用户 ID（该 ID 仅由系统使用，应用并不知晓）。系统会为应用中的所有文件设置权限，使得只有被分配给该应用的用户id对应的用户才能访问这些文件；
- 每个进程都拥有自己的虚拟机 (VM)，因此应用代码独立于其他应用而运行。
- 默认情况下，每个应用都在其自己的 Linux 进程内运行。Android 系统会在需要执行任何应用组件时启动该进程，然后当不再需要该进程或系统必须为其他应用恢复内存时，其便会关闭该进程。

Android系统实现了**应用最少特权**的原则。也就是说，默认情况下，每个应用程序只能访问其工作所需的组件，没有更多。这创造了一个非常安全的环境，在该环境中，应用程序无法访问未经许可的系统部分。但是，一个应用程序可以通过其他方法与其他应用程序共享数据，并且应用程序可以访问系统服务：

* 可以安排两个app共享相同的Linux用户ID。在这种情况下，他们可以访问彼此的文件。为了节省系统资源，具有相同用户ID的app也可以安排在相同的Linux进程中运行并共享相同的VM。这些app还必须被签署相同的证书签名。
* 一个app可以请求许可来访问设备数据，比如设备的定位、摄像头、蓝牙连接。用户必须明确地授予这些许可。

# 应用程序组件（App components）

应用程序组件是Android应用程序的重要构件模块。系统或用户可以通过每个组件来进入app。有些组件依赖其他组件。

有四大类应用程序组件：

1. Activity
2. Service
3. Broadcast receiver
4. Content provider

每种类型都有一个独特的目的，并且具有独特的生命周期，可以定义组件的创建和破坏方式。

## Activity

Activity是与用户交互的切入点，代表着具有用户界面的一个屏幕。比如，一个邮箱app可能有一个activity，展示了一列新邮件，另一个activity用于撰写邮件，以及另一activity在阅读邮件。尽管这些activity共同工作去形成凝聚的用户体验，但每个activity独立于其他activity。例如，如果电子邮件app允许，其他的app可以从任意一个activity启动。比如，拍照app可以启动邮箱app的activity，以撰写一个新的邮件，由此允许用户来分享照片。

Activity促进着系统和app之间的以下关键交互：

* 持续跟踪用户当前关心什么（屏幕上有什么），来确保系统继续运行在托管着activity的进程。
* 了解先前使用的进程包含用户可能返回的内容（已停止的activity），因此更优先地保持这些进程。
* 帮助app处理终止其进程的情况，便于用户返回到已恢复其之前状态的activity。
* 为app提供一种用户流交互的实现，并让系统协调这些流。（最经典的示例是共享）

>An *activity* is the entry point for interacting with the user. It represents a single screen with a user interface. For example, an email app might have one activity that shows a list of new emails, another activity to compose an email, and another activity for reading emails. Although the activities work together to form a cohesive user experience in the email app, each one is independent of the others. As such, a different app can start any one of these activities if the email app allows it. For example, a camera app can start the activity in the email app that composes new mail to allow the user to share a picture. An activity facilitates the following key interactions between system and app:
>
>- Keeping track of what the user currently cares about (what is on screen) to ensure that the system keeps running the process that is hosting the activity.
>- Knowing that previously used processes contain things the user may return to (stopped activities), and thus more highly prioritize keeping those processes around.
>- Helping the app handle having its process killed so the user can return to activities with their previous state restored.
>- Providing a way for apps to implement user flows between each other, and for the system to coordinate these flows. (The most classic example here being share.)

# 异步消息Intent启动组件

在四种组件类型中，有三种类型（Activity, Service, Broadcast Receiver）均通过**异步消息Intent**进行启动，Intent会在运行时对各个组件进行互相绑定。可以将Intent视为请求其他组件做出动作的信使，无论这个组件是属于你的app还是其他app。

需要使用Intent对象创建Intent，该对象通过定义消息来启动特定组件（显式Intent）或特定的**组件类型**（隐式Intent）。

对于Activity和服务，Intent 会定义要执行的操作（例如，`view`或`send`某内容），并且可指定待操作数据的`URI`，以及正在启动的组件可能需要了解的信息。例如，Intent可能会传达对Activity的请求，以便显示图像或打开网页。在某些情况下，可以通过启动Activity来接收结果，这样Activity还会返回`Intent`中的结果。例如，您可以发出一个Intent，让用户选取某位联系人并将其返回给您。返回的Intent包含指向所选联系人的URI。

每种组件都有独特的启动方法：

1. 可以通过传递一个`Intent`调用`startActivity()`启动一个activity或给这个activity新的事情去做。（或者`startActivityForResult()`，当你想让activity返回一个结果时）
2. 

# 清单文件（AndroidManifest.xml）

在Android系统启动应用组件之前，系统必须通过读取应用的清单文件(`AndroidManifest.xml`)确认组件存在。应用必须在此文件中声明其所有组件，该文件必须**位于应用项目目录的根目录**中。

除了声明应用的组件外，清单文件还有许多其他作用，如：

- 确定应用需要的任何用户权限，如互联网访问权限或对用户联系人的读取权限。
- 根据应用使用的API，声明应用所需的最低[API级别](https://developer.android.com/guide/topics/manifest/uses-sdk-element#ApiLevels)。
- 声明应用使用或需要的硬件和软件功能，如相机、蓝牙服务或多点触摸屏幕。
- 声明应用需要链接的API库（Android框架API除外），如[Google地图库](http://code.google.com/android/add-ons/google-apis/maps-overview.html)。

## 声明组件

清单文件的主要任务是告知系统应用组件的相关信息。例如，清单文件可按如下所示声明Activity：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest ... >
    <application android:icon="@drawable/app_icon.png" ... >
        <activity android:name="com.example.project.ExampleActivity"
                  android:label="@string/example_label" ... >
        </activity>
        ...
    </application>
</manifest>
```

在`<application>`元素中，`android:icon`属性指向标识应用的图标所对应的资源。

在`<activity>`元素中，`android:name`属性指定`Activity`子类的完全限定类名，`android:label`属性指定用作Activity的用户可见标签的字符串。

您必须使用以下元素声明所有应用组件：

- Activity的`<activity>`元素。
- 服务的`<service>`元素。
- 广播接收器的`<receiver>`元素。
- 内容提供程序的`<provider>`元素。

如果未在清单文件中声明源代码中包含的Activity、服务和内容提供程序，则这些组件对系统不可见，因此不会运行。不过，可以`BroadcastReceiver`对象的形式，在清单中声明或在代码中动态创建广播接收器；以及通过调用`registerReceiver()`，在系统中注册广播接收器。

## 声明组件功能

如上文启动组件中所述，可以使用`Intent`来启动Activity、服务和广播接收器。您可以通过在Intent中显式命名目标组件（使用组件类名）来使用`Intent`。还可使用隐式Intent，通过它来描述要执行的操作类型和待操作数据（可选）。借助隐式 Intent，系统能够在设备上找到可执行该操作的组件，并启动该组件。如果有多个组件可以执行Intent所描述的操作，则**由用户选择使用哪一个组件**。

>注意：如果使用Intent来启动`Service`，需要使用显式Intent来确保应用的安全性。使用隐式Intent启动服务存在安全隐患，因为无法确定哪些服务将响应Intent，且用户无法看到哪些服务已启动。从Android 5.0（API级别21）开始，如果使用隐式Intent调用`bindService()`，系统会抛出异常。所以勿为服务声明Intent过滤器。

通过将收到的Intent与设备上其他应用的清单文件中提供的**Intent过滤器**进行比较，系统便可识别能响应Intent的组件。

在应用的清单文件中声明Activity时，您可以选择性地加入声明Activity功能的Intent过滤器，以便响应来自其他应用的 Intent。可以将`<intent-filter>`元素作为组件声明元素的子项进行添加，从而为组件声明Intent过滤器。

例如，如果构建的电子邮件应用包含用于撰写新电子邮件的Activity，则可通过声明Intent过滤器来响应`“send”`Intent（目的是发送新电子邮件），如下方示例所示：

```xml
<manifest ... >
    ...
    <application ... >
        <activity android:name="com.example.project.ComposeEmailActivity">
            <intent-filter>
                <action android:name="android.intent.action.SEND" />
                <data android:type="*/*" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

如果另一个应用创建包含`ACTION_SEND`操作的Intent并将其传递到`startActivity()`，则系统可能会启动您的Activity，以便用户能够草拟并发送电子邮件。

## 声明应用要求

Android设备多种多样，但并非所有设备都提供相同的特性和功能。以防将应用安装在缺少应用所需特性的设备上，必须通过在清单文件中声明设备和软件要求，为该应用支持的设备类型明确定义一个配置文件。其中的大多数声明只是为了提供信息，系统并不会读取它们，但Google Play等外部服务会读取它们，以便在用户通过其设备搜索应用时为用户提供过滤功能。

例如，如果应用需要相机功能，并使用Android 2.1（API级别7）中引入的 API，必须在清单文件中声明以下要求，如下方示例所示：

```xml
<manifest ... >
    <uses-feature android:name="android.hardware.camera.any"
                  android:required="true" />
    <uses-sdk android:minSdkVersion="7" android:targetSdkVersion="19" />
    ...
</manifest>
```

通过示例中所述的声明，没有相机且Android版本低于2.1的设备将无法从Google Play安装您的应用。不过，可以声明应用使用相机，但并不要求必须使用。在此情况下，`required`属性设置为`false`，并在运行时检查设备是否拥有相机，然后根据需要停用任何相机功能。

# 应用资源

Android应用并非仅包含代码，它还需要与源代码分离的资源，如图像、音频文件以及任何与应用的视觉呈现有关的内容。例如，可以通过XML文件定义Activity界面的动画、菜单、样式、颜色和布局。借助应用资源，无需修改代码即可轻松更新应用的各种特性。通过提供备用资源集，可以针对各种设备配置（如不同的语言和屏幕尺寸）优化应用。

对于在Android项目中加入的每一项资源，SDK构建工具均会定义唯一的整型ID，可以利用此ID来引用资源，这些资源或来自应用代码，或来自XML中定义的其他资源。例如，如果您的应用包含名为`logo.png`的图像文件（保存在`res/drawable/`目录中），则SDK工具会生成名为`R.drawable.logo`的资源ID。此ID映射到应用特定的整型数，您可以利用它来引用该图像，并将其插入您的界面。

如果提供与源代码分离的资源，则其中最重要的一个优点在于，可以提供适用于不同设备配置的备用资源。例如，通过在XML中定义界面字符串，可以将字符串翻译为其他语言，并将这些字符串保存在单独的文件中。然后，Android系统会根据向资源目录名称追加的语言限定符（如为法语字符串值追加`res/values-fr/`）和用户的语言设置，对界面应用相应的语言字符串。

Android支持许多不同的备用资源限定符。限定符是资源目录名称中加入的短字符串，用于定义这些资源适用的设备配置。例如，您应根据设备的屏幕方向和尺寸为Activity创建不同的布局。当设备屏幕为纵向（长型）时，可能想要一种垂直排列按钮的布局；但当屏幕为横向（宽型）时，可以按水平方向排列按钮。如要根据方向更改布局，可以定义两种不同的布局，然后对每个布局的目录名称应用相应的限定符。然后，系统会根据当前设备方向自动应用相应的布局。
