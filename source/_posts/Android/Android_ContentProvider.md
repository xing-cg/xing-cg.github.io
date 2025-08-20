---
title: Android_ContentProvider
categories:
  - - Android
tags: 
date: 2022/8/5
updated: 
comments: 
published:
---

# 为了跨程序共享数据

对于Android数据持久化技术，包括文件存储、SharedPreferences存储以及数据库存储，这些持久化技术所保存的数据都只能在当前应用程序中访问。虽然文件和SharedPreferences存储中提供了 `MODE_WORLD_READABLE`和`MODE_WORLD_WRITEABLE`这两种操作模式，用于供给其他的应用程序访问当前应用的数据，但这两种模式在Android 4.2版本中都已被废弃了。因为Android官方已经不再推荐使用这种方式来实现跨程序数据共享的功能，而是应该使用更加安全可靠的**内容提供器(ContentProvider)**技术。

为什么要将我们程序中的数据共享给其他程序呢？当然，这个是要视情况而定的，比如说账号和密码这样的隐私数据显然是不能共享给其他程序的，不过一些可以让其他程序进行二次开发的基础性数据，我们还是可以选择将其共享的。例如系统的电话簿程序，它的数据库中保存了很多的联系人信息，如果这些数据都不允许第三方的程序进行访问的话，恐怕很多应用的功能都要大打折扣了。除了电话簿之外，还有短信、媒体库等程序都实现了跨程序数据共享的功能，而使用的技术当然就是内容提供器了，下面就来对这一技术进行深入的探讨。

# 运行时权限

内容提供器（Content Provider）主要用于在不同的应用程序之间实现数据共享的功能，它提供了一套完整的机制，允许一个程序访问另一个程序中的数据，同时还能保证被访数据的安全性。

目前，使用内容提供器是Android实现跨程序共享数据的标准方式。

不同于文件存储和SharedPreferences存储中的两种全局可读写操作模式，内容提供器可以选择只对哪一部分数据进行共享，从而保证程序中的隐私数据不会有泄漏的风险。

在正式开始学习内容提供器之前，需要先掌握另外一个非常重要的知识——Android运行时权限，因为内容提供器会使用到运行时权限的功能。当然不光是内容提供器用到。

Android的**权限机制**从第一个版本开始就已经存在了。但其实之前Android的权限机制在保护用户安全和隐私等方面起到的作用比较有限，尤其是一些大家都离不开的常用软件，非常容易“店大欺客”。为此，Android开发团队在Android 6.0系统中引用了**运行时权限**这个功能，从而更好地保护了用户的安全和隐私。

## Android权限机制详解

首先来看一下过去Android的权限机制是什么样的。比如为了要访问系统的网络状态以及监听开机广播，于是在`AndroidManifest.xml`文件中添加了这样两句权限声明：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.broadcasttest">
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED">
    </uses-permission>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE">
    </uses-permission>
    ...
