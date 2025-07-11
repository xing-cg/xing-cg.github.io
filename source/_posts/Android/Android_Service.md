---
title: Android_Service
typora-root-url: ../..
categories:
  - [Android]
tags:
  - null 
date: 2022/8/19
update:
comments:
published:
---

# 服务

服务是Android中实现程序后台运行的解决方案，适合不需要和用户交互而且长期运行的任务。服务的运行不依赖于任何用户界面，即使程序被切换到后台，或者用户打开了另外一个应用程序，服务仍然能够保持正常运行。不过需要注意的是，服务并不是运行在一个独立的进程当中的，而是依赖于创建服务时所在的应用程序进程。当某个应用程序进程被杀掉时，所有依赖于该进程的服务也会停止运行。另外，也不要被服务的后台概念所迷惑，实际上服务并不会自动开启线程，所有的代码都是默认运行在主线程当中的。也就是说，我们需要在服务的内部手动创建子线程，并在这里执行具体的任务，否则就有可能出现主线程被阻塞住的情况。

# 定义一个服务

新建一个ServiceTest项目，然后点击`com.example.servicetest`，`New`，`Service`，`Service`。将服务命名为`MyService`，Exported属性表示是否允许其他程序访问这个服务，Enabled属性表示是否启用这个服务。

观察MyService中的代码，如下所示：

```java
public class MyService extends Service {
    public MyService() {
        
    }
    @Override
    public IBinder onBind(Intent intent) {
        throw new UnsupportedOperationException("Not yet implemented");
    }
}
```

可见，MyService继承自Service类。

onBind方法特别醒目。这是Service中唯一的抽象方法，必须在子类中实现。

既然是定义一个服务，自然应该在服务中去处理一些事情，处理事情的逻辑应该写在哪里？这时，就要重写Service中的另外的一些方法。如下：

```java
public class MyService extends Service {
    ...
    @Override
    public void onCreate() {
        super.onCreate();
    }
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        return super.onStartCommand(intent, flags, startId);
    }
    @Override
    public void onDestroy() {
        super.onDestroy();
    }
}
```

可以看到，这里我们又重写了onCreate、onStartCommand和onDestroy这3个方法，它们是每个服务中最常用到的3个方法了。其中onCreate方法会在服务创建的时候调用，onStartCommand方法会在每次服务启动的时候调用，onDestroy方法会在服务销毁的时候调用。

通常情况下，如果我们希望服务一旦启动就立刻去执行某个动作，就可以将逻辑写在onStartCommand方法里。而当服务销毁时，我们又应该在onDestroy方法中去回收那些不再使用的资源。

另外需要注意，每一个服务都需要在AndroidManifest.xml文件中进行注册才能生效，这是Android四大组件共有的特点。打开AndroidManifest.xml文件瞧一瞧，代码如下所示：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.servicetest">
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        ...
        <service
            android:name=".MyService"
            android:enabled="true"
            android:exported="true">
        </service>
    </application>
</manifest>
```

这样的话，就已经将一个服务完全定义好了。Exported属性表示是否允许其他程序访问这个服务，Enabled属性表示是否启用这个服务。

# 启动和停止服务

定义好了服务之后，接下来就应该考虑如何去启动以及停止这个服务。启动和停止主要是借助Intent来实现。在ServiceTest项目中尝试去启动以及停止MyService这个服务。

首先，修改`activity_main.xml`中的代码。我们在布局文件中加入了两个按钮，分别是用于启动服务和停止服务的。如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <Button
        android:id="@+id/start_service"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Start Service"/>
    <Button
        android:id="@+id/stop_service"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Stop Service"/>
</LinearLayout>
```

