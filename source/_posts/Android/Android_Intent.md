---
title: Android_Intent
typora-root-url: ../..
categories:
  - [Android]
tags:
  - null 
date: 2022/7/29
update:
comments:
published:
---

# 简介

Intent主要解决活动之间的相互跳转。

# FirstActivity

从Add No Activity手动创建活动。

右击`com.example.activitytest`包，新建一个Empty Activity，命名为`FirstActivity.java`。

创建和加载布局：

1. 右击`app/src/main/res`目录，新建一个目录，名为`layout`，对着`layout`目录右键新建一个Layout resource file，会弹出一个新建布局资源文件的窗口，将此布局文件命名为`first_layout.xml`，根元素写成LinearLayout。

2. 编辑`first_layout.xml`。
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
       android:layout_width="match_parent"
       android:layout_height="match_parent"
       android:orientation="vertical">
   
       <Button
           android:id="@+id/button_1"
           android:layout_width="match_parent"
           android:layout_height="wrap_content"
           android:text="Button 1"></Button>
   </LinearLayout>
   <!-- 这里添加了一个Button元素，并在Button元素的内部增加了几个属性。
   android:id是给当前的元素定义一个唯一标识符，之后可以在代码中对这个元素进行操作。
   @+id/button_1这种语法可能比较陌生，但如果把加号去掉，变成@id/button_1就成为了在xml中引用资源的语法。
   只不过是把string替换成了id。这说明，如果你需要在xml中引用一个id，就使用@id/id_name这种语法。
   而如果需要在xml中定义一个id则要使用@+id/id_name这种语法。
   随后android:layout_width指定了当前元素的宽度；
   使用match_parent表示让当前元素和父元素一样宽。
   android:layout_height指定了当前元素的高度。
   这里使用wrap_content表示当前元素的高度只要能刚好包含里面的内容就行。
   android:text指定了元素中显示的文字内容 -->
   ```

3. 重新回到FirstActivity,java，在onCreate方法中加入一行代码
   ```java
   public class FirstActivity extends AppCompatActivity {
       @Override
       protected void onCreate(Bundle savedInstanceState) {
           super.onCreate(savedInstanceState);   //默认有这行
           /**
            * 这里调用了setContentView方法来给当前活动加载一个布局，
            * 一般传入一个布局文件的id。
            * 项目中添加的任何资源都会在R文件中生成一个相应的资源id，
            * 因此创建的first_layout.xml布局的id已添加到R文件中。
            * 只需要调用R.layout.first_layout就可以得到first_layout.xml布局的id
            */
           setContentView(R.layout.first_layout);//加入此行
       }
   }
   ```

4. 所有的活动都要在AndroidManifest.xml文件中注册才能生效。在`application`标签内通过添加`activity`标签来对活动进行注册。但是仅仅注册活动还是不够的，因为还没为程序配置主活动。也就是说程序运行起来后不知道首先要启动哪个活动。配置主活动的方法就是在`<activity>`标签的内部加入`<intent-filter>`标签，并在此标签里添加`<action android:name="android.intnet.action.MAIN" />`和`<category android:name="android.intent.category.LAUNCHER" />`这两句声明即可。
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <manifest xmlns:android="http://schemas.android.com/apk/res/android"
       package="com.example.intenttest">
   
       <application
           android:allowBackup="true"
           android:icon="@mipmap/ic_launcher"
           android:label="@string/app_name"
           android:roundIcon="@mipmap/ic_launcher_round"
           android:supportsRtl="true"
           android:theme="@style/Theme.IntentTest">
           <!-- 此处定义程序的主活动，即点击桌面应用程序图标时首先打开的就是这个活动
                如果没有声明任何一个活动作为主活动，这个程序仍然是可以正常安装，
                只是无法在启动器中看到或者打开此程序。这种程序一般是作为第三方服务供其他应用在内部进行调用，
                如支付宝快捷支付服务。-->
           <!-- 在<activity>中使用了android:name来指定具体注册哪一个活动。
                .FirstAcitivty表示com.example.activitytest.FirstActivity的缩写。
                由于最外层的<manifest>标签中已经通过package属性指定了程序的包名是com.example.activitytest，因此在注册活动时这一部分可以省略，直接使用.FirstActivity。-->
               <!-- android:label可指定活动中标题栏的内容，标题栏是显示在活动最顶部。
                    给主活动指定的label不仅会成为标题栏中的内容，还会成为启动器(Launcher)
                    中应用程序显示的名称。-->
           <activity
               android:name=".FirstActivity"
               android:label="This is FirstActivity"
               android:exported="true">
               <intent-filter>
                   <action android:name="android.intent.action.MAIN" />
                   <category android:name="android.intent.category.LAUNCHER" />
               </intent-filter>
           </activity>
       </application>
   </manifest>
   ```