</manifest>
```

因为访问系统的网络状态以及监听开机广播涉及了用户设备的安全性，因此必须在AndroidManifest.xml中加入权限声明，否则程序就会崩溃。

那么现在问题来了，加入了这两句权限声明后，对于用户来说到底有什么影响呢？为什么这样就可以保护用户设备的安全性了呢?

其实用户主要在以下两个方面得到了保护，一方面，如果用户在低于6.0系统的设备上安装该程序，会在安装界面给出提醒——“此应用将获得以下权限：查看网络连接、开机启动”。这样用户就可以清楚地知晓该程序一共申请了哪些权限，从而决定是否要安装这个程序。

另一方面，用户可以随时在应用程序管理界面查看任意一个程序的权限申请情况。这样该程序申请的所有权限就尽收眼底，什么都瞒不过用户的眼睛，以此保证应用程序不会出现各种滥用权限的情况。

这种权限机制的设计思路其实非常简单，就是用户如果认可你所申请的权限，那么就会安装程序，如果不认可申请的权限，那么拒绝安装就可以了。

但是理想是美好的，现实却很残酷，因为很多我们所离不开的常用软件普遍存在着滥用权限的情况，不管到底用不用得到，反正先把权限申请了再说。比如说微信所申请的权限。

Android开发团队当然也意识到了这个问题，于是在6.0系统中加入了运行时权限功能。也就是说，用户不需要在安装软件的时候一次性授权所有申请的权限，而是可以在软件的使用过程中再对某一项权限申请进行授权。比如说一款相机应用在运行时申请了地理位置定位权限，就算我拒绝了这个权限，但是我应该仍然可以使用这个应用的其他功能，而不是像之前那样直接无法安装它。

当然，并不是所有权限都需要在运行时申请，对于用户来说，不停地授权也很烦琐。Android现在将所有的权限归成了两类，一类是普通权限，一类是危险权限。准确地讲，其实还有第三类特殊权限，不过这种权限使用得很少，因此不在讨论范围之内。普通权限指的是那些不会直接威胁到用户的安全和隐私的权限，对于这部分权限申请，系统会自动帮我们进行授权，而不需要用户再去手动操作了。危险权限则表示那些可能会触及用户隐私或者对设备安全性造成影响的权限，如获取设备联系人信息、定位设备的地理位置等，对于这部分权限申请，必须要由用户手动点击授权才可以，否则程序就无法使用相应的功能。

但是Android中有一共有上百种权限，怎么从中区分哪些是普通权限，哪些是危险权限？其实不难，危险权限很少，除了危险权限之外，剩余的就是普通权限。下表为到Android 10为止所有的危险权限。

| 权限组名             | 权限名                                                       |
| -------------------- | ------------------------------------------------------------ |
| CALENDAR             | `READ_CALENDAR`<br />`WRITE_CALENDAR`                        |
| CALL_LOG             | `READ_CALL_LOG`<br />`WRITE_CALL_LOG`<br />`PROCESS_OUTGOING_CALLS` |
| CAMERA               | CAMERA                                                       |
| CONTACTS             | `READ_CONTACTS`<br />`WRITE_CONTACTS`<br />`GET_ACCOUNTS`    |
| LOCATION             | `ACCESS_FINE_LOCATION`<br />`ACCESS_COARSE_LOCATION`<br />`ACCESS_BACKGROUND_LOCATION` |
| MICROPHONE           | `RECORD_AUDIO`                                               |
| PHONE                | `READ_PHONE_STATE`<br />`READ_PHONE_NUMBERS`<br />`CALL_PHONE`<br />`ANSWER_PHONE_CALLS`<br />`ADD_VOICEMAIL`<br />`USE_SIP`<br />`ACCEPT_HANDOVER` |
| SENSORS              | `BODY_SENSORS`                                               |
| ACTIVITY_RECOGNITION | `ACTIVITY_RECOGNITION`                                       |
| SMS                  | `SEND_SMS`<br />`RECEIVE_SMS`<br />`READ_SMS`<br />`RECEIVE_WAP_PUSH`<br />`RECEIVE_MMS` |
| STORAGE              | `READ_EXTERNAL_STORAGE`<br />`WRITE_EXTERNAL_STORAGE`<br />`ACCESS_MEDIA_LOCATION` |

每当要使用一个权限时，可以先到这张表中来查一下，如果是属于这张表中的权限，那么就需要进行运行时权限处理，如果不在这张表中，那么只需要在AndroidManifest.xml文件中添加一下权限声明就可以了。

另外注意一下，表格中每个危险权限都属于一个权限组，我们在进行运行时权限处理时使用的是权限名，但是用户一旦同意授权了，那么该权限所对应的权限组中所有的其他权限也会同时被授权。但是谨记，不要基于此规则来实现任何功能逻辑，因为Android系统随时可能调整权限的分组。

## 在程序运行时申请权限

首先新建一个RuntimePermissionTest项目。先以`CALL_PHONE`这个权限作为示例。

`CALL_PHONE`这个权限是编写拨打电话功能的时候需要声明的，因为拨打电话会涉及用户手机的资费问题，因而被列为了危险权限。在Android 6.0系统出现之前，拨打电话功能的实现非常简单，修改`activity_main.xml`布局文件，如下所示：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <Button
        android:id="@+id/make_call"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Make Call">
    </Button>
</LinearLayout>
```

修改MainActivity中的代码，如下：

Java版：

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button makeCall = (Button) findViewById(R.id.make_call);
        makeCall.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                try {
                    Intent intent = new Intent(Intent.ACTION_CALL);
                    intent.setData(Uri.parse("tel:10086"));
                    startActivity(intent);
                } catch (SecurityException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

kotlin版：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        makeCall.setOnClickListener {
            try {
                val intent = Intent(Intent.ACTION_CALL)
                intent.data = Uri.parse("tel:10086")
                startActivity(intent)
            } catch (e: SecurityException) {
                e.printStackTrace()
            }
        }
    }
}
```

可以看到，在按钮的点击事件中，构建了一个隐式Intent，Intent的action指定为`Intent.ACTION_CALL`，这是一个系统内置的打电话的动作，然后在data部分指定了协议是tel，号码是10086。要注意区分，`Intent.ACTION_DIAL`表示打开拨号界面，这个是不需要声明权限的，而`Intent.ACTION_CALL`则可以直接拨打电话，因此必须声明权限。

另外为了防止程序崩溃，将所有操作都放在了异常捕获代码块当中。那么接下来修改`AndroidManifest.xml`文件，在其中声明如下权限：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.filetest">
    <uses-permission android:name="android.permission.CALL_PHONE">
    </uses-permission>
    ...
</manifest>
```

这样拨打电话的功能成功实现了，在低于Android 6.0系统的手机上都是可以正常运行的，但是如果在6.0或者更高版本系统的手机上运行，点击Make Call按钮就没有任何效果，这时观察logcat中的打印日志，会看到错误信息中提醒我们“Permission Denial”，可以看出，这是权限被禁止所导致，因为Android 6.0及以上系统在使用危险权限时必须进行运行时权限处理。

修改MainActivity中的代码，如下：

Java代码：

```java
public class MainActivity extends AppCompatActivity {
    @Override

    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button makeCall = (Button) findViewById(R.id.makeCall);
        makeCall.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                if(ContextCompat.checkSelfPermission(MainActivity.this,
                        Manifest.permission.CALL_PHONE) != PackageManager.PERMISSION_GRANTED) {
                    ActivityCompat.requestPermissions(MainActivity.this, new String[]{ Manifest.permission.CALL_PHONE }, 1);
                } else {
                    call();
                }
            }
        });
    }
    private void call() {
        try {
            Intent intent = new Intent(Intent.ACTION_CALL);
            intent.setData(Uri.parse("tel:10086"));
            startActivity(intent);
        } catch (SecurityException e) {
            e.printStackTrace();
        }
    }
    // 用户选择同意或拒绝后，会回调到此方法中，授权的结果会封装在grantResults参数中。
    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions,
                                          int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        switch (requestCode) {
            case 1:
                if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    call();
                } else {
                    Toast.makeText(this, "You denied the permission", Toast.LENGTH_SHORT).show();
                }
                break;
            default:

        }
    }
}
```

