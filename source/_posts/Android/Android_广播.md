---
title: Android_广播
categories:
  - - Android
tags: 
date: 2022/9/8
updated: 
comments: 
published:
---

# 广播

Android中的每个应用程序都可以对自己感兴趣的广播进行注册，这些广播可能是来自于系统的，也可能是来自于其他应用程序的。

Android提供了一套完整的API，允许应用程序自由地发送和接收广播。

广播主要可以分为两种类型：标准广播和有序广播。

## 标准广播

完全异步执行。广播发出之后，所有的BroadcastReceiver几乎会在同一时间收到这条广播消息。这种广播效率比较高，同时也意味着它是无法截断的。

## 有序广播

有序广播是一种同步执行的广播，广播发出之后，同一时刻只会有一个BroadcastReceiver能够收到这条广播消息，当这个BroadcastReceiver中的逻辑执行完毕后，广播才会继续传递。

优先级较高的BroadcastReceiver可以先收到广播消息，并且前面的BroadcastReceiver还可以截断正在传递的广播，这样后面的BroadcastReceiver就无法收到广播消息了。

# 接收系统广播

Android内置了很多系统级别的广播，可以在应用程序中通过监听这些广播来得到各种系统的状态信息。比如手机开机完成后会发出一条广播，电池的电量发生变化会发出一条广播，系统时间发生改变也会发出一条广播，等等。如果想要接收这些广播，就需要使用BroadcastReceiver。

应用可以根据感兴趣的广播，自由地注册BroadcastReceiver，这样当有相应的广播发出时，相应的BroadcastReceiver就能够收到该广播，并可以在内部进行逻辑处理。

注册BroadcastReceiver的方式一般有两种：在代码中注册和在AndroidManifest.xml中注册。其中前者也被称为动态注册，后者也被称为静态注册。

## 动态注册监听时间变化

创建一个BroadcastReceiver其实只需新建一个类，让它继承自BroadcastReceiver，并重写父类的onReceive方法就行了。这样当有广播到来时，onReceive方法就会得到执行，具体的逻辑就可以在这个方法中处理。

下面先通过动态注册的方式编写一个能够监听时间变化的程序，体会BroadcastReceiver的基本用法。新建一个BroadcastTest项目，然后修改MainActivity中的代码。

如下所示：

```kotlin
class MainActivity : AppCompatActivity() {
    inner class TimeChangeReceiver : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            Toast.makeText(context, "Time has changed", Toast.LENGTH_SHORT).show()
        }
    }
    
    lateinit var timeChangeReceiver: TimeChangeReceiver
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        val intentFilter = IntentFilter()
        intentFilter.addAction("android.intent.action.TIME_TICK")
        timeChangeReceiver = TimeChangeReceiver()
        registerReceiver(timeChangeReceiver, intentFilter)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        unregisterReceiver(timeChangeReceiver)
    }
}
```

在MainActivity中定义了一个内部类TimeChangeReceiver，这个类是继承自BroadcastReceiver的，并重写了父类的onReceive方法。这样每当系统时间发生变化时，onReceive方法就会得到执行，这里只是简单地使用Toast提示了一段文本信息。

然后观察onCreate方法，首先我们创建了一个IntentFilter的实例，并给它添加了一个值为`android.intent.action.TIME_TICK`的action，每当系统时间发生变化时，系统发出此值的广播。BroadcastReceiver想要监听什么广播，就在这里添加相应的action。

接下来创建了一个TimeChangeReceiver的实例，然后调用registerReceiver方法进行注册，将TimeChangeReceiver的实例和IntentFilter的实例都传了进去，这样TimeChangeReceiver就会收到所有值为`android.intent.action.TIME_TICK`的广播，也就实现了监听系统时间变化的功能。

最后要记得，一定要注销动态注册的BroadcastReceiver才行，在onDestroy方法中通过调用unregisterReceiver方法来实现。

现在运行一下程序，然后静静等待时间发生变化。系统每隔一分钟就会发出一条`android.intent.action.TIME_TICK`的广播，因此最多只需要等待一分钟就可以收到这条广播。

### 总结

接收其他系统广播的用法是一模一样的。Android 系统还会在亮屏熄屏、电量变化、网络变化等场景下发出厂播。完整的系统厂播列表到如下的路径中查看：`<Android SDK>/platforms/<任意android api版本>/data/broadcast_actions.txt`。

## 静态注册实现开机启动

动态注册的BroadcastReceiver可以自由地控制注册与注销，在灵活性方面有很大的优势。但是它存在着一个缺点，即必须在程序启动之后才能接收广播，因为注册的逻辑是写在onCreate方法中的。

让程序在未启动的情况下也能接收广播需要使用静态注册的方式。从理论上来说，动态注册能监听到的系统广播，静态注册也应该能监听到，在过去的Android系统中确实是这样。但是由于大量恶意的应用程序利用这个机制在程序未启动的情况下监听系统广播，从而使任何应用都可以频繁地从后台被唤醒，严重影响了用户手机的电量和性能，因此Android系统几乎每个版本都在削减静态注册BroadcastReceiver的功能。