销毁活动：

1. 如何销毁一个活动？最简单的方法是按一下Back键就可以销毁当前的活动。如果不想通过按键的方式，而是希望在程序中通过代码来销毁活动，则调用Activity类提供的finish方法，就可以销毁当前活动了。

2. 修改按钮监听器中的代码：
   ```java
   button1.setOnClickListener(new View.OnClickListener() {
       @Override
       public void onClick(View v) {
           finish();
       }
   });
   ```

3. 重新运行程序，点击一下按钮，当前的活动就被销毁了。效果和按下back键是一样的。

# Intent

编写第二个活动。创建一个Empty Activity，活动的对话框中，不要勾选Launcher Activity选项。

由于SecondActivity不是主活动，因此不需要配置`<intent-filter>`标签里的内容，注册活动的代码也简单了很多。

剩下的问题是如何去启动此SecondActivity活动。

需要引入一个概念——Intent。

Intent是Android程序中各组件之间进行交互的一种重要方式，不仅可以指明当前组件想要执行的动作，还可以在不同组件之间传递数据。

Intent一般可被用于启动活动、启动服务以及发送广播等场景。

Intent大致可以分为两种：显式Intent和隐式Intent。

# 显式Intent

意图很明显的Intent，称之为显式Intent。

Intent有多个构造函数的重载，其中一个是`Intent(Context packageContext, Class<?> cls)`。这个构造函数接受两个参数，第一个参数`Context`要求提供一个启动活动的上下文，第二个参数Class则是指定想要启动的目标活动。

如何使用此Intent？

Activity类中提供了一个`startActivity()`方法，此方法是专门用于启动活动的，接收一个Intent参数。可把构建好的Intent传入startActivity方法就可以启动目标活动了。

FirstActivity中按钮的点击事件，代码如下：

```java
button1.setOnClickListener(new View.OnClickListener(){
    @Override
    public void onClick(View v){
        Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
        startActivity(intent);
    }
});
```

首先构建出了一个Intent，传入`FirstActivity.this`作为上下文，传入`SecondActivity.class`作为目标活动。这个Intent意图即为在FirstActivity这个活动的基础上打开SecondActivity这个活动。通过startActivity()方法来执行这个Intent。

# 隐式Intent

相比于显式Intent，隐式Intent则含蓄了许多，它并不明确指出要启动哪一个活动，而是指定了一系列抽象的action和category等信息。然后交由系统去分析这个Intent，帮忙找出合适的活动来启动。

何为合适的活动？简单来说是可以响应我们这个隐式Intent的活动。目前SecondActivity还不能响应隐式Intent。

通过在`<activity>`标签下配置`<intent-filter>`的内容，可以指定当前活动能够响应的`action`和`category`，打开`AndroidManifest.xml`，添加如下代码：

```xml
<activity android:name=".SecondActivity" >
    <intent-filter>
        <action android:name="com.example.activitytest.ACTION_START"></action>
        <category android:name="android.intent.category.DEFAULT"></category>
    </intent-filter>
</activity>
```

在`<action>`这个标签中我们指明了当前活动可以响应`com.example.activity.ACTION_START`这个action，而`<category>`标签则包含了一些附加信息，更精确地指明了当前的活动能够响应的Intent中还可能带有的category。只有`<action>`和`<category>`中的内容同时能够匹配上Intent中指定的action和category时，这个活动才能响应该Intent。

修改FirstActivity中按钮的点击事件，代码如下所示：

```java
button1.setOnClickListener(new View.OnClickListener(){
    @Override
    public void onClick(View v){
        Intent intent = new Intent("com.example.activitytest.ACTION_START");
        startActivity(intent);
    }
});
```

此处使用到了Intent的另一个构造函数，直接将action的字符串传了进去，表明想要启动能够响应`com.example.activitytest.ACTION_START`这个action的活动。为何没有指定category？因为`android.intent.category.DEFAULT`是一种默认的category，在调用startActivity方法的时候会自动将这个category添加到Intent中。

## 指定多个category

每个Intent中只能指定一个action，但却能指定多个category。

最开始Intent只有一个默认的category。现在来增加一个。可以调用Intent中的`addCategory`方法来添加一个category。

修改FirstActivity中按钮的点击事件，代码如下所示：

```java
button1.setOnClickListener(new View.OnClickListener(){
    @Override
    public void onClick(View v){
        Intent intent = new Intent("com.example.activity.ACTION_START");
        intent.addCategory("com.example.activitytest.MY_CATEGORY");
        startActivity(intent);
    }
});
```