kotlin版：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        makeCall.setOnClickListener {
            if(ContextCompat.checkSelfPermission(this,
                    Manifest.permission.CALL_PHONE) != PackageManager.PERMISSION_GRANTED) {
                ActivityCompat.requestPermissions(this,
                arrayOf(Manifest.permission.CALL_PHONE), 1)
            }
            else {
                call()
            }
        }
    }
    override fun onRequestPermissionsResult(requestCode: Int,
                                            permissions: Array<String>,
                                            grantResults； IntArray) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        when (requestCode) {
            1 -> {
                if (grantResults.isNotEmpty() &&
                    grantResults[0] == PackManager.PERMISSION_GRANTED) {
                    call()
                } else {
                    Toast.makeText(this, "You denied the permission",
                                   Toast.LENGTH_SHORT).show()
                }
            }
        }
    }
    private fun call() {
        try {
            val intent = Intent(Intent.ACTION_CALL)
            intent.data = Uri.parse("tel:10086")
            startActivity(intent)
        } catch (e: SecurityException) {
            e.printStackTrace()
        }
    }
}
```

上面的代码将运行时权限的完整流程都覆盖了，下面来具体解析一下。说白了，运行时权限的核心就是在程序运行过程中由用户授权去执行某些危险操作，程序是不可以擅自做主去执行这些危险操作的。因此，第一步就是要先判断用户是不是已经给过授权了，借助的是ContextCompat.checkSelfPermission方法。checkSelfPermission方法接收两个参数，第一个参数是Context，第二个参数是具体的权限名，比如打电话的权限名就是`Manifest.permission.CALL_PHONE`，然后我们使用方法的返回值和`PackageManager.PERMISSION_GRANTED`做比较，相等就说明用户已经授权，不等就表示用户没有授权。

如果已经授权的话就简单了，直接去执行拨打电话的逻辑操作就可以了，这里我们把拨打电话的逻辑封装到了call方法当中。如果没有授权的话，则需要调用`ActivityCompat.requestPermissions`方法来向用户申请授权，`requestPermissions`方法接收3个参数，第一个参数要求是Activity的实例，第二个参数是一个String数组，我们把要申请的权限名放在数组中即可，第三个参数是请求码，只要是唯一值就可以了，这里传入了1。

调用完了`requestPermissions`方法之后，系统会弹出一个权限申请的对话框，然后用户可以选择同意或拒绝我们的权限申请，不论是哪种结果，最终都会回调到`onRequestPermissionsResult`方法中，而授权的结果则会封装在`grantResults`参数当中。这里我们只需要判断一下最后的授权结果，如果用户同意的话就调用call方法来拨打电话，如果用户拒绝的话我们只能放弃操作，并且弹出一条失败提示。

现在重新运行一下程序，并点击Make Call按钮。由于用户还没有授权过我们拨打电话权限，因此第一次运行会弹出一个权限申请的对话框，用户可以选择同意或者拒绝。

如果选择同意，就成功进入到拨打电话界面了，由于用户已经完成了授权操作，之后再点击Make Call按钮就不会再弹出权限申请对话框了，而是可以直接拨打电话。那可能你会担心，万一以后我又后悔了怎么办？没有关系，用户随时都可以将授予程序的危险权限进行关闭，进入`Settings/Apps/RuntimePermissionTest/Permissions`。在这里我们就可以对任何授予过的危险权限进行关闭了。

# 访问其他程序中的数据

内容提供器的用法一般有两种，一种是使用现有的内容提供器来读取和操作相应程序中的数据，另一种是创建自己的内容提供器给程序的数据提供外部访问接口。

如果一个应用程序通过内容提供器对其数据提供了外部访问接口，那么任何其他的应用程序就都可以对这部分数据进行访问。Android系统中自带的电话簿、短信、媒体库等程序都提供了类似的访问接口，这就使得第三方应用程序可以充分地利用这部分数据来实现更好的功能。

## ContentResolver的基本用法

对于每一个应用程序来说，如果想要访问内容提供器中共享的数据，就一定要借助`ContentResolver`类，可以通过Context中的getContentResolver方法获取到该类的实例。

ContentResolver中提供了一系列的方法用于对数据进行CRUD操作，其中insert方法用于添加数据，update方法用于更新数据，delete方法用于删除数据，query方法用于查询数据。SQLiteDatabase中也是使用这几个方法来进行CRUD操作的，只不过它们在方法参数上稍微有一些区别。

不同于SQLiteDatabase，ContentResolver中的增删改查方法不接收表名参数，而是使用一个Uri参数代替，这个参数被称为**内容URI**。内容URI给内容提供器中的数据建立了唯一标识符，它主要由两部分组成：authority和path。（其实还有一个固定的部分，是头部协议声明）

authority是用于对不同的应用程序做区分的，一般为了避免冲突，都会采用程序包名的方式来进行命名。比如某个程序的包名是`com.example.app`，那么该程序对应的`authority`就可以命名为`com.example.app.provider`。

path则是用于对**同一应用程序中不同的表**做区分的，通常都会添加到authority的后面。比如某个程序的数据库里存在两张表：tablel和table2，这时就可以将path分别命名为`/tablel`和`/table2`，然后把`authority`和`path`进行组合，内容`URI`就变成了`com.example.app.provider/table1`和`com.example.app.provider/table2`。不过，目前还很难辨认出这两个字符串就是两个内容URI，我们还需要**在字符串的头部加上协议声明**。因此，内容URI最标准的格式写法如下：

`content://com.example.app.provider/tablel`，`content://com.example.app.provider/table2`