## 隐式广播

在Android 8.0系统之后，所有隐式广播都不允许使用静态注册的方式来接收了。隐式广播指的是那些没有具体指定发送给哪个应用程序的广播，大多数系统广播属于隐式广播，但是少数特殊的系统广播目前仍然允许使用静态注册的方式来接收。这些特殊的系统广播列表详见https://developer.android.google.cn/guide/components/broadcast-exceptions.html。

在这些特殊的系统广播当中，有一条值为`android.intent.action.BOOT_COMPLETED`的广播，是一条开机广播，用它来举例。

实现一个开机启动的功能。在开机的时候，应用程序肯定是没有启动的，因此这个功能显然不能使用动态注册的方式来实现，而应该使用静态注册的方式来接收开机广播，然后在onReceive方法里执行相应的逻辑，这样就可以实现开机启动的功能了。

```kotlin
class BootCompleteReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        Toast.makeText(context, "Boot Complete", Toast.LENGTH_LONG).show()
    }
}
```

静态的BroadcastReceiver一定要在AndroidManifest.xml文件中注册才可以使用。

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.example.broadcasttest">
    <application android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        ...
        <!-- + -->
        <receiver
            android:name=".BootCompleteReceiver"
            android:enabled="true"
            android:exported="true">
        </receiver>
    </appLication>
</manifest>
```

可以看到，`<application>`标签内出现了一个新的标签`<receiver>`所有静态的BroadcastReceiver都是在这里进行注册的。它的用法其实和`<activity>`标签非常相似，也是通过`android:name`指定具体注册哪一个BroadcastReceiver，Exported属性表示是否允许这个BroadcastReceiver**接收本程序以外的广播**，Enabled属性表示是否启用这个BroadcastReceiver。

不过目前的BootCompleteReceiver是无法收到开机广播的，还需要对AndroidManifest.xml文件进行修改才行，如下所示：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.example.broadcasttest">
    <!-- + -->
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <application android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        ...
        <receiver
            android:name=".BootCompleteReceiver"
            android:enabled="true"
            android:exported="true">
            <!-- + -->
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
            </intent-filter>
        </receiver>
    </appLication>
</manifest>
```

由于Android系统启动完成后会发出一条值为`android.intent.action.BOOT_COMPLETED`的广播，因此我们在`<receiver>`标签中又添加了一个`<intent-filter>`标签，并在里面声明了相应的action。

另外，Android系统为了保护用户设备的安全和隐私，做了严格的规定：如果程序需要进行一些对用户来说比较敏感的操作，必须在AndroidManifest.xml文件中进行权限声明。否则程序将会直接崩溃。比如这里接收系统的开机广播就是需要进行权限声明的，所以我们在上述代码中使用`<uses-permission>`标签声明了`android.permission.RECEIVE_BOOT_COMPLETED`——接收系统开机广播的权限。Android 6.0系统中引入了更加严格的运行时权限，从而能够更好地保证用户设备的安全和隐私。

重新运行程序，则程序已经可以接收开机广播了。重启设备，在启动完成之后就会收到开机广播。

### 总结

需要注意的是，不要在onReceive方法中添加过多的逻辑或者进行任何的耗时操作，因为BroadcastReceiver中是不允许开启线程的，当onReceive方法运行了较长时间而没有结束时，程序就会出现错误。

# 发送自定义广播

前面讲的是系统广播，现在看一下在应用程序中发送自定义的广播。

## 发送标准广播

在发送广播之前，我们还是需要先定义一个BroadcastReceiver来准备接收此广播，不然发出去也是白发。因此新建一个MyBroadcastReceiver，并在onReceive方法中加人如下代码：

```kotlin
class MyBroadcastReceiver : BroadcastReceiver() {
    override fun onReceiver(context: Context, intent: Intent) {
        Toast.makeText(context, "received in MyBroadcastReceiver", Toast.LENGTH_SHORT).show()
    }
}
```

当MyBroadcastReceiver收到自定义的广播时。就会弹出"received in MyBroadcastReceiver"的提示。然后在AndroidManifest.xml中对这个BroadcastReceiver进行修改：

```xml
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
    <application
        ...>
        <receiver android:name=".MyBroadcastReceiver"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="com.example.broadcasttest.MY_BROADCAST"/>
            </intent-filter>
        </receiver>
    </application>
```

可以看到，这里让MyBroadcastReceiver接收一条值为`com.example.broadcasttest.MY_BROADCAST`的广播，因此待会儿在发送广播的时候，我们就需要发出这样的一条广播。

接下来修改`activity_main.xml`中的代码，如下：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
    <Button
        android:id="@+id/button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Send Broadcast">
    </Button>