然后修改MainActivity中的代码，如下所示：

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button startServiceBtn = (Button) findViewById(R.id.start_service);
        Button stopServiceBtn = (Button) findViewById(R.id.stop_service);
        startServiceBtn.setOnClickListener(this);
        stopServiceBtn.setOnClickListener(this);
    }
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.start_service:
                Intent startIntent = new Intent(this, MyService.class);
                startService(startIntent);
                break;
            case R.id.stop_service:
                Intent stopIntent = new Intent(this, MyService.class);
                stopService(stopIntent);
                break;
            default:
                break;
        }
    }
}
```

可以看到，这里在onCreate方法中分别获取到了Start Service按钮和Stop Service按钮的实例并给它们注册了点击事件。然后在Start Service按钮的点击事件里，我们构建出了一个Intent对象，并调用startService方法来启动MyService这个服务。在Stop Serivce按钮的点击事件里，我们同样构建出了一个Intent对象，并调用stopService方法来停止MyService这个服务。

startService和stopService方法都是定义在Context类中的，所以我们在活动里可以直接调用这两个方法。注意，这里完全是由活动来决定服务何时停止的，如果没有点击Stop Service按钮，服务就会一直处于运行状态。那服务有没有什么办法让自已停止下来呢？当然可以，只需要在MyService的任何一个位置调用stopSelf方法就能让这个服务停止下来了。那么接下来又有一个问题需要思考了，如何证实服务已经成功启动或者停止了呢？最简单的方法就是在MyService的几个方法中加入打印日志，如下所示：

```java
    @Override
    public void onCreate() {
        super.onCreate();
        Log.d("MyService", "onCreate executed");
    }
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.d("MyService", "onStartCommand executed");
        return super.onStartCommand(intent, flags, startId);
    }
    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d("MyService", "onDestroy executed");
    }
```

点击一下Start Service按钮，观察logcat中的打印日志。

```
com.example.servicetest D/MyService: onCreate executed
com.example.servicetest D/MyService: onStartCommand executed
```

MyService中的onCreate和onStartCommand方法都执行了，说明这个服务确实已经启动成功了，并且你还可以在`Settings`，`System`，`Developer options`，`Running services`中找到ServiceTest。可以在其中停止服务。则会看到日志：

```
com.example.servicetest D/MyService: onDestroy executed
```

由此证明，MyService确实已经成功停止下来了。话说回来，虽然我们已经学会了启动服务以及停止服务的方法，有一个疑惑，那就是onCreate方法和onStartCommand方法到底有什么区别呢？因为刚刚点击Start Service按钮后两个方法都执行了。其实onCreate方法是在服务第一次创建的时候调用的，而onStartCommand方法则在每次启动服务的时候都会调用，由于刚才我们是第一次点击Start Service按钮，服务此时还未创建（Create）过，所以两个方法都会执行，之后再连续多点击几次Start Service按钮，会发现只有onStartCommand得到执行。

# 活动和服务进行通信 - onBind

了解了启动和停止服务的方法，虽然服务是在活动里启动的，但在启动了服务之后，活动与服务基本就没有什么关系了。之后服务会一直处于运行状态，但具体运行的是什么逻辑，活动就控制不了了。这就类似于活动通知了服务一下：你可以启动了！然后服务就去忙自己的事情了，但活动并不知道服务到底去做了什么事情，以及完成得如何。

那么有没有什么办法能让活动和服务的关系更紧密一些呢，这就需要借助我们刚刚忽略的onBind方法了。

比如说，目前我们希望在MyService里提供一个下载功能，然后在活动中可以决定何时开始下载，以及随时查看下载进度。实现这个功能的思路是创建一个专门的Binder对象来对下载功能进行管理，修改MyService中的代码，如下所示：

```java
public class MyService extends Service {
    
    private DownloadBinder mBinder = new DownloadBinder();
    class DownloadBinder extends Binder {
        public void startDownload() {
            Log.d("MyService", "startDownload executed");
        }
        public int getProgress() {
            Log.d("MyService", "getProgress executed");
            return 0;
        }
    }
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
    
    ...
}
```

可以看到，这里我们新建了一个DownLoadBinder类，并让它继承自Binder，然后在它的内部提供了开始下载以及查看下载进度的方法。当然这只是两个模拟方法，并没有实现真正的功能，我们在这两个方法中分别打印了一行日志。

接着，MyService中创建了DownloadBinder的实例，然后在onBind方法里返回了这个实例，这样MyService中的工作就全部完成了。

下面就要看一看，在活动中如何去调用服务里的这些方法了。首先需要在布局文件里新增两个按钮，修改`activity_main.xml`中的代码，如下所示：

```xml
    <Button android:id="@+id/bind_service"
        android:layout_height="wrap_content"
        android:layout_width="match_parent"
        android:text="Bind Service"/>
    <Button android:id="@+id/unbind_service"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Unbind Service"/>