发现，内容URI可以非常清楚地表达出我们想要访问哪个程序中哪张表里的数据。也正是因此，ContentResolver中的增删改查方法才都接收Uri对象作为参数，因为如果使用表名的话，系统将无法得知我们期望访问的是哪个应用程序里的表。

在得到了内容URI字符串之后，我们还需要将它**解析成Uri对象**才可以作为参数传入。解析的方法也相当简单，只需要调用`Uri.parse`方法，代码如下所示：`Uri uri = Uri.parse("content://com.example.app.provider/table1")`。现在我们就可以使用这个Uri对象来查询table1表中的数据了，代码如下所示：

```java
Cursor cursor = getContentResolver().query(
    uri,
    projection,
    selection,
    selectionArgs,
    sortOrder);
```

这些参数和SQLiteDatabase中query方法里的参数很像，但总体来说要简单一些，毕竟这是在访问其他程序中的数据，没必要构建过于复杂的查询语句。

query方法的参数说明：

| query方法参数 | 对应SQL部分               | 描述                             |
| ------------- | ------------------------- | -------------------------------- |
| uri           | from table_name           | 指定查询某个应用程序下的某一张表 |
| projection    | select column1, column2   | 指定查询的列名                   |
| selection     | where column = value      | 指定where的约束条件              |
| selectionArgs | -                         | 为where中的占位符提供具体的值    |
| sortOrder     | order by column1, column2 | 指定查询结果的排序方式           |

查询完成后返回的仍然是一个Cursor对象，这时我们就可以将数据从Cursor对象中逐个读取出来了。读取的思路仍然是通过移动游标的位置来遍历Cursor的所有行，然后再取出每一行中相应列的数据，代码如下所示：

```java
if(cursor != null) {
    while (cursor.moveToNext()) {
        String column1 = cursor.getString(cursor.getColumnIndex("column1"));
        int column2 = cursor.getInt(cursor.getColumnIndex("column2"));
    }
    cursor.close();
}
```

向table1表添加数据，代码如下：

```java
ContentValues values = new ContentValues();
values.put("column1", "text");
values.put("column2", 1);
getContentResolver().insert(uri, values);
```

可以看到，仍然是将待添加的数据组装到ContentValues中，然后调用ContentResolver的insert方法，将Uri和ContentValues作为参数传入即可。

现在如果我们想要更新这条新添加的数据，把column1的值清空，可以借助ContentResolver的update方法实现。代码如下所示：

```java
ContentValues values = new ContentValues();
values.put("column1", "");
getContentResolver().update(uri, values, "column1 = ? and column2 = ?",
                            new String[] {"text", "1"});
```

注意上述代码使用了selection和selectionArgs参数来对想要更新的数据进行约束，以防止所有的行都会受影响。最后，可以调用ContentResolver的delete方法将这条数据删除掉，代码如下所示：

```java
getContentResolver().delete(uri, "column2 = ?", new String[] {"1"});
```

到这里为止，我们就把ContentResolver中的增删改查方法全部学完了。这些知识其实和SQLiteDatabase很类似，需要特别注意的只有uri这个参数而已。那么接下来，我们就利用目前所学的知识，看一看如何读取系统电话簿中的联系人信息。

## 示例：读取电话簿联系人

新建一个ContactsTest项目。

`activity_main.xml`：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <ListView
        android:id="@+id/contacts_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
    </ListView>
</LinearLayout>
```

MainActivity：

```java
public class MainActivity extends AppCompatActivity {
    ArrayAdapter<String> adpter;
    List<String> contactsList = new ArrayList<>();
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // 首先获取ListView控件的实例
        ListView contactsView = (ListView) findViewById(R.id.contacts_view);
        adapter = new ArrayAdapter<String>(this, android.R.layout.simple_list_item_1, contactsList);
        // 设置配置器
        contactsView.setAdapter(adapter);
        
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_CONTACTS) != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.READ_CONTACTS}, 1);          
        } else {
            readContacts();
        }
    }
    
    private void readContacts() {
        Cursor cursor = null;
        try {
            // 查询联系人数据
            // ContactsContract.CommonDataKinds.Phone类已经做好了封装，提供了一个CONTENT_URI常量
            cursor = getContentResolver().query(ContactsContract.CommonDataKinds.Phone.CONTENT_URI, null, null, null, null);
            if(cursor != null) {
                while (cursor.moveToNext()) {
                    // 获取联系人姓名
                    // DISPLAY_NAME 对应联系人名字这一列
                    String displayName = cursor.getString(cursor.getColumnIndex(ContactContract.CommonDataKinds.Phone.DISPLAY_NAME));
                    // 获取联系人手机号
                    // NUMBER 对应联系人手机号这一列
                    String number = cursor.getString(cursor.getColumnIndex(ContactsContract.CommonDataKinds.Phone.NUMBER));
                    // 添加到ListView的数据源List容器中
                    contactsList.add(displayName + "\n" + number);
                }
                // 通知刷新一下ListView
                adapter.notifyDataSetChanged();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (cursor != null) {
                // 千万不要忘记关闭Cursor对象
                cursor.close();
            }
        }
    }
    @Override
    public void onRequestPermissionResult(int requestCode, String[] permissions, int[] grantResults) {
        switch (requestCode) {
            case 1:
                if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    readContacts();
                } else {
                    Toast.makeText(this, "You denied the permission", Toast.LENGTH_SHORT).show();
                }
                break;
            default:
        }
    }
}
```

读取系统联系人的权限千万不要忘记在AndroidManifest.xml中声明

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.contactstest">
    <uses-permission android:name="android.permission.READ_CONTACTS" />
    ...
</manifest>
```

