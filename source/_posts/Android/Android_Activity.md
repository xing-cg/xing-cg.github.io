---
title: Android_Activity
typora-root-url: ../..
categories:
  - [Android]
tags:
  - null 
date: 2022/7/18
update:
comments:
published:
---

# 简介

`Activity`类是Android应用的关键组件，而Activity的启动和组合方式则是该平台应用模型的基本组成部分。在编程范式中，应用是通过`main()`方法启动的，而Android系统与此不同，它会调用与其生命周期特定阶段相对应的特定回调方法来启动`Activity`实例中的代码。

# Activity的概念

移动应用体验与桌面体验不同之处在于，用户与应用的互动并不总是在同一位置开始，而是经常以不确定的方式开始。例如，如果从主屏幕打开电子邮件应用，可能会看到电子邮件列表，如果通过社交媒体应用启动电子邮件应用，则可能会直接进入电子邮件应用的邮件撰写界面。

`Activity`类的目的就是促进这种范式的实现。当一个应用调用另一个应用时，调用方应用会调用另一个应用中的Activity，而不是整个应用。通过这种方式，Activity充当了应用与用户互动的入口点。可以将Activity实现为`Activity`类的子类。

Activity提供窗口供应用在其中绘制界面。此窗口通常会填满屏幕，但也可能比屏幕小，并浮动在其他窗口上面。通常，一个Activity实现应用中的一个屏幕。例如，应用中的一个Activity实现“偏好设置”屏幕，而另一个Activity实现“选择照片”屏幕。

大多数应用包含多个屏幕，这意味着它们包含多个Activity。通常，应用中的一个Activity会被指定为**主Activity，这是用户启动应用时出现的第一个屏幕**。然后，**每个Activity可以启动另一个Activity，以执行不同的操作**。例如，一个简单的电子邮件应用中的主Activity可能会提供显示电子邮件收件箱的屏幕。主Activity可能会从该屏幕启动其他Activity，以提供执行写邮件和打开邮件这类任务的屏幕。

虽然应用中的各个Activity协同工作形成凝聚的用户体验，但每个Activity与其他Activity之间只存在松散的关联，应用内不同Activity之间的依赖关系通常很小。事实上，Activity经常会启动属于其他应用的Activity。例如，浏览器应用可能会启动社交媒体应用的“分享”Activity。

要在应用中使用Activity，您必须在应用的清单中注册关于Activity的信息，并且必须适当地管理Activity的生命周期。

# 配置清单

要使应用能够使用Activity，必须在清单中声明Activity及其特定属性。

## 声明Activity

要声明Activity，请打开清单文件，并添加`<activity>`元素作为`<application>`元素的子元素。例如：

```xml
    <manifest ... >
      <application ... >
          <activity android:name=".ExampleActivity" />
          ...
      </application ... >
      ...
    </manifest >
```

此元素唯一的必要属性是`android:name`，该属性用于指定Activity的类名称。除了name，还可以添加用于定义标签、图标或界面主题等Activity特征的属性。