这里我们指定了一个自定义的category，值为`com.example.activitytest.MY_CATEGORY`。调用Intent中的`addCategory`方法来添加一个category。

这时，工作还没完毕，SecondActivity的`<intent-filter>`标签中并没有声明可以响应这个category，所以此时没有任何活动可以响应该Intent。现在我们在`<intent-filter>`中再添加一个category的声明，如下：

```xml
<activity android:name=".SecondActivity">
    <intent-filter>
        <action android:name="com.example.activitytest.ACTION_START"></action>
        <category android:name="android.intent.category.DEFAULT"></category>
        <category android:name="com.example.activitytest.MY_CATEGORY"></category>
    </intent-filter>
</activity>
```

## 更多用法

使用隐式Intent，我们不仅可以启动自己程序内的活动，还可以启动其他程序的活动，这使得Android多个应用程序之间的功能可以共享。比如你的应用程序需要展示一个网页，没必要自己去实现一个浏览器，只需要调用系统的浏览器来打开这个网页。

修改FirstActivity中按钮点击事件的代码，如下：

```java
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        /**
         * 这里首先指定了Intent的action是Intent.ACTION_VIEW，这是一个Android系统
         * 内置的动作，其常量值为android.intent.action.VIEW。然后通过Uri.parse()方法
         * 将一个网址字符串解析成一个Uri对象，再调用Intent的setData()方法将这个Uri对象传递进去
         */
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setData(Uri.parse("http://www.baidu.com"));
        startActivity(intent);
    }
});
```

与此对应，我们还可以在`<intent-filter>`标签中再配置一个`<data>`标签，用于更精确地指定当前活动能够响应什么类型的数据。`<data>`标签中主要可以配置以下内容。

* `android:scheme`用于指定数据的协议部分，如上例中的http部分。
* `android:host`用于指定数据的主机名部分，如上例中`www.baidu.com`部分。
* `android:port`用于指定数据的端口部分，一般紧随在主机名之后。
* `android:path`用于指定主机名和端口之后的部分，如一段网址中跟在域名之后的内容
* `android:mimeType`用于指定可以处理的数据类型，允许使用通配符的方式进行指定。

只有data标签中指定的内容和Intent中携带的Data完全一致时，当前活动才能够响应该Intent。一般在`<data>`标签中不会指定过多的内容，如上面浏览器示例中，其实只要指定`android:scheme`为http就可以响应所有的http协议的Intent了。

为了理解，建立ThirdActivity，布局文件起名为`third_layout`。

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android">
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    <Button>
        android:id="@+id/button_3"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Button 3"
    </Button>
</LinearLayout>
```

ThirdActivity.java中的代码保持不变就可以了。

在AndroidManifest.xml中修改ThirdActivity的注册信息：

```xml
<activity android:name=".ThirdActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW"></action>
        <category android:name="android.intent.category.DEFAULT"></category>
        <data android:scheme="http"></data>
    </intent-filter>