# 创建自己的内容提供器

在自己的程序中访问其他应用程序的数据，总体来说思路还是非常简单的，只需要获取到该应用程序的内容URI，然后借助ContentResolver进行CRUD操作就可以了。可是那些提供外部访问接口的应用程序都是如何实现这种功能的呢？它们又是怎样保证数据的安全性使得隐私数据不会泄漏出去？

## 创建内容提供器的步骤

前面已经提到过，如果想要实现跨程序共享数据的功能，官方推荐的方式是使用内容提供器，可以通过新建一个类去继承ContentProvider的方式来创建一个自己的内容提供器。ContentProvider类中有6个抽象方法，我们在使用子类继承它的时候，需要将这6个方法全部重写。新建MyProvider继承自ContentProvider，代码如下所示：

```java
public class MyProvider extends ContentProvider {
    @Override
    public boolean onCreate() {
        return false;
    }
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        return null;
    }
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        return null;
    }
    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        return 0;
    }
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        return 0;
    }
    @Override
    public String getType(Uri uri) {
        return null;
    }
}
```

1. onCreate()
   1. 初始化内容提供器的时候调用。通常会在这里完成对数据库的创建和升级等操作，返回true表示内容提供器初始化成功，返回false则表示失败。注意，只有当存在ContentResolver尝试访问我们程序中的数据时，内容提供器才会被初始化。
2. query()
   1. 从内容提供器中查询数据。使用uri参数来确定查询哪张表，projection参数用于确定查询哪些列，selection和selectionArgs参数用于约束查询哪些行，sortOrder参数用于对结果进行排序，查询的结果存放在Cursor对象中返回。
3. insert()
   1. 向内容提供器中添加一条数据。使用uri参数来确定要添加到的表，待添加的数据保存在values参数中。添加完成后，返回一个用于表示这条新记录的URI。

4. update()
   1. 更新内容提供器中已有的数据。使用uri参数来确定更新哪一张表中的数据，新数据保存在values参数中，selection和selectionArgs参数用于约束更新哪些行，受影响的行数将作为返回值返回。

5. delete()
   1. 从内容提供器中删除数据。使用uri参数来确定删除哪一张表中的数据，selection和selectionArgs 参数用于约束删除哪些行，被删除的行数将作为返回值返回。

6. getType()
   1. 根据传入的内容URI来返回相应的MIME类型。


可以看到，几乎每一个方法都会带有Uri这个参数，这个参数也正是调用ContentResolver的增删改查方法时传递过来的。而现在，我们需要对传入的Uri参数进行解析，从中分析出调用方期望访问的表和数据。回顾一下，一个标准的内容URI写法是这样的：`content://com.example.app.provider/table1`。这就表示调用方期望访问的是`com.example.app`这个应用的table1表中的数据。除此之外，我们还可以在这个内容URI的后面加上一个`id`，如下所示：`content://com.example.app.provider/tablel/1`。这就表示调用方期望访问的是`com.example.app`这个应用的`table1`表中id为1的数据。内容URI的格式主要就只有以上两种，以路径结尾就表示期望访问该表中所有的数据，以id结尾就表示期望访问该表中拥有相应id的数据。我们可以使用通配符的方式来分别匹配这两种格式的内容URI，规则如下：

* `*`，表示匹配任意长度的任意字符；
* `#`，表示匹配任意长度的数字。

所以，一个能够匹配任意表的内容URI格式就可以写成：`content://com.example.app.provider/*`，而一个能够匹配`table1`表中任意一行数据的内容URI格式就可以写成：`content://com.example.app.provider/table1/#`。

接着，我们再借助UriMatcher这个类就可以轻松地实现匹配内容URI的功能。UriMatcher中提供了一个addURI方法，这个方法接收3个参数，可以分别把authority、path和一个自定义代号传进去。这样，当调用UriMatcher的match方法时，就可以将一个Uri对象传入，返回值是某个能够匹配这个Uri对象所对应的自定义代号，利用这个代号，就可以判断出调用方期望访问的是哪张表中的数据了。修改`MyProvider`中的代码，如下所示：

```java
public class MyProvider extends ContenProvider {
    public static final int TABLE1_DIR = 0;
    public static final int TABLE1_ITEM = 1;
    public static final int TABLE2_DIR = 2;
    public static final int TABLE2_ITEM = 3;
    private static UriMatcher uriMatcher;
    static {
        uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        uriMatcher.addURI("com.example.app.provider", "table1", TABLE1_DIR);
        uriMatcher.addURI("com.example.app.provider ", "table1/#", TABLE1_ITEM);
        uriMatcher.addURI("com.example.app.provider ", "table2", TABLE2_DIR);
        uriMatcher.addURI("com.example.app.provider ", "table2/#", TABLE2_ITEM);
    }
    ...
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        switch (uriMatcher.match(uri)) {
            case TABLE1_DIR:
                break;
            case TABLE1_ITEM:
                break;
            case TABLE2_DIR:
                break;
            case TABLE2_ITEM:
                break;
            default:
                break;
        }
    }
    ...
}
```