>注意：发布应用后，就不应再更改 Activity 名称，否则可能会破坏某些功能，例如应用快捷方式。如需详细了解发布后应避免的更改，参阅[不可更改的内容](http://android-developers.blogspot.com/2011/06/things-that-cannot-change.html)。

## 声明intent过滤器

Intent过滤器是Android平台的一项非常强大的功能。借助这项功能，不但可以根据显式请求启动Activity，还可以根据隐式请求启动Activity。例如，显式请求可能会告诉系统“在Gmail应用中启动`发送电子邮件Activity`”；而隐式请求可能会告诉系统“在任何能够完成此工作的Activity中启动`发送电子邮件屏幕`”。当**系统界面询问用户使用哪个应用来执行任务**时，这就是intent过滤器在起作用。

要使用此功能，需要在`<activity>`元素中声明`<intent-filter>`属性。此元素的定义包括`<action>`元素，以及可选的`<category>`元素和/或`<data>`元素。这些元素组合在一起，可以指定`Activity`能够响应的`intent`类型。例如，以下代码段展示了如何配置一个发送文本数据并接收其他Activity的文本数据发送请求的Activity：

```xml
    <activity android:name=".ExampleActivity" android:icon="@drawable/app_icon">
        <intent-filter>
            <action android:name="android.intent.action.SEND" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="text/plain" />
        </intent-filter>
    </activity>
```

在此示例中，`<action>`元素指定该Activity会发送数据。将`<category>`元素声明为`DEFAULT`可使`Activity`能够接收启动请求。`<data>`元素指定此Activity可以发送的数据类型。以下代码段展示了如何调用上述`Activity`：

```java
    // Create the text message with a string
    Intent sendIntent = new Intent();
    sendIntent.setAction(Intent.ACTION_SEND);
    sendIntent.setType("text/plain");
    sendIntent.putExtra(Intent.EXTRA_TEXT, textMessage);
    // Start the activity
    startActivity(sendIntent);
```

如果打算构建一个独立的应用，不允许其他应用激活其Activity，则不需要任何其他intent过滤器。不想让其他应用访问的Activity不应包含intent过滤器，可以自己使用显式intent启动它们。

## 声明权限

您可以使用清单的`<activity>`标记来控制哪些应用可以启动某个Activity。**父Activity和子Activity必须在其清单中具有相同的权限，前者才能启动后者**。如果您为父Activity声明了`<uses-permission>`元素，则每个子Activity都必须具有匹配的`<uses-permission>`元素。

例如，假设您的应用想要使用一个名为`SocialApp`的应用在社交媒体上分享文章，则SocialApp本身必须定义调用它的应用所需具备的权限：

```xml
    <manifest>
    <activity android:name="...."
       android:permission=”com.google.socialapp.permission.SHARE_POST”

    />
```

然后，为了能够调用SocialApp，应用必须匹配SocialApp清单中设置的权限：

```xml
    <manifest>
       <uses-permission android:name="com.google.socialapp.permission.SHARE_POST" />
    </manifest>
```

# Activity生命周期

一个Activity在其生命周期中会经历多种状态。可以使用一系列回调来处理状态之间的转换。下面将介绍这些回调。

![Activity生命周期的简化图示](https://developer.android.com/guide/components/images/activity_lifecycle.png)

## 返回栈

Android中的活动是可以层叠的。每启动一个新的活动，就会覆盖在原活动之上，点击Back键会销毁最上面的活动，下面的一个活动就会重新显示出来。

其实Android是使用任务（Task）来管理活动的，一个任务就是一组存放在栈里的活动的集合，这个栈也被称作返回栈（Back Stack）。栈是一种后进先出的数据结构，在默认情况下，每当我们启动了一个新的活动，它会在返回栈中入栈，并处于栈顶的位置。而每当我们按下Back键或调用finish方法去销毁一个活动时，处于栈顶的活动会出栈，这时前一个入栈的活动就会重新处于栈顶的位置。系统总是会显示处于栈顶的活动给用户。

## onCreate()

必须实现此回调，它会在系统创建Activity时触发。

Activity会在创建后进入“**已创建**”状态。在`onCreate()`方法中，需执行基本应用启动逻辑，该逻辑在Activity的整个生命周期中只应发生一次。例如，`onCreate()`的实现可能会将数据绑定到列表，将Activity与ViewModel相关联，并实例化某些类作用域变量。此方法会接收`savedInstanceState`参数，后者是包含Activity先前保存状态的`Bundle`对象。如果Activity此前未曾存在，`Bundle`对象的值为null。

如果有一个生命周期感知型组件与Activity生命周期相关联，该组件将收到`ON_CREATE`事件。系统将调用带有`@OnLifecycleEvent`注释的方法，以使生命周期感知型组件可以执行已创建状态所需的任何设置代码。

应该初始化Activity的基本组件：例如，应用应该在此处创建视图并将数据绑定到列表。最重要的是，必须在此处调用`setContentView()`来定义Activity界面的布局。

`onCreate()`方法的以下示例显示执行Activity某些基本设置的一些代码，例如声明界面（在XML布局文件中定义）、定义成员变量，以及配置某些界面。在本示例中，系统通过将文件的资源ID`R.layout.main_activity`传递给`setContentView()`来指定XML布局文件。

```java
TextView textView;
// some transient state for the activity instance
String gameState;
@Override
public void onCreate(Bundle savedInstanceState) {
    // call the super class onCreate to complete the creation of activity like
    // the view hierarchy
    super.onCreate(savedInstanceState);
    // recovering the instance state
    if (savedInstanceState != null) {
        gameState = savedInstanceState.getString(GAME_STATE_KEY);
    }
    // set the user interface layout for this activity
    // the layout file is defined in the project res/layout/main_activity.xml file
    setContentView(R.layout.main_activity);
    // initialize member TextView so we can manipulate it later
    textView = (TextView) findViewById(R.id.text_view);
}
// This callback is called only when there is a saved instance that is previously saved by using
// onSaveInstanceState(). We restore some state in onCreate(), while we can optionally restore
// other state here, possibly usable after onStart() has completed.
// The savedInstanceState Bundle is same as the one used in onCreate().
@Override
public void onRestoreInstanceState(Bundle savedInstanceState) {
    textView.setText(savedInstanceState.getString(TEXT_VIEW_KEY));
}
// invoked when the activity may be temporarily destroyed, save the instance state here
@Override
public void onSaveInstanceState(Bundle outState) {
    outState.putString(GAME_STATE_KEY, gameState);
    outState.putString(TEXT_VIEW_KEY, textView.getText());
    // call superclass to save any view hierarchy
    super.onSaveInstanceState(outState);
}
```

除了定义XML文件，然后将其传递给`setContentView()`，还可以在Activity代码中新建View对象，并将新建的View插入到`ViewGroup`中，以构建视图层次结构。然后，将根`ViewGroup`传递给`setContentView()`以使用该布局。

Activity并未处于“已创建”状态。`onCreate()`方法完成执行后，Activity进入“**已开始**”状态，系统会相继调用`onStart()`和`onResume()`方法。

`onCreate()`完成后，下一个回调将是`onStart()`。

## onStart()

`onCreate()`退出后，Activity将进入“**已开始**”状态（Started state），并对用户可见。此回调包含**Activity进入前台与用户进行互动之前的最后准备工作**。

当Activity进入“**已开始**”状态时，系统会调用此回调。`onStart()`调用使Activity对用户可见，因为应用会为Activity进入前台并支持互动做准备。例如，应用通过此方法来初始化保持界面的代码。（initializes the code that maintains the UI.）

当Activity进入已开始状态时，与Activity生命周期相关联的所有生命周期感知型组件都将收到`ON_START`事件。

`onStart()`方法会非常快速地完成。一旦此回调结束，Activity便会进入“**已恢复**”状态（Resumed state），系统将调用`onResume()`方法。

## onResume()

系统会在Activity开始与用户互动之前调用此回调。此时，该Activity位于Activity堆栈的顶部，并会**捕获所有用户输入**。应用的大部分核心功能都是在`onResume()`方法中实现的。

`onResume()`回调后面总是跟着`onPause()`回调。

Activity会在进入“已恢复”状态时来到前台，然后系统调用`onResume()`回调。这是应用与用户互动的状态。应用会一直保持这种状态，直到某些事件导致焦点远离应用，此类事件包括接到来电、用户导航到另一个Activity，或设备屏幕关闭。

当Activity进入已恢复状态时，与Activity生命周期相关联的所有生命周期感知型组件都将收到 `ON_RESUME`事件。这时，生命周期组件可以启用在组件可见且位于前台时需要运行的任何功能，例如启动相机预览。

当发生中断事件时，Activity进入“**已暂停**”状态，系统调用`onPause()`回调。

如果Activity从“已暂停”状态返回“已恢复”状态，系统将再次调用`onResume()`方法。因此，`onResume()`负责初始化在`onPause()`期间释放的组件，并执行每次Activity进入“已恢复”状态时必须完成的任何其他初始化操作。

以下是生命周期感知型组件的示例，该组件在收到`ON_RESUME`事件时访问相机：

```java
public class CameraComponent implements LifecycleObserver {
    ...
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void initializeCamera() {
        if (camera == null) {
            getCamera();
        }
    }
    ...
}
```

`LifecycleObserver`收到`ON_RESUME`事件后，上述代码便会初始化相机。然而，在多窗口模式下，即使处于“已暂停”状态，Activity也可能完全可见。例如，当用户处于多窗口模式，并点按另一个不包含Activity的窗口时，Activity将进入“已暂停”状态。如果希望相机仅在应用处于“已恢复”（可见且在前台运行）状态时可用，应在收到上述`ON_RESUME`事件后初始化相机。如果希望在Activity 处于“已暂停”状态但可见时（例如在多窗口模式下）保持相机可用，应在收到`ON_START`事件后初始化相机。但注意，若要让相机在Activity处于“已暂停”状态时可用，可能会导致系统在多窗口模式下拒绝其他处于“已恢复”状态的应用访问相机。有时可能有必要让相机在Activity处于“已暂停”状态时保持可用，但这样做实际可能会降低整体用户体验。生命周期的哪个阶段更适合在多窗口环境下控制共享系统资源需要仔细考虑。

无论选择在哪个构建事件中执行初始化操作，都务必使用相应的生命周期事件来释放资源。如果在收到`ON_START`事件后初始化某些内容，需在收到`ON_STOP`事件后释放或终止相应内容。如果在收到`ON_RESUME`事件后初始化某些内容，需在收到`ON_PAUSE`事件后将其释放。

注意，上述代码段将相机初始化代码放置在生命周期感知型组件中。可以直接将此代码放入Activity生命周期回调（例如`onStart()`和`onStop()`），但不建议这样做。通过将此逻辑添加到独立的生命周期感知型组件中，可以对多个Activity重复使用该组件，而无需复制代码。

## onPause()

当Activity失去焦点并进入“**已暂停**”状态时，系统就会调用`onPause()`。例如，当用户点按“`返回`”或“`最近使用的应用`”按钮时，就会出现此状态。当系统为Activity调用`onPause()`时，从技术上来说，这意味着Activity仍然部分可见，但大多数情况下，这表明用户正在离开该Activity，该Activity很快将进入“**已停止**”或“**已恢复**”状态。

如果用户希望界面继续更新，则处于“**已暂停**”状态的Activity也可以继续更新界面。例如，显示导航地图屏幕或播放媒体播放器的Activity就属于此类Activity。即使此类Activity失去了焦点，用户仍希望其界面继续更新。

不应使用`onPause()`来保存应用或用户数据、进行网络呼叫或执行数据库事务。

`onPause()`执行完毕后，下一个回调为`onStop()`或`onResume()`，具体取决于Activity进入“**已暂停**”状态后发生的情况。

---

系统将此方法视为用户将要离开Activity的第一个标志（尽管这并不总是意味着Activity会被销毁）；此方法表示Activity不再位于前台（尽管在用户处于多窗口模式时Activity仍然可见）。使用`onPause()`方法以暂停或调整当`Activity`处于“已暂停”状态时不应继续（或应有节制地继续）的操作，以及你希望很快恢复的操作。Activity进入此状态的原因有很多。例如：

- 如`onResume()`部分所述，某个事件会中断应用执行。这是最常见的情况。
- 在Android 7.0（API 级别 24）或更高版本中，有多个应用在多窗口模式下运行。无论何时，都只有一个应用（窗口）可以拥有焦点，因此系统会暂停所有其他应用。
- 有新的半透明Activity（例如对话框）处于开启状态。只要Activity仍然部分可见但并未处于焦点之中，它便会一直暂停。

当Activity进入已暂停状态时，与Activity生命周期相关联的所有生命周期感知型组件都将收到 `ON_PAUSE`事件。这时，生命周期组件可以停止在组件未位于前台时无需运行的任何功能，例如停止相机预览。

还可以使用`onPause()`方法释放系统资源、传感器（例如GPS）手柄，或当Activity暂停且用户不需要它们时仍然可能影响电池续航时间的任何资源。然而，正如上文的onResume()部分所述，如果处于多窗口模式，“已暂停”的Activity仍完全可见。因此，应该考虑使用onStop()而非onPause()来完全释放或调整与界面相关的资源和操作，以便更好地支持多窗口模式。

响应`ON_PAUSE`事件的以下`LifecycleObserver`示例与上述`ON_RESUME`事件示例相对应，会释放在收到`ON_RESUME`事件后初始化的相机：
```java
public class JavaCameraComponent implements LifecycleObserver {
    ...
    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void releaseCamera() {
        if (camera != null) {
            camera.release();
            camera = null;
        }
    }
    ...
}
```

请注意，上述代码段是在LifecycleObserver收到`ON_PAUSE`事件后释放相机。

`onPause()`的执行非常简短，不一定有足够的时间来执行保存操作。因此，不应使用 `onPause()`来保存应用或用户数据、进行网络调用或执行数据库事务。而应在`onStop()`期间执行高负载的关闭操作。

`onPause()`方法的完成**并不意味着Activity离开“已暂停”状态。相反，Activity会保持此状态**，直到其恢复或变成对用户完全不可见。如果Activity恢复，系统将再次调用`onResume()`回调。如果Activity从“已暂停”状态返回“已恢复”状态，系统会让Activity实例继续驻留在内存中，并会在系统调用`onResume()`时重新调用该实例。在这种情况下，无需重新初始化在任何回调方法导致Activity进入“已恢复”状态期间创建的组件。如果Activity变为完全不可见，系统会调用`onStop()`。

## onStop()

当Activity对用户不再可见时，系统会调用`onStop()`。出现这种情况的原因可能是Activity被销毁，新的Activity启动，或者现有的Activity正在进入“**已恢复**”状态并覆盖了已停止的Activity。在所有这些情况下，停止的Activity都将完全不再可见。

系统调用的下一个回调将是`onRestart()`（如果Activity重新与用户互动）或者`onDestroy()`（如果Activity彻底终止）。

---

如果Activity不再对用户可见，说明其已进入“已停止”状态，因此系统将调用`onStop()`回调。例如，当新启动的Activity覆盖整个屏幕时，可能会发生这种情况。系统也可能在Activity已结束运行并即将终止时调用`onStop()`。

当Activity进入已停止状态时，与Activity生命周期相关联的所有生命周期感知型组件都将收到`ON_STOP`事件。这时，生命周期组件可以停止在组件未显示在屏幕上时无需运行的任何功能。

在`onStop()`方法中，应用应释放或调整在应用对用户不可见时的无用资源。例如，应用可以暂停动画效果，或从精确位置更新切换到粗略位置更新。使用`onStop()`而非`onPause()`可确保与界面相关的工作继续进行，即使用户在多窗口模式下查看Activity也能如此。

应使用`onStop()`执行CPU相对密集的关闭操作。例如，如果您无法找到更合适的时机来将信息保存到数据库，可以在`onStop()`期间执行此操作。以下示例展示了`onStop()`的实现，它将草稿笔记内容保存到持久性存储空间中：

```java
@Override
protected void onStop() {
    // call the superclass method first
    super.onStop();

    // save the note's current draft, because the activity is stopping
    // and we want to be sure the current note progress isn't lost.
    ContentValues values = new ContentValues();
    values.put(NotePad.Notes.COLUMN_NAME_NOTE, getCurrentNoteText());
    values.put(NotePad.Notes.COLUMN_NAME_TITLE, getCurrentNoteTitle());

    // do this update in background on an AsyncQueryHandler or equivalent
    asyncQueryHandler.startUpdate (
            mToken,  // int token to correlate calls
            null,    // cookie, not used here
            uri,    // The URI for the note to update.
            values,  // The map of column names and new values to apply to them.
            null,    // No SELECT criteria are used.
            null     // No WHERE columns are used.
    );
}
```

注意，上述代码示例直接使用SQLite。但应该改用Room，这是一个通过SQLite提供抽象层的持久性库。

当Activity进入“已停止”状态时，Activity对象会继续驻留在内存中：**该对象将维护所有状态和成员信息，但不会附加到窗口管理器**（It maintains all state and member information, but is not attached to the window manager.）。Activity恢复后，Activity会重新调用这些信息。无需重新初始化在任何导致Activity进入“已恢复”状态的回调方法中创建的组件。系统还会追踪布局中每个View对象的当前状态，比如用户在`EditText`微件中输入文本，系统将保留文本内容，因此无需保存和恢复文本。

> 注意：Activity停止后，如果系统需要恢复一些内存，可能会销毁包含该Activity的进程。即使系统在Activity停止后销毁相应进程，系统仍会保留`Bundle`（键值对的blob）中View对象的状态，并在用户返回Activity时恢复这些对象。请参阅[保存和恢复 Activity 状态](https://developer.android.com/guide/components/activities/activity-lifecycle#saras)。

进入“已停止”状态后，Activity要么返回与用户互动，要么结束运行并消失。如果Activity返回，系统将调用`onRestart()`；如果Activity结束运行，系统将调用`onDestroy()`。

## onRestart()

当处于“**已停止**”状态的Activity即将重启时，系统就会调用此回调。`onRestart()`会从Activity停止时的状态恢复Activity。

此回调后面总是跟着`onStart()`。

## onDestroy()

系统会在销毁Activity之前调用此回调。

此回调是Activity接收的最后一个回调。通常，实现`onDestroy()`是为了确保在销毁Activity或包含该Activity的进程时释放该Activity的所有资源。

---

销毁Activity之前，系统会先调用`onDestroy()`。系统调用此回调的原因有：

1. Activity即将结束（由于用户彻底关闭Activity或由于系统为Activity调用`finish()`），或
2. 由于配置变更（例如设备旋转或多窗口模式），系统暂时销毁Activity。

当Activity进入已销毁状态时，与Activity生命周期相关联的所有生命周期感知型组件都将收到`ON_DESTROY`事件。这时，生命周期组件可以在Activity被销毁之前清理所需的任何数据。

应使用`ViewModel`对象来包含Activity的相关视图数据，而不是在Activity中加入逻辑来确定Activity被销毁的原因。如果因配置变更而重新创建Activity，ViewModel不必执行任何操作，因为系统将保留ViewModel并将其提供给下一个Activity实例。如果不重新创建Activity，ViewModel将调用`onCleared()`方法，以便在Activity被销毁前清除所需的任何数据。

可以使用`isFinishing()`方法区分这两种情况。

如果Activity即将结束，onDestroy()是Activity收到的最后一个生命周期回调。如果由于配置变更而调用onDestroy()，系统会立即新建Activity实例，然后在新配置中为新实例调用`onCreate()`。

`onDestroy()`回调应释放先前的回调，例如`onStop()`尚未释放的所有资源。

# Activity状态和从内存中弹出(ejection)

系统会在需要释放RAM时终止进程；系统终止给定进程的可能性取决于当时进程的状态，然而，**进程状态取决于在进程中运行的Activity的状态**。表1展示了进程状态、Activity状态以及系统终止进程的可能性之间的关系。

| 系统终止进程的可能性 | 进程状态                   | Activity状态           |
| -------------------- | -------------------------- | ---------------------- |
| 较小                 | 前台（拥有或即将得到焦点） | 已创建、已开始、已恢复 |
| 较大                 | 后台（失去焦点）           | 已暂停                 |
| 最大                 | 后台（不可见）、空         | 已停止、已销毁         |

系统永远不会直接终止Activity以释放内存，而是会终止Activity所在的进程。系统不仅会销毁Activity，还会销毁在该进程中运行的所有其他内容。如需了解如何在系统启动的进程被终止时保留和恢复Activity的界面状态，参阅[保存和恢复Activity状态](https://developer.android.com/guide/components/activities/activity-lifecycle#saras)。

用户还可以使用“设置”下的“应用管理器”来终止进程，以终止相应的应用。

了解一般进程，参阅[进程和线程](https://developer.android.com/guide/components/processes-and-threads)。如需详细了解进程生命周期如何与其中Activity的状态相关联，参阅[进程生命周期](https://developer.android.com/guide/components/processes-and-threads#Lifecycle)部分。

## 保存和恢复瞬时界面状态

用户期望Activity的界面状态在整个配置变更（例如旋转或切换到多窗口模式）期间保持不变。但是，**默认情况下，系统会在发生此类配置更改时销毁Activity，从而清除存储在Activity实例中的任何界面状态**。同样，如果用户暂时从应用切换到其他应用，并在稍后返回应用，用户希望界面状态保持不变。但是，当用户离开应用且Activity停止时，系统可能会销毁该应用的进程。

当Activity因系统限制而被销毁时，应组合使用`ViewModel`、`onSaveInstanceState()`和/或本地存储来保留用户的瞬时界面状态。可以在进一步详细了解用户期望与系统行为，以及如何在系统启动的Activity和进程被终止后最大程度地保留复杂的界面状态数据，参阅[保存界面状态](https://developer.android.com/topic/libraries/architecture/saving-states)。

下面概述实例状态的定义，以及如何实现onSaveInstance()方法，该方法是Activity本身的回调。如果界面数据简单且轻量，例如原始数据类型或简单对象（比如String），可以单独使用onSaveInstanceState()使界面状态在配置更改和系统启动的进程被终止时保持不变。但在大多数情况下，应使用ViewModel和onSaveInstanceState()（如[保存界面状态](https://developer.android.com/topic/libraries/architecture/saving-states)中所述），因为 onSaveInstanceState()会产生序列化/反序列化费用。

## 实例状态

在某些情况下，Activity会因正常的应用行为而被销毁，例如当用户按下返回按钮或Activity通过调用`finish()`方法发出销毁信号时，此时系统和用户对该Activity实例的概念将永远消失。在这些情况下，用户的期望与系统行为相匹配，无需完成任何额外工作。

但是，如果系统因系统限制（例如配置变更或内存压力）而销毁Activity，虽然实际的Activity实例会消失，但系统会记住它曾经存在过。如果用户尝试回退到该Activity，系统将**使用已保存的一组描述Activity销毁时状态的数据**重建该Activity实例。

系统用于恢复先前状态的已保存数据称为**实例状态**，是存储在**Bundle对象中的键值对集合**。默认情况下，系统使用`Bundle`实例状态来保存**Activity布局中每个View对象的相关信息**（例如在`EditText`微件中输入的文本值）。这样，如果Activity实例被销毁并重新创建，布局便会恢复为其先前的状态，且无需编写代码。但是，Activity可能包含你想要恢复的更多状态信息，例如追踪用户在Activity中的进程的成员变量。

> 注意：为了使Android系统恢复Activity中视图的状态，每个视图必须具有`android:id`属性提供的唯一ID。

`Bundle`对象并不适合保留大量数据，因为它需要在主线程上进行序列化处理并占用系统进程内存。如需保存大量数据，应组合使用持久性本地存储、`onSaveInstanceState()`方法和`ViewModel`类来保存数据，正如[保存界面状态](https://developer.android.com/topic/libraries/architecture/saving-states)中所述。

## 使用onSaveInstanceState()保存简单轻量的界面状态

当Activity开始停止时，系统会调用`onSaveInstanceState()`方法，以便Activity可以将状态信息保存到实例状态Bundle中。此方法的默认实现保存有关Activity视图层次结构状态的瞬时信息，例如`EditText`微件中的文本或`ListView`微件的滚动位置。

如需保存Activity的其他实例状态信息，必须替换`onSaveInstanceState()`，并将键值对添加到Activity意外销毁时事件中所保存的`Bundle`对象中。替换onSaveInstanceState()时，如果希望默认实现保存视图层次结构的状态，必须调用父类实现。例如：

```java
static final String STATE_SCORE = "playerScore";
static final String STATE_LEVEL = "playerLevel";
// ...
@Override
public void onSaveInstanceState(Bundle savedInstanceState) {
    // Save the user's current game state
    savedInstanceState.putInt(STATE_SCORE, currentScore);
    savedInstanceState.putInt(STATE_LEVEL, currentLevel);

    // Always call the superclass so it can save the view hierarchy state
    super.onSaveInstanceState(savedInstanceState);
}
```

> 注意：当用户显式关闭Activity时，或者在其他情况下调用`finish()`时，系统不会调用onSaveInstanceState()。

如需保存持久性数据（例如用户首选项或数据库中的数据），应在Activity位于前台时抓住合适机会。如果没有这样的时机，应在执行`onStop()`方法期间保存此类数据。

## 使用保存的实例状态恢复Activity界面状态

重建先前被销毁的Activity后，可以从系统传递给Activity的`Bundle`中恢复保存的实例状态。`onCreate()`和 `onRestoreInstanceState()`回调方法均会收到包含实例状态信息的相同`Bundle`。

因为无论系统是新建Activity实例还是重新创建之前的实例，都会调用`onCreate()`方法，所以在尝试读取之前，必须检查状态Bundle是否为null。如果为null，系统将新建Activity实例，而不会恢复之前销毁的实例。

例如，以下代码段显示如何在`onCreate()`中恢复某些状态数据：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState); // Always call the superclass first

    // Check whether we're recreating a previously destroyed instance
    if (savedInstanceState != null) {
        // Restore value of members from saved state
        currentScore = savedInstanceState.getInt(STATE_SCORE);
        currentLevel = savedInstanceState.getInt(STATE_LEVEL);
    } else {
        // Probably initialize members with default values for a new instance
    }
    // ...
}
```

可以选择实现系统在`onStart()`方法之后调用的`onRestoreInstanceState()`，而不是在`onCreate()`期间恢复状态。仅当存在要恢复的已保存状态时，系统才会调用`onRestoreInstanceState()`，因此您无需检查`Bundle`是否为 null：

```java
public void onRestoreInstanceState(Bundle savedInstanceState) {
    // Always call the superclass so it can restore the view hierarchy
    super.onRestoreInstanceState(savedInstanceState);

    // Restore state members from saved instance
    currentScore = savedInstanceState.getInt(STATE_SCORE);
    currentLevel = savedInstanceState.getInt(STATE_LEVEL);
}
```

>注意：应始终调用`onRestoreInstanceState()`的父类实现，以便默认实现可以恢复视图层次结构的状态。

# 处理Activity状态更改

用户触发和系统触发的不同事件会导致Activity从一个状态转换到另一个状态。下面介绍发生此类转换的一些常见情况，以及如何处理这些转换。

有关Activity状态的详情，参阅Activity生命周期。要了解如何借助ViewModel类来管理Activity生命周期，参阅了解ViewModel类。

## 配置发生了更改

有很多事件会触发配置更改。最显著的例子或许是横屏和竖屏之间的屏幕方向变化。其他情况，如语言或输入设备的改变等，也可能导致配置更改。

当配置发生更改时，Activity会被销毁并重新创建。原始Activity实例将触发`onPause()`、`onStop()`和`onDestroy()`回调。系统将创建新的Activity实例，并触发`onCreate()`、`onStart()`和`onResume()`回调。

结合使用ViewModels、onSaveInstanceState()方法和/或持久性本地存储，可使Activity的界面状态在配置发生更改后保持不变。在决定这些选项的组合方式时，需要考虑界面数据的复杂程度、应用的用例以及检索速度与内存使用的权衡。有关保存Activity界面状态的详情，请参阅[保存界面状态](https://developer.android.com/topic/libraries/architecture/saving-states)。

## 处理多窗口模式的情况

一旦应用进入多窗口模式（适用于Android 7.0（API级别24）及更高级别），系统会向当前运行的Activity发送配置更改通知，从而完成上述生命周期转换。如果已经处于多窗口模式的应用调整了大小，也会出现这种行为。Activity可以自行处理配置更改，也可以让系统销毁Activity并使用新维度重新创建。

有关多窗口模式生命周期的详情，参阅[多窗口模式支持](https://developer.android.com/guide/topics/ui/multi-window)页的[多窗口模式生命周期](https://developer.android.com/guide/topics/ui/multi-window#lifecycle)部分。

在多窗口模式下，虽然用户可以看到两个应用，但只有与用户交互的应用位于前台且具有焦点。该Activity处于“已恢复”状态，而另一个窗口中的应用则处于“已暂停”状态。

当用户从应用A切换到应用B时，系统会对应用A调用`onPause()`，对应用B调用`onResume()`。每当用户在应用之间切换时，系统就会在这两种方法之间切换。

有关多窗口模式的详情，参阅[多窗口模式支持](https://developer.android.com/guide/topics/ui/multi-window)。

## Activity或对话框显示在前台

如果有新的Activity或对话框出现在前台，并且**局部覆盖**了正在进行的Activity，则被覆盖的Activity会失去焦点并进入“已暂停”状态。然后，系统会调用`onPause()`。

当被覆盖的Activity返回到前台并重新获得焦点时，会调用`onResume()`。

如果有新的Activity或对话框出现在前台，夺取了焦点且**完全覆盖**了正在进行的Activity，则被覆盖的Activity会失去焦点并进入“已停止”状态。然后，系统会快速地接连调用`onPause()`和`onStop()`。

当被覆盖的Activity的同一实例返回到前台时，系统会对该Activity调用`onRestart()`、`onStart()`和`onResume()`。如果被覆盖的Activity的新实例进入后台，则系统不会调用`onRestart()`，而只会调用`onStart()`和`onResume()`。

>注意：当用户点按“概览（Overview）”或主屏幕按钮时，系统的行为就好像当前Activity已被完全覆盖一样。

## 用户点按“返回”按钮

如果Activity位于前台，并且用户点按了返回按钮，Activity将依次经历`onPause()`、`onStop()`和`onDestroy()`回调。活动不仅会被销毁，还会从返回堆栈中移除。

需要注意的是，在这种情况下，默认不会触发`onSaveInstanceState()`回调。此行为基于的假设是，用户点按返回按钮时不期望返回Activity的同一实例。不过，可以通过替换`onBackPressed()`方法实现某种自定义行为，例如“confirm-quit”对话框。

如果替换`onBackPressed()`方法，强烈建议从被替换的方法调用`super.onBackPressed()`。否则，返回按钮的行为可能会让用户感觉突兀。

## 系统终止应用进程

如果某个应用处于后台并且系统需要为前台应用释放额外的内存，则系统可能会终止后台应用以释放更多内存。要详细了解系统如何确定要销毁哪些进程，参阅[Activity状态和从内存中弹出](https://developer.android.com/guide/components/activities/activity-lifecycle#asem)以及[进程和应用生命周期](https://developer.android.com/guide/components/activities/process-lifecycle)。

要了解如何在系统终止应用进程时保存Activity界面状态，请参阅[保存和恢复Activity 状态](https://developer.android.com/guide/components/activities/activity-lifecycle#saras)。

# 在Activity之间导航

在应用的生命周期中，应用很可能会多次进入和退出Activity。例如，用户可以点按设备的返回按钮，或者Activity可能需要启动不同的Activity。本部分介绍如何实现Activity转换，包括从另一个Activity启动Activity、保存Activity状态，以及恢复Activity状态。

## 从一个Activity启动另一个Activity

Activity通常需要在某个时刻启动另一个Activity。例如，当应用需要从当前屏幕移动到新屏幕时，就会出现这种需求。

根据Activity是否希望从即将启动的新Activity中获取返回结果，可以使用`startActivity()`或`startActivityForResult()`方法启动新Activity。这两种方法都需要传入一个`Intent`对象。

`Intent`对象指定要启动的具体Activity，或描述要执行的操作类型（系统选择相应的Activity，该Activity甚至可以来自不同应用）。`Intent`对象还可以携带已启动的Activity的少量数据。如需详细了解`Intent`类，参阅[Intent 和 Intent 过滤器](https://developer.android.com/guide/components/intents-filters)。

> Intent对象的作用：
>
> 1. 指定要启动的具体Activity
> 2. 可以携带已启动的Activity的少量数据

## startActivity()

如果新启动的Activity不需要返回结果，当前Activity可以通过调用`startActivity()`方法来启动它。

在自己的应用本身中工作时，通常只需启动已知Activity。例如，以下代码段显示如何启动一个名为`SignInActivity`的 Activity。

```java
Intent intent = new Intent(this, SignInActivity.class);
startActivity(intent);
```

应用可能还希望使用Activity中的数据执行某些操作，例如发送电子邮件、短信或状态更新。在这种情况下，应用自身可能不具有执行此类操作所需的Activity，因此可以改为利用设备上其他应用提供的Activity来执行这些操作，这便是intent的真正价值所在。可以创建一个intent，对想执行的操作进行描述，系统会从其他应用启动相应的Activity。如果有多个Activity可以处理intent，用户可以选择要使用哪一个。例如，如果想允许用户发送电子邮件，可以创建以下intent：

```java
Intent intent = new Intent(Intent.ACTION_SEND);
intent.putExtra(Intent.EXTRA_EMAIL, recipientArray);
startActivity(intent);
```

添加到intent中的`EXTRA_EMAIL`extra是一个字符串数组，其中包含电子邮件的收件人电子邮件地址。当电子邮件应用响应此intent时，该应用会读取extra中提供的字符串数组，并将该数组放入电子邮件撰写表单的“收件人”字段。在这种情况下，电子邮件应用的Activity会启动，并且当用户完成操作时，你的Activity会继续运行。

## startActivityForResult()

有时，希望在Activity结束时从Activity中获取返回结果。例如，可以启动一项Activity，让用户在联系人列表中选择收件人；当Activity结束时，系统将返回用户选择的收件人。为此，可以调用`startActivityForResult(Intent, int)`方法，其中整数参数会标识该调用。此标识符用于消除来自同一Activity的多次`startActivityForResult(Intent, int)`调用之间的歧义。这不是全局标识符，不存在与其他应用或Activity冲突的风险。结果通过`onActivityResult(int, int, Intent)`方法返回。

当子级Activity退出时，它可以调用`setResult(int)`将数据返回到其父级。子级Activity必须始终提供结果代码，该结果代码可以是标准结果`RESULT_CANCELED`、`RESULT_OK`，也可以是从`RESULT_FIRST_USER`开始的任何自定义值。此外，子级Activity可以根据需要返回包含它所需的任何其他数据的`Intent`对象。父级Activity使用`onActivityResult(int, int, Intent)`方法，以及父级Activity最初提供的整数标识符来接收信息。

如果子级Activity由于任何原因（例如崩溃）而失败，父级Activity将收到代码为`RESULT_CANCELED`的结果。

```java
public class MyActivity extends Activity {
     // ...
     static final int PICK_CONTACT_REQUEST = 0;
     public boolean onKeyDown(int keyCode, KeyEvent event) {
         if (keyCode == KeyEvent.KEYCODE_DPAD_CENTER) {
             // When the user center presses, let them pick a contact.
             startActivityForResult(
                 new Intent(Intent.ACTION_PICK,
                 new Uri("content://contacts")),
                 PICK_CONTACT_REQUEST);
            return true;
         }
         return false;
     }
     protected void onActivityResult(int requestCode, int resultCode,
             Intent data) {
         if (requestCode == PICK_CONTACT_REQUEST) {
             if (resultCode == RESULT_OK) {
                 // A contact was picked.  Here we will just display it
                 // to the user.
                 startActivity(new Intent(Intent.ACTION_VIEW, data));
             }
         }
     }
 }
```

## 协调Activity

当一个Activity启动另一个Activity时，它们都会经历生命周期转换。第一个Activity停止运行并进入“已暂停”或“已停止”状态，同时创建另一个Activity。如果这些Activity共享保存到磁盘或其他位置的数据，必须要明确第一个Activity在创建第二个Activity之前并未完全停止。相反，启动第二个Activity的过程与停止第一个Activity的过程重叠。

生命周期回调的顺序已有明确定义，特别是当两个Activity在同一个进程（应用）中，并且其中一个要启动另一个时。以下是Activity A启动Activity B时的操作发生顺序：

1. Activity A的`onPause()`方法执行。
2. Activity B的`onCreate()`、`onStart()`和`onResume()`方法依次执行（Activity B现在具有用户焦点）。
3. 然后，如果Activity A在屏幕上不再显示，其`onStop()`方法执行。

可以利用这种可预测的生命周期回调顺序管理从一个Activity到另一个Activity的信息转换。