</activity>
```

我们在ThirdActivity的`<intent-filter>`中配置了当前活动能够响应的action是`Intent.ACTION_VIEW`的常量值，而category则毫无疑问指定了默认的category值，另外在`<data>`标签中我们通过`android:scheme`指定了数据的协议必须是http协议，这样ThirdActivity应该就和浏览器一样，能够响应一个打开网页的Intent了。运行一下程序试试，在FirstActivity的界面点击一下按钮，结果应该可以看到系统自动弹出一个列表，显示了目前能够响应这个Intent的所有程序。可响应的程序有两个：一是Broswer，会打开浏览器并显示网页；二是这个app名字，会启动ThirdActivity，但是实际上这个活动并没有加载、显示网页的功能。

# 向活动传递数据

## 向下一个活动传递

Intent不仅可以启动一个活动，还可以在启动活动的时候传递数据。

Intent中提供了一系列`putExtra`方法的重载，可以把我们想要传递的数据暂存在Intent中，启动了另一个活动后，只需要把这些数据再从Intent中取出就可以了。比如FirstActivity中有一个字符串，现在想把这个字符串传递到SecondActivity中，可以这样编写：

```java
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        String data = "Hello SecondActivity";
        Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
        intent.putExtra("extra_data", data);
        startActivity(intent);
    }
});
```

这里我们使用显示Intent的方式来启动SecondActivity，并通过`putExtra()`方法传递了一个字符串。注意这里putExtra方法接收两个参数，第一个参数是键，用于后面从Intent中取值，第二个参数才是真正要传递的数据。

然后在SecondActivity中将传递的数据取出，并打印出来，代码如下：

```java
public class SecondActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.second_layout);
        Intent intent = getIntent();
        String data = intent.getStringExtra("extra_data");
        Log.d("SecondActivity", data);
    }
}
```

首先可以通过`getlntent`方法获取到用于启动`SecondActivity`的`Intent`，然后调用`getStringExtra`方法，传入相应的键值，就可以得到传递的数据了。这里由于我们传递的是字符串，所以使用`getStringExtra`方法来获取传递的数据。如果传递的是整型数据，则使用`getlntExtra`方法；如果传递的是布尔型数据，则使用`getBooleanExtra`方法，以此类推。

重新运行程序，在FirstActivity的界面点击一下按钮会跳转到SecondActivity，査看logcat打印的信息。

```
507-24507/com.example.intenttest D/SecondActivity: Hello SecondActivity
```

## 向上一个传递

既然可以传递数据给下一个活动，那么能不能够返回数据给上一个活动呢？答案是肯定的。不过不同的是，返回上一个活动只需要按一下Back键就可以了，并没有一个用于启动活动的Intent来传递数据。通过查阅文档发现，Activity中还有一个`startActivityForResult()`方法也是用于启动活动的，但这个方法期望在**活动销毁的时候能够返回一个结果给上一个活动**。毫无疑问，这就是我们所需要的。

`startActivityForResult`方法接收两个参数，第一个参数还是Intent，第二个参数是**请求码**，用于在之后的回调中判断数据的来源。我们修改FirstActivity中按钮的点击事件，代码如下所示∶

```java
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
        startActivityForResult(intent, 1);
    }
});
```

这里我们使用了startActivityForResult方法来启动SecondActivity，请求码只要是一个唯一值就可以了，这里传入了1。接下来我们在SecondActivity中给按钮注册点击事件，并在点击事件中添加返回数据的逻辑，代码如下所示：

```java
public class SecondActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.second_layout);
        Button button2 = (Button) findViewById(R.id.button_2);
        button2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent();
                intent.putExtra("data_return", "Hello FirstActivity");
                setResult(RESULT_OK, intent);
                finish();
            }
        });
    }
}
```

可以看到，我们还是构建了一个Intent，只不过这个Intent仅仅是用于传递数据而已，它没有指定任何的“意图”。紧接着把要传递的数据存放在Intent中，然后调用了setResult方法。这个方法非常重要，是专门用于向上一个活动返回数据的。setResult方法接收两个参数，第一个参数用于向上一个活动返回处理结果，一般只使用RESULT_OK或RESULT_CANCELED这两个值，第二个参数则把带有数据的Intent传递回去，然后调用了finish方法来销毁当前活动。

由于我们是使用startActivityForResult方法来启动SecondActivity的，在SecondActivity被销毁之后会回调上一个活动的onActivityResult方法，因此我们需要在FirstActivity中重写这个方法来得到返回的数据，如下所示∶

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    switch (requestCode) {
        case 1:
            if(resultCode == RESULT_OK) {
                String returnedData = data.getStringExtra("data_return");
                Log.d("FirstActivity", returnedData);
            }
            break;
        default:
    }
}
```

onActivityResult方法带有三个参数，第一个参数requestCode，即我们在启动活动时传入的请求码。第二个参数resultCode，即我们在返回数据时传入的处理结果。第三个参数data，即携带着返回数据的Intent。由于在一个活动中有可能调用startActivityForResult方法去启动很多不同的活动，每一个活动返回的数据都会回调到onActivityResult这个方法中，因此我们首先要做的就是通过检查requestCode的值来判断数据来源。确定数据是从SecondActivity返回的之后，我们再通过resultCode的值来判断处理结果是否成功。最后从data中取值并打印出来，这样就完成了向上一个活动返回数据的工作。

重新运行程序，在FirstActivity的界面点击按钮会打开SecondActivity，然后在SecondActivity界面点击Button2按钮会回到FirstActivity，这时查看logcat的打印信息。

```
31184-31184/com.example.intenttest D/FirstActivity: Hello FirstActivity
```

但是可能会存在缺陷。如果用户在SecondActivity中并不是通过点击按钮，而是通过按下back键回到FirstActivity，这样数据就无法返回到上一个活动。可以在SecondActivity中重写onBackPressed方法来解决：

```java
@Override
public void onBackPressed() {
    Intent intent = new Intent();
    intent.putExtra("data_return", "Hello FirstActivity");
    setResult(RESULT_OK, intent);
    finish();
}
```

如此，当用户按下back键，就会去执行onBackPressed方法中的代码。