可以看到，MyProvider中新增了4个整型常量，其中`TABLE1_DIR`表示访问`table1`表中的所有数据，`TABLE1_ITEM`表示访问`table1`表中的单条数据，`TABLE2_DIR`表示访问`table2`表中的所有数据，`TABLE2_ITEM`表示访问`table2`表中的单条数据。接着在静态代码块里我们创建了UriMatcher的实例，并调用addURI方法，将期望匹配的内容URI格式传递进去，注意这里传入的路径参数是可以使用通配符的。然后当query方法被调用的时候，就会通过UriMatcher的match方法对传入的Uri对象进行匹配，如果发现UriMatcher中某个内容URI格式成功匹配了该Uri对象，则会返回相应的自定义代号，然后我们就可以判断出调用方期望访问的到底是什么数据了。

上述代码只是以query方法为例做了个示范，其实insert、update、delete这几个方法的实现也是差不多的，它们都会携带Uri这个参数，然后同样利用UriMatcher的match方法判断出调用方期望访向的是哪张表，再对该表中的数据进行相应的操作就可以了。

除此之外，还有一个方法会比较陌生，即getType方法。它是所有的内容提供器都必须提供的一个方法，用于获取Uri对象所对应的MIME类型。一个内容URI所对应的MIME字符串主要由3部分组成，Android对这3个部分做了如下格式规定：

1. 必须以vnd开头。
2. 如果内容URI以路径结尾，则后接`android.cursor.dir/`，如果内容URI以id结尾，则后接 `android.cursor.item/`。
3. 最后接上`vnd.<authority>.<path>`。

所以，对于`content://com.example.app.provider/table1`这个内容URI，它所对应的MIME类型就可以写成：`vnd.android.cursor.dir/vnd.com.example.app.provider.table1`。对于`content://com.example.app.provider/table1/1`这个内容URI，它所对应的MIME类型就可以写成：`vnd.android.cursor.item/vnd.com.example.app.provider.table1`。

现在可以继续完善MyProvider中的内容了，这次来实现`getType()`方法中的逻辑，代码如下：

```java
public class MyProvider extends ContentProvider {
    ...
    @Override
    public String getType(Uri uri) {
        switch (uriMatcher.match(uri)) {
            case TABLE1_DIR:
                return "vnd.android.cursor.dir/vnd.com.example.app.provider.table1";
            case TABLE1_ITEM:
                return "vnd.android.cursor.item/vnd.com.example.app.provider.table1";
            case TABLE2_DIR:
                return "vnd.android.cursor.dir/vnd.com.example.app.provider.table2";
            case TABLE2_ITEM:
                return "vnd.android.cursor.item/vnd.com.example.app.provider.table2";
            default:
                break;
        }
        return null;
    }
}
```

到这里，一个完整的内容提供器就创建完成了，现在任何一个应用程序都可以使用ContentResolver来访问我们程序中的数据。那么前面所提到的，如何才能保证隐私数据不会泄漏出去呢？其实多亏了内容提供器的良好机制，这个问题在不知不觉中已经被解决了。因为**所有的CRUD操作都一定要匹配到相应的内容URI格式才能进行**，而我们当然不可能向UriMatcher中添加隐私数据的URI，所以这部分数据根本无法被外部程序访问到，安全问题也就不存在了。下面就来实战一下，真正体验一回跨程序数据共享的功能。

## 实现跨程序数据共享

简单起见，还是在DatabaseTest项目的基础上继续开发，通过内容提供器来给它加入外部访问接口。打开DatabaseTest项目，首先将MyDatabaseHelper中使用Toast弹出创建数据库成功的提示去除掉，因为跨程序访问时不能直接使用Toast。接下来，创建一个内容提供器，右击`com.example.databasetest`包，new，other，Content Provider。可以看到，这里我们将内容提供器命名为`DatabaseProvider.authority`指定为`com.example.databasetest.provider`，`Exported`属性表示是否允许外部程序访问我们的内容提供器，`Enabled`属性表示是否启用这个内容提供器。将两个属性都勾中，点击Finish完成创建。

接着修改DatabaseProvider中的代码，如下：

```java
public class DatabaseProvider extends ContentProvider {
    public static final int BOOK_DIR = 0;
    public static final int BOOK_ITEM = 1;
    public static final int CATEGORY_DIR = 2;
    public static final int CATEGORY_ITEM = 3;
    public static final String AUTHORITY = "com.example.sqlitetest.provider";
    private static UriMatcher uriMatcher;
    private MyDatabaseHelper dbHelper;
    static {
        uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        uriMatcher.addURI(AUTHORITY, "book", BOOK_DIR);
        uriMatcher.addURI(AUTHORITY, "book/#", BOOK_ITEM);
        uriMatcher.addURI(AUTHORITY, "category", CATEGORY_DIR);
        uriMatcher.addURI(AUTHORITY, "category/#", CATEGORY_ITEM);
    }
    @Override
    public boolean onCreate() {
        dbHelper = new MyDatabaseHelper(getContext(), "Bookstore.db", null, 2);
        return true;
    }
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        // 查询数据
        SQLiteDatabase db = dbHelper.getReadableDatabase();
        Cursor cursor = null;
        switch (uriMatcher.match(uri)) {
            case BOOK_DIR:
                cursor = db.query("Book", projection, selection, selectionArgs, null, null, sortOrder);
                break;
            case BOOK_ITEM:
                String bookId = uri.getPathSegments().get(1);
                cursor = db.query("Book", projection, "id = ?", new String[] {bookId}, null, null, sortOrder);
                break;
            case CATEGORY_DIR:
                cursor = db.query("Category", projection, selection, selectionArgs, null, null, sortOrder);
                break;
            case CATEGORY_ITEM:
                String categoryId = uri.getPathSegments().get(1);
                cursor = db.query("Category", projection, "id = ?", new String[] {categoryId}, null, null, sortOrder);
                break;
            default:
                break;
        }
        return cursor;
    }
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        // 添加数据
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        Uri uriReturn = null;
        switch (uriMatcher.match(uri)) {
            case BOOK_DIR:
            case BOOK_ITEM:
                long newBookId = db.insert("Book", null, values);
                uriReturn = Uri.parse("content://" + AUTHORITY + "/book/" + newBookId);
                break;
            case CATEGORY_DIR:
            case CATEGORY_ITEM:
                long newCategoryId = db.insert("Category", null, values);
                uriReturn = Uri.parse("content://" + AUTHORITY + "/category/" + newCategoryId);
                break;
            default:
                break;
        }
        return uriReturn;
    }
    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        // 更新数据
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        int updateRows = 0;
        switch (uriMatcher.match(uri)) {
            case BOOK_DIR:
                updateRows = db.update("Book", values, selection, selectionArgs);
                break;
            case BOOK_ITEM:
                String bookId = uri.getPathSegments().get(1);
                updateRows = db.update("Book", values, "id = ?", new String[]{bookId});
                break;
            case CATEGORY_DIR:
                updateRows = db.update("Category", values, selection, selectionArgs);
                break;
            case CATEGORY_ITEM:
                String categoryId = uri.getPathSegments().get(1);
                updateRows = db.update("Category", values, "id = ?", new String[]{categoryId});
                break;
            default:
                break;
        }
        return updateRows;
    }
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        // 删除数据
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        int deleteRows = 0;
        switch (uriMatcher.match(uri)) {
            case BOOK_DIR:
                deleteRows = db.delete("Book", selection, selectionArgs);
                break;
            case BOOK_ITEM:
                String bookId = uri.getPathSegments().get(1);
                deleteRows = db.delete("Book", "id = ?", new String[]{bookId});
                break;
            case CATEGORY_DIR:
                deleteRows = db.delete("Category", selection, selectionArgs);
                break;
            case CATEGORY_ITEM:
                String categoryId = uri.getPathSegments().get(1);
                deleteRows = db.delete("Category", "id = ?", new String[]{categoryId});
                break;
            default:
                break;
        }
        return deleteRows;
    }
    @Override
    public String getType(Uri uri) {
        switch (uriMatcher.match(uri)) {
            case BOOK_DIR:
                return "vnd.android.cursor.dir/vnd.com.example.sqlitetest.provider.book";
            case BOOK_ITEM:
                return "vnd.android.cursor.item/vnd.com.example.sqlitetest.provider.book";
            case CATEGORY_DIR:
                return "vnd.android.cursor.dir/vnd.com.example.sqlitetest.provider.category";
            case CATEGORY_ITEM:
                return "vnd.android.cursor.item/vnd.com.example.sqlitetest.provider.category";
        }
        return null;
    }
}
```

代码虽然很长，但是这些内容都非常容易理解，因为使用到的全部都是上面的知识。首先在类的一开始，同样是定义了4个常量，分别用于表示访问Book表中的所有数据、访问Book表中的单条数据、访问Category表中的所有数据和访问Category表中的单条数据。然后在静态代码块里对UriMatcher进行了初始化操作，将期望匹配的几种URI格式添加了进去。

接下来就是每个抽象方法的具体实现了，先来看下onCreate方法，这个方法的代码很短，就是创建了一个MyDatabaseHelper的实例，然后返回true表示内容提供器初始化成功，这时数据库就已经完成了创建或升级操作。

接着看一下query方法，在这个方法中先获取到了SQLiteDatabase的实例，然后根据传入的Uri参数判断出用户想要访问哪张表，再调用SQLiteDatabase的query进行查询，并将Cursor对象返回就好了。注意当访问单条数据的时候有一个细节，这里调用了Uri对象的getPathSegments方法，它会将内容URI权限之后的部分以`"/"`符号进行分割，并把分割后的结果放入到一个字符串列表中，那这个列表的第0个位置存放的就是路径，第1个位置存放的就是id了。得到了id之后，再通过selection和selectionArgs参数进行约束，就实现了查询单条数据的功能。

再往后就是 insert方法，同样它也是先获取到了SQLiteDatabase的实例、然后根据传入的Uri参数判断出用户想要往哪张表里添加数据。再调用SQLiteDatabase的insert方法进行添加就可以了。注意insert方法要求返回一个能够表示这条新增数据的URI，所以我们还需要调用`Uri.parse`方法来将一个内容URI解析成Uri对象，当然这个内容URI是以新增数据的id结尾的。

接下来就是update方法了，也是先获取SQLiteDatabase的实例。然后根据传入的Uri参数判断出用户想要更新哪张表里的数据，再调用SQLiteDatabase的update方法进行更新就好了，受影响的行数将作为返回值返回。

下面是delete方法，仍然是先获取到SQLiteDatabase的实例，然后根据传入的Uri参数判断出用户想要删除哪张表里的数据，再调用SQLiteDatabase的delete方法进行删除就好了，被删除的行数将作为返回值返回。

最后是getType方法。这样我们就将内容提供器中的代码全部编写完了。另外还有一点需要注意，内容提供器一定要在AndroidManifest.xml文件中注册才可以使用。不过幸运的是，由于我们是使用Android Studio的快捷方式创建的内容提供器，因此注册这一步已经被自动完成了。打开AndroidManifest.xml文件瞧一瞧，代码如下所示：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.sqlitetest">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.SQLiteTest">
        <provider
            android:name=".DatabaseProvider"
            android:authorities="com.example.sqlitetest.provider"
            android:enabled="true"
            android:exported="true"></provider>

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

可以看到，`<application>`标签内出现了一个新的标签`<provider>`，我们使用它来对`DatabaseProvider`这个内容提供器进行注册。`android:name`属性指定了`DatabaseProvider`的类名，`android:authorities`属性指定了DatabaseProvider的authority，而enabled和exported属性则是根据我们刚才勾选的状态自动生成的，这里表示允许DatabaseProvider被其他应用程序进行访问。

现在DatabaseTest这个项目就已经拥有了跨程序共享数据的功能了，我们赶快来尝试一下。首先需要将DatabaseTest程序删除掉，以防止遗留数据对我们造成干扰。然后运行一下项目，将DatabaseTest程序重新安装在模拟器上了。接着关闭掉DatabaseTest这个项目并创建一个新项目ProviderTest我们就将通过这个程序去访问DatabaseTest中的数据。

先来编写一下布局文件，修改`activity_main.xml`中的代码，如下所示：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
    <Button
        android:id="@+id/add_data"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Add To Book">
    </Button>
    <Button
        android:id="@+id/update_data"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Update Book">
    </Button>
    <Button
        android:id="@+id/delete_data"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Delete From Book">
    </Button>
    <Button
        android:id="@+id/query_data"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Query From Book">
    </Button>
</LinearLayout>
```

布局文件很简单、里面放置了4个按钮，分别用于添加、查询、修改和删除数据。然后修改MainActivity中的代码，如下所示：

```java
public class MainActivity extends AppCompatActivity {
    private String newId;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button addData = (Button) findViewById(R.id.add_data);
        addData.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Uri uri = Uri.parse("content://com.example.sqlitetest.provider/book");
                ContentValues values = new ContentValues();
                values.put("name", "A Clash of Kings");
                values.put("author", "George Martin");
                values.put("pages", 1040);
                values.put("price", 22.85);
                Uri newUri = getContentResolver().insert(uri, values);
                newId = newUri.getPathSegments().get(1);
            }
        });

        Button updateData = (Button) findViewById(R.id.update_data);
        updateData.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Uri uri = Uri.parse("content://com.example.sqlitetest.provider/book");
                ContentValues values = new ContentValues();
                values.put("name", "A Storm of Swords");
                values.put("pages", 1216);
                values.put("price", 24.05);
                getContentResolver().update(uri, values, null, null);
            }
        });

        Button deleteData = (Button) findViewById(R.id.delete_data);
        deleteData.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Uri uri = Uri.parse("content://com.example.sqlitetest.provider/book");
                getContentResolver().delete(uri, null, null);
            }
        });

        Button queryData = (Button) findViewById(R.id.query_data);
        queryData.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Uri uri = Uri.parse("content://com.example.sqlitetest.provider/book");
                Cursor cursor = getContentResolver().query(uri, null, null, null, null, null);
                if(cursor != null) {
                    while(cursor.moveToNext()) {
                        String name = cursor.getString(cursor.getColumnIndex("name"));
                        String author = cursor.getString(cursor.getColumnIndex("author"));
                        int pages = cursor.getInt(cursor.getColumnIndex("pages"));
                        double price = cursor.getDouble(cursor.getColumnIndex("price"));
                        Log.d("MainActivity", "book name is " + name);
                        Log.d("MainActivity", "book author is " + author);
                        Log.d("MainActivity", "book pages is " + pages);
                        Log.d("MainActivity", "book price is " + price);
                    }
                    cursor.close();
                }
            }
        });
    }
}
```

可以看到，我们分别在这4个按钮的点击事件里面处理了增删改查的逻辑。添加数据的时候，首先调用了Uri.parse方法将一个内容URI解析成Uri对象，然后把要添加的数据都存放到ContentValues对象中，接着调用ContentResolver的insert方法执行添加操作就可以了。注意insert方法会返回一个Uri对象，这个对象中包含了新增数据的id，我们通过getPathSegments方法将这个id取出，稍后会用到它。

查询数据的时候，同样是调用了Uri.parse方法将一个内容URI解析成Uri对象，然后调用ContentResolver的query方法去查询数据，查询的结果当然还是存放在Cursor对象中的。之后对Cursor进行遍历，从中取出查询结果，并打印出来。

更新数据的时候，也是先将内容URI解析成Uri对象，然后把想要更新的数据存放到ContentValues对象中，再调用ContentResolver的update方法执行更新操作就可以了。

注意这里我们为了不想让Book表中的其他行受到影响，在调用Uri.parse方法时，给内容URI的尾部增加了一个id，而这个id正是添加数据时所返回的。这就表示我们只希望更新刚刚添加的那条数据，Book表中的其他行都不会受影响。

删除数据的时候，也是使用同样的方法解析了一个以id结尾的内容URI，然后调用ContentResolver的delete方法执行删除操作就可以了。由于我们在内容URI里指定了一个id，因此只会删掉拥有相应id的那行数据，Book表中的其他数据都不会受影响。