```

这两个按钮分别是用于活动绑定服务和活动取消绑定服务的，当一个活动和服务绑定了之后，就可以调用该服务里的Binder提供的方法了。修改MainActivity中的代码，如下所示：

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private MyService.DownloadBinder downloadBinder;
    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            downloadBinder = (MyService.DownloadBinder) service;
            downloadBinder.startDownload();
            downloadBinder.getProgress();
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // ...
        Button bindService = (Button) findViewById(R.id.bind_service);
        Button unbindService = (Button) findViewById(R.id.unbind_service);
        bindService.setOnClickListener(this);
        unbindService.setOnClickListener(this);
    }
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            // ...
            case R.id.bind_service:
                Intent bindIntent = new Intent(this, MyService.class);
                bindService(bindIntent, connection, BIND_AUTO_CREATE);
                break;
            case R.id.unbind_service:
                unbindService(connection);
                break;
            default:
                break;
        }
    }
}
```

这里首先创建了一个ServiceConnection的匿名类，在里面重写了onServiceConnected方法和onServiceDisconnected方法，这两个方法分别会在活动与服务成功绑定以及活动与服务的连接取消绑定的时候调用。在onServiceConnected方法中，我们又通过向下转型得到了DownloadBinder的实例，有了这个实例，活动和服务之间的关系就变得非常紧密了。现在我们可以在活动中根据具体的场景来调用DownloadBinder中的任何public方法，即实现了指挥服务干什么服务就去干什么的功能。这里仍然只是做了个简单的测试，在onServiceConnected方法中调用了DownloadBinder的startDownload和getProgress方法。

当然，现在活动和服务其实还没进行绑定呢，这个功能是在Bind Service按钮的点击事件里完成的。可以看到，这里我们仍然是构建出了一个Intent对象，然后调用bindService方法将MainActivity和 MyService进行绑定。

bindService方法接收3个参数，第一个参数就是构建出的Intent对象，第二个参数是前面创建出的ServiceConnection的实例，第三个参数则是一个标志位，这里传入`BIND_AUTO_CREATE`表示在活动和服务进行绑定后自动创建服务。这会使得MyService中的onCreate方法得到执行，**但onStartCommand方法不会执行**。

然后如果我们想解除活动和服务之间的绑定，调用一下unbindService方法就可以了，这也是Unbind Service按钮的点击事件里实现的功能。但是解除绑定不会结束服务。

点击一下 Bind Service按钮，然后观察logcat中的打印日志。

```
com.example.servicetest D/MyService: onCreate executed
com.example.servicetest D/MyService: startDownload executed
com.example.servicetest D/MyService: getProgress executed
```

可以看到，首先是MyService的onCreate方法得到了执行，然后startDownload和getProgress方法都得到了执行，说明确实已经在活动里成功调用了服务里提供的方法了。

另外需要注意，任何一个服务在整个应用程序范围内都是通用的，即MyService不仅可以和MainActivity绑定，还可以和任何一个其他的活动进行绑定，而且在绑定完成后它们都可以获取到相同的DownloadBinder实例。

# 服务的生命周期

前面使用到的onCreate、onStartCommand、onBind和onDestroy等方法都是在服务的生命周期内可能回调的方法。

一旦在项目的任何位置调用了Context的startService方法，相应的服务就会启动起来，并回调onStartCommand方法。如果这个服务之前还没有创建过，onCreate方法会先于onStartCommand方法执行。服务启动了之后会一直保持运行状态，直到 stopService或stopSelf方法被调用。

注意，虽然每调用一次startService方法，onStartCommand就会执行一次，但实际上每个服务都只会存在一个实例。所以不管你调用了多少次startService方法，只需调用一次stopService或stopSelf方法，服务就会停止下来了。

另外，还可以调用Context的bindService来获取一个服务的持久连接，这时就会回调服务中的onBind方法。类似地，如果这个服务之前还没有创建过，onCreate方法会先于onBind方法执行。之后，调用方可以获取到onBind方法里返回的IBinder对象的实例，这样就能自由地和服务进行通信了。只要调用方和服务之间的连接没有断开，服务就会一直保持运行状态。

当调用了startService方法后，又去调用stopService方法，这时服务中的onDestroy方法就会执行，表示服务已经销毁了。

但是需要注意，我们是完全有可能对一个服务既调用了startService方法，又调用了bindService方法的，这种情况下该如何才能让服务销毁掉呢？根据Android系统的机制，一个服务只要被启动或者被绑定了之后，就会一直处于运行状态，**必须要让以上两种条件同时不满足，服务才能被销毁**。所以，这种情况下要同时调用stopService和unbindService方法，onDestroy方法才会执行。

实测，当bind之后，直接点击stopservice不会调用destroy。如果点击了stopservice之后，只要unbind，就会立即destroy。可参阅：https://blog.csdn.net/qq_30591155/article/details/105200843

这样就已经把服务的生命周期完整地走了一遍。

# 前台服务