</LinearLayout>
```

这里在布局文件中定义了一个按钮。用于作为发送广播的触发点。

然后修改MainActivity中的代码，如下所示：

```kotlin
class MyBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        Toast.makeText(context, "received in MyBroadcastReceiver",
            Toast.LENGTH_LONG).show()
    }
}
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        val button: Button = findViewById(R.id.button)
        button.setOnClickListener {
            val intent = Intent("com.example.broadcasttest.MY_BROADCAST")
            intent.setPackage(packageName)
            sendBroadcast(intent)
        }
    }
}
```

可以看到，我们在按钮的点击事件里面加入了发送自定义广播的逻辑。

首先构建了一个Intent对象并把要发送的广播的值传入。然后调用Intent的setPackage方法，并传入当前应用程序的包名。packageName是getPackageName的语法糖写法，用于获取当前应用程序的包名。最后调用sendBroadcast方法将广播发送出去，这样所有监听`com.example.broadcasttest.MY_BROADCAST`这条广播的BroadcastReceiver就会收到消息了。

此时发出去的广播就是一条**标准广播**。

> 对setPackage方法进行更详细的说明：前面已经说过，在Android8.0系统之后，静态注册的BroadcastReceiver是无法接收隐式广播的，而**默认情况下我们发出的自定义广播恰恰都是隐式广播**。因此这里一定要调用setPackage方法，指定这条广播是发送给哪个应用程序的，从而让它变成一条显式广播，否则静态注册的BroadcastReceiver将无法接收到这条广播。

现在重新运行程序并点击"Send Broadcast"按钮，即可看到效果。

另外，由于广播是使用Intent来发送的、因此你还可以在Intent中携带一些数据传递给相应的BroadcastReceiver，这一点和Activity的用法是比较相似的。

## 发送有序广播

和标准广播不同，有序广播是一种同步执行的广播，并且是可以被截断的。为了验证这一点，我们需要再创建一个新的BroadcastReceiver。新建AnotherBroadcastReceiver，代码如下所示：

```kotlin
class AnotherBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        Toast.makeText(context, "received in AnotherBroadcastReceiver",
            Toast.LENGTH_SHORT).show()
    }
}
```

很简单，这里仍然是在onReceive方法中弹出了一段文本信息。

然后在AndroidManifest.xml中对这个BroadcastReceiver的配置进行修改，代码如下所示：

```xml
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
    <application
        ...>
        <receiver
            android:name=".AnotherBroadcastReceiver"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="com.example.broadcasttest.MY_BROADCAST" />
            </intent-filter>
        </receiver>
    </application>
```

可以看到，AnotherBroadcastReceiver同样接收的是`com.example.broadcasttest.MY_BROADCAST`这条广播。现在重新运行程序，并点击"Send Broadcast"按钮，就会分别弹出两次提示信息。

不过，到目前为止，程序发出的都是标准广播。现在我们来尝试一下发送有序广播。重新回到BroadcastTest项目，然后修改MainActivity中的代码，如下所示：

```kotlin
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        val button: Button = findViewById(R.id.button)
        button.setOnClickListener {
            val intent = Intent("com.example.broadcasttest.MY_BROADCAST")
            intent.setPackage(packageName)
            // modify
            sendOrderedBroadcast(intent, null)
        }
```

可以看到，发送有序广播只需要改动一行代码，即将sendBroadcast方法改成sendOrderedBroadcast方法。send0rderedBroadcast方法接收两个参数：第一个参数仍然是Intent；第二个参数是一个**与权限相关的字符串**，这里传入null就行了。现在重新运行程序，并点击"Send Broadcast"按钮，你会发现，两个BroadcastReceiver仍然都可以收到这条广播。

看上去好像和标准广播并没有什么区别。不过别忘了，这个时候的BroadcastReceiver是有先后顺序的，而且前面的BroadcastReceiver还可以将广播截断，以阻止其继续传播。

设定BroadcastReceiver的先后顺序：在注册的时候进行设定。修改AndroidManifest.xml中的代码，如下所示：

```xml
<receiver
    android:name=".MyBroadcastReceiver"
    android:enabled="true"
    android:exported="true">
                   <!-- + -->
    <intent-filter android:priority="100">
        <action android:name="com.example.broadcasttest.MY_BROADCAST"/>
    </intent-filter>
</receiver>
```

可以看到，我们通过`android:priority`属性给`BroadcastReceiver`设置了优先级，优先级比较高的BroadcastReceiver就可以先收到广播。这里将MyBroadcastReceiver的优先级设成了100。以保证它一定会在AnotherBroadcastReceiver之前收到广播。

既然已经获得了接收广播的优先权，那么MyBroadcastReceiver就可以选择是否允许广播继续传递了。修改MyBroadcastReceiver中的代码，如下所示：

```kotlin
class MyBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        Toast.makeText(context, "received in MyBroadcastReceiver",
            Toast.LENGTH_SHORT).show()
        // modify
        abortBroadcast()
    }
}
```

如果在onReceive方法中调用了abortBroadcast方法，就表示将这条广播截断，后面的BroadcastReceiver将无法再接收到这条广播。

现在重新运行程序，并点击"Send Broadcast"按钮，你会发现只有MyBroadcastReceiver中的Toast信息能够弹出，说明这条广播经过MyBroadcastReceiver之后确实终止传递了。