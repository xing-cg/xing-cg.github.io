---
title: Android_应用初体验
categories:
  - - Android
tags: 
date: 2022/8/14
updated: 
comments: 
published:
---

# 介绍

本文将介绍如何构建简单的Android应用。首先，了解如何通过Android Studio创建`“Hello, World!”`项目并运行它。然后，为应用创建一个新界面，该界面会接受用户输入，并切换到应用中的一个新屏幕以显示用户输入内容。

开始之前，需要了解有关Android应用的两个基本概念：它们如何提供多个入口点，以及它们如何适应不同的设备。

## 应用提供多个入口点

Android应用都是将各种可单独调用的组件加以组合构建而成。例如，activity是提供界面(UI)的一种应用组件。

“主”activity在用户点按您的应用图标时启动。还可以将用户从其他位置（例如，从通知中，甚至从其他应用中）引导至某个activity。

其他组件（如`WorkManager`）可使应用能够在没有界面的情况下执行后台任务。

## 应用可适应不同的设备

Android允许您为不同的设备提供不同的资源。例如，您可以针对不同的屏幕尺寸创建不同的布局。系统会根据当前设备的屏幕尺寸确定要使用的布局。

如果应用的任何功能需要使用特定的硬件（例如摄像头），您可以在运行时查询该设备是否具有该硬件，如果没有，则停用相应的功能。您可以指定应用需要使用特定的硬件，这样，Google Play就不会允许在没有这些硬件的设备上安装应用。

# 界面布局

Android应用的界面（UI）以**布局**和**微件**的层次结构形式构建而成。布局是`ViewGroup`对象，即控制其子视图在屏幕上的放置方式的容器。微件是`View`对象，即按钮和文本框等界面组件。

![ViewGroup对象如何在布局中形成分支并包含View对象的图示](https://developer.android.com/static/images/viewgroup_2x.png)

Android提供了ViewGroup和View类的XML词汇表，因此界面的大部分内容都在XML文件中定义。

Android Studio的布局编辑器会在开发者拖放视图构建布局时自动编写XML代码，如何使用布局编辑器创建布局呢？

# 构建简单的界面 - 布局编辑器

左下方的Component Tree面板显示布局的视图层次结构。在本例中，根视图是`ConstraintLayout`，它仅包含一个`TextView`对象。

`ConstraintLayout`是一种布局，它根据同级视图和父布局的约束条件定义每个视图的位置，这样一来，使用扁平视图层次结构既可以创建简单布局，又可以创建复杂布局。这种布局无需嵌套布局。嵌套布局是布局内的布局，会增加绘制界面所需的时间。

例如可以声明以下布局。

* 视图A距离父布局顶部16dp
* 视图A距离父布局左侧16dp
* 视图B距离视图A右侧16dp
* 视图B与视图A顶部对齐

![ConstraintLayout内放有两个视图的图示](https://developer.android.com/static/training/basics/firstapp/images/constraint-example_2x.png)

## 添加文本框

按照下面的步骤添加文本框：

1. 首先，需要移除布局中已有的内容。在Componrnt Tree面板中点击TextView，然后按Delete键。
2. 在Palette面板中，点击Text以显示可用的文本控件。
3. 将Plain Text拖动到设计编辑器中，并将其放在靠近布局顶部的位置。这是一个接受纯文本输入的`EditText`微件。
4. 点击设计编辑器中的视图。现在，可以在每个角上看到调整视图大小的正方形手柄，并在每个边上看到圆形约束锚点。为了更好地控制，可能需要放大编辑器。
5. 点击并按住顶边上的锚点，将其向上拖动，直至其贴靠到布局顶部，然后将其释放。这是一个约束条件：它会将视图约束在已设置的默认外边距内。在本例中，将其设置为距离布局顶部16dp。
6. 使用相同的过程创建一个从视图左侧到布局左侧的约束条件。

结果如图。

![按照到父布局顶部和左侧的距离约束文本框](https://developer.android.com/static/images/training/basics/firstapp/building-ui-constrained-top-left.png)

## 添加按钮

1. 在Palette面板中，点击Buttons
2. 将Button微件拖到设计编辑器中，并将其放在靠近右侧的位置。
3. 创建一个从按钮左侧到文本框右侧的约束条件。
4. 如需按水平对齐约束视图，请创建一个文本基线之间的约束条件。为此，请右键点击按钮，然后选择`Show Baseline`。基线锚点显示在按钮内部。点击并按住此锚点，然后将其拖动到相邻文本框中显示的基线锚点上。（也可以右键点击文本框，从文本框内部发出基线锚点，拖动到相邻按钮中显示的基线锚点上。）

结果应如图所示。

![按照到文本框右侧的距离以及文本框基线来约束按钮](https://developer.android.com/static/images/training/basics/firstapp/building-ui-constrained-baseline.png)

> 还可以根据顶边或底边实现水平对齐。但按钮的图片周围有内边距，因此如果以这种方式对齐，那么它们看上去是没有对齐的。

## 更改界面字符串

若要预览界面，点击工具栏中的`Select Design Surface`，然后选择`Design`。文本输入和按钮标签应设置为默认值。

若要更改界面字符串，按以下步骤操作：

1. Project窗口，打开`app > res > values > strings.xml`。这是一个字符串资源文件，可以在此文件中指定所有界面字符串。可以利用该文件在一个位置管理所有界面字符串，使字符串的查找、更新和本地化变得更加容易。

2. 点击窗口顶部的`Open editor`。此时将打开`Translations Editor`，它提供了一个可以添加和修改默认字符串的简单界面。还有助于让所有已翻译的字符串井然有序。

3. 点击加号Add Key可以创建一个新字符串作为文本框的“提示文本”。此时会打开如图所示的窗口。
   ![用于添加新字符串的对话框](https://developer.android.com/static/training/basics/firstapp/images/add-string.png)

   在Add Key对话框中，完成以下步骤：

   1. 在`Key`字段中输入“`edit_message`”。
   2. 在`Default Value`字段中输入“`Enter a message`”。
   3. 点击`OK`。

4. 再添加一个名为"`button_send`"且值为"`Send`"的键。

现在，您可以为每个视图设置这些字符串。若要返回布局文件，请点击标签页栏中的`activity_main.xml`。然后，添加字符串，如下所示：

1. 点击布局中的文本框。如果右侧还未显示`Attributes`窗口，请点击右侧边栏上的 `Attributes`。
2. 找到`text`属性（当前设为“`Name`”）并删除相应的值。
3. 找到`hint`属性，然后点击文本框右侧的![img](https://developer.android.com/static/studio/images/buttons/pick-resource.png)(`Pick a Resource`)。在显示的对话框中，双击列表中的`edit_message`。
4. 点击布局中的按钮，找到其`text`属性（当前设为“`Button`”）。然后点击![img](https://developer.android.com/static/studio/images/buttons/pick-resource.png)(`Pick a Resource`)，并选择`button_send`。

## 让文本框大小可灵活调整

若要创建一个适应不同屏幕尺寸的布局，需要**让文本框拉伸**以填充去除按钮和外边距后**剩余的所有水平空间**。

继续操作之前，点击工具栏中的`Select Design Surface`，然后选择`Blueprint`。

若要让文本框大小可灵活调整，请按以下步骤操作：

1. 选择两个视图。若要执行此操作，请点击一个视图，在按住`Shift`键的同时点击另一个视图，然后右键点击任一视图并依次选择`Chains > Create Horizontal Chain`。布局随即显示出来，如图所示。
   ![选择Create Horizontal Chain后所得到的结果](https://developer.android.com/static/images/training/basics/firstapp/building-ui-horizontal-chain.png)

   链是两个或多个视图之间的双向约束条件，可让您采用一致的方式安排链接的视图。

2. 选择按钮并打开`Attributes`窗口。然后使用`Constraint Widget`将右外边距设为`16 dp`。

3. 点击文本框以查看其属性。然后，点击宽度指示器（途中红框框住的地方）两次，确保将其设置为锯齿状线(Match Constraints)，如图中的`标注1`所示。
   ![点击以将宽度更改为Match Constraints](https://developer.android.com/static/images/training/basics/firstapp/building-ui-match-constraints-2x.png)

   “Match constraints”表示宽度将延长以符合水平约束条件和外边距的定义。因此，文本框将拉伸以填充去除按钮和所有外边距后剩余的水平空间。

现在，布局已经完成，如图。

![文本框现在拉伸以填充剩余空间](https://developer.android.com/static/images/training/basics/firstapp/building-ui-constraint-fill.png)

最终布局XML应为以下内容：将其与**Code**标签页中看到的内容进行比较看看错了没。属性顺序不同没影响。
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.example.myfirstapp.MainActivity">

    <EditText
        android:id="@+id/editText"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginStart="16dp"
        android:layout_marginLeft="16dp"
        android:layout_marginTop="16dp"
        android:ems="10"
        android:hint="@string/edit_message"
        android:inputType="textPersonName"
        app:layout_constraintEnd_toStartOf="@+id/button"
        app:layout_constraintHorizontal_bias="0.5"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="16dp"
        android:layout_marginStart="16dp"
        android:text="@string/button_send"
        app:layout_constraintBaseline_toBaselineOf="@+id/editText"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.5"
        app:layout_constraintStart_toEndOf="@+id/editText" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

此时可以测试运行一下。

下面构建在点按按钮后启动的另一个activity。

# 启动另一个Activity

将向MainActivity添加一些代码，以便在用户点按Send按钮时启动一个显示消息的新activity。

## 响应“Send”按钮

向`MainActivity`类添加一个在用户点按Send按钮时调用的方法：

1. 在`app > java > com.example.myfirstapp > MainActivity`文件中，添加以下`sendMessage()`方法桩，这个方法将在某个事件发生后被回调。
   ```java
   public class MainActivity extends AppCompatActivity
   {
       // ...
       
       /* Called when the user taps the Send Button */
       public void sendMessage(View view)
       {
           // Do something in response to button
       }
   }
   ```

   >可能会看到一条错误，因为Android Studio无法解析用作方法参数的View类。若要清除错误，点击View声明，将光标置于其上，然后按Alt+Enter（在 Mac 上则按Option+Enter）进行快速修复。如果出现一个菜单，选择Import class。

2. 返回到activity_main.xml文件，并从该按钮调用此方法：

   1. 选择布局编辑器中的相应按钮。
   2. 在Attributes窗口中，找到`onClick`属性，并从其下拉列表中选择`sendMessage[MainActivity]`；或者在`activity_main.xml`中的`Button`标签中加入`android:onClick="sendMessage"`属性。

   现在，当用户点按该按钮时，系统将调用`sendMessage()`方法。

   >请注意此方法中提供的详细信息。系统需要这些信息来识别此方法是否与`android:onClick`属性兼容。具体来说，此方法具有以下特性：
   >
   >1. public。
   >2. 返回值为空，或在`Kotlin`中为隐式`Unit`。
   >3. `View`是唯一的参数。这是在第1步结束时点击的`View`对象。

3. 接下来，填写此方法，以读取文本字段的内容，并将该文本传递给另一个activity。

## 构建一个intent

`Intent`是在相互独立的组件（如两个`activity`）之间提供运行时绑定功能的对象。`Intent`表示应用执行某项操作的意图。可以使用`intent`执行多种任务，但在本例中，`intent`将用于启动另一个`activity`。

在`MainActivity`中，添加`EXTRA_MESSAGE`常量和`sendMessage()`代码，如下所示：
```java
public class MainActivity extends AppCompatActivity {
    public static final String EXTRA_MESSAGE = "com.example.myfirstapp.MESSAGE";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    /** Called when the user taps the Send button */
    public void sendMessage(View view) {
        Intent intent = new Intent(this, DisplayMessageActivity.class);
        EditText editText = (EditText) findViewById(R.id.editText);
        String message = editText.getText().toString();
        intent.putExtra(EXTRA_MESSAGE, message);
        startActivity(intent);
    }
}
```

`DisplayMessageActivity`仍有错误，但没有关系，下一部分中将修复错误。

`sendMessage()`将发生以下情况：

* `Intent`构造函数会获取两个参数：`Context`和`Class`。
  `Context`参数首先会被使用，因为`Activity`类是`Context`的子类，在构造activity时，则先构造context父模块。
  在本例中，系统将`Intent`传递到的应用组件的`class`参数是要启动的activity。
* `putExtra()`方法将`editText`的值添加到intent。`Intent`能够以称为`"extra"`的键值对形式携带数据。
  key是一个公共常量`EXTRA_MESSAGE`，因为下一个activity将使用该键检索文本值。为intent extra定义键时，最好使用应用的软件包名称作为前缀。这样可以确保这些键是独一无二的，这在应用需要与其他应用进行交互时很重要。
  value是message，即为editText的内容。将在下一个activity中获取。
* `startActivity()`方法将启动一个由`Intent`指定的`DisplayMessageActivity`实例，接下来需要创建该类。

# 创建第二个activity

1. 在Project窗口中，右键点击app文件夹，然后依次选择`New > Activity > Empty Activity`。
2. 在`Configure Activity`窗口中，输入“`DisplayMessageActivity`”作为`Activity Name`。将所有其他属性保留为默认设置，然后点击`Finish`。

Android Studio会自动执行下列三项操作：

* 创建`DisplayMessageActivity`文件。
* 创建`DisplayMessageActivity`文件对应的布局文件 `activity_display_message.xml`。
* 在`AndroidManifest.xml`中添加所需的`<activity>`元素。

如果您运行应用并点按第一个`activity`上的按钮，将启动第二个`activity`，但它为空。这是因为第二个`activity`使用模板提供的空布局。

## 添加文本视图

新activity包含一个空白布局文件。请按以下步骤操作，在显示消息的位置添加一个文本视图：

1. 打开`app > res > layout > activity_display_message.xml`文件。
2. 点击工具栏中的![img](https://developer.android.com/static/studio/images/buttons/layout-editor-autoconnect-on.png)`Enable Autoconnection to Parent`。系统将启用`Autoconnect`。
3. 在`Palette`面板中，点击`Text`，将`TextView`拖动到布局中，然后将其放置在靠近布局顶部中心的位置，使其贴靠到出现的垂直线上。`Autoconnect`将添加左侧和右侧约束条件，以便将该视图放置在水平中心位置。
4. 再创建一个从文本视图顶部到布局顶部的约束条件，使该视图如图1中所示。

或者，您可以对文本样式进行一些调整，方法是在`Attributes`窗口的`Common Attributes`面板中展开`textAppearance`，然后更改`textSize`和`textColor`等属性。

## 显示消息

在此步骤中，修改第二个activity以显示第一个activity传递的消息。

在`DisplayMessageActivity`中，将以下代码添加到`onCreate()`方法中：
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_display_message);
    
    // Get the Intent that started this activity and extract the string
    Intent intent = getIntent();
    String message = intent.getStringExtra(MainActivity.EXTRA_MESSAGE);

    // Capture the layout's TextView and set the string as its text
    TextView textView = findViewById(R.id.textView);
    textView.setText(message);
}
```

## 添加向上导航功能

应用中，不是主入口点的每个屏幕（所有不是主屏幕的屏幕）都必须提供导航功能，以便将用户引导至应用层次结构中的逻辑父级屏幕。为此，请在应用栏中添加向上按钮。

若要添加向上按钮，需要在`AndroidManifest.xml`文件中声明哪个activity是逻辑父级。打开`app > manifests > AndroidManifest.xml`文件，找到`DisplayMessageActivity`的`<activity>`标记，然后将其替换为以下代码：

```xml
<activity android:name=".DisplayMessageActivity"
          android:parentActivityName=".MainActivity">
    <!-- The meta-data tag is required if you support API level 15 and lower -->
    <meta-data
        android:name="android.support.PARENT_ACTIVITY"
        android:value=".MainActivity" />
</activity>
```

Android系统现在会自动向应用栏添加向上按钮。