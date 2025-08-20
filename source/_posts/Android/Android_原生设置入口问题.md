---
title: Android_原生设置入口问题
categories:
  - - Android
tags: 
date: 2022/8/10
updated: 
comments: 
published:
---

# 问题引入

> 某沃学习机上面，出现几次系统的动画时长缩放被设置为0，导致应用动画异常。

某沃学习机Android版本：10。

>transition_animation_scale 0.5
>window_animation_scale 0.5
>animator_duration_scale 1.0

实际上，这个是开发者选项中才能调整的参数，而且这个设置的入口在定制版系统上屏蔽掉了，不介入第三方应用或者开发工具不会直接打开，所以可能的原因有：

1. 通过开发工具或者第三方应用进入了开发者选项，手动调整参数。
2. 第三方应用修改了动画时长缩放配置；

针对第一种可能：网上下载了一个[Open Settings(shortcut to settings)](https://apkpure.com/open-settings-shortcut-to-settings/dxidev.openhiddensettings)APK。尝试在某度学习机和某沃学习机上打开，发现，某度进入了厂家定制的设置中，而某沃却轻松地进入了Android的原生设置程序中。

实际上学习机这类产品是不想把原生设置的机会让度给普通用户的，所以，某沃的学习机进不去实属没有周全考虑系统安全性。

经自己模拟开发第三方应用设置参数，排除了第三方应用直接设置的可能，见下文。

# Android Settings中System/Global/Secure

https://blog.csdn.net/qq_37580586/article/details/122327215

来电铃声、锁屏时间、日期格式等等这些属性的设置通常有Settings为入口，通过SettingsProvider来进行。SettingsProvider也是所有系统设置的管理者。

在Android 5.0版本之前，SettingsProvider中系统设置是存储在settings.db数据库中；但是在Android 6.0之后，SettingsProvider中系统设置改为由xml存储在data分区。

为何要从settings.db改为xml存储？

1. 这次修改主要涉及到`global`, `secure`, `system`三个表。改为了`settings_global.xml`、`settings_secure.xml`、`settings_system.xml`，对应`/frameworks/base/core/java/android/provider/Settings.java`中的三个内部类：Global、Secure、System。其实还有两个xml，`settings_config.xml`、`settings_ssaid.xml`。
   1. Global：所有的偏好设置对系统的所有用户公开，第三方APP只读；
   2. System：包含各种各样的用户偏好系统设置，第三方APP只读；
   3. Secure：安全性的用户偏好系统设置，第三方APP只读。
2. 实现方式从之前的数据库，改为异步性能更加优良的xml。
3. 这次修改主要是基于性能的考量（写入一条耗时从400ms降低为10ms），同时也能够使得保存数据的过程更加可信。
4. 实际上，使保存数据过程更加可信这一条并不是问题的关键，写入失败的情况非常罕见，而且上层应用修改SettingsProvider设置都是通过SettingsProvider来实现的。所以当上层APP下次再次启动的时候，并不知道数据写入失败。
5. 从db改为xml以及相应逻辑，可以有效的防止某些恶意APP监听某些设置选项，进而频繁的进行操作。
6. 每个用户都有自己的一份SettingsProvider设置xml文档。通常位于`/data/system/users/userid/`。
7. 控制APP针对SettingsProvider的写入，即合法性判断
8. 控制SettingsProvider的大小（数据量大小，占用内存大小）

其实主要原因就是因为性能、安全两个原因。

# 与SystemProperties对比

`settings_global.xml`、`settings_secure.xml`、`settings_system.xml`，对应`/frameworks/base/core/java/android/provider/Settings.java`中的三个内部类。

它们三个都继承了`NameValueTable`，而NameValueTable都继承了`BaseColumns`。查看`/frameworks/base/core/java/android/provider/BaseColumns.java`可以看到：
```java
package android.provider;
public interface BaseColumns {
    /**
     * The unique ID for a row.
     * <P>Type: INTEGER (long)</P>
     */
    public static final String _ID = "_id";

    /**
     * The count of rows in a directory.
     * <P>Type: INTEGER</P>
     */
    public static final String _COUNT = "_count";
}
```

Settings中不论是Global、System还是Secure的数据都由键值对组成的。

SettingsProvider有点类似Android的properties系统（Android属性系统）：SystemProperties。但是SettingsProvider和SystemProperties有以下不同点：

1. 数据保存方式不同：SystemProperties的数据保存属性文件中（`/system/build.prop`等），开机后会被加载到`system properties store`；SettingsProvider的数据保存在文件`/data/system/users/0/settings_***.xml`和数据库`settings.db`中；
2. 作用范围不同：SystemProperties可以实现跨进程、跨层次调用，即底层的`c/c++`可以调用，java层也可以调用；SettingsProvider只能能在java层（APP）使用；
3. 公开程度不同：SystemProperties有部分功能上层第三方APP可以使用（对于加了`@hide`标记的第三方APP仅可读，不可修改）；SettingsProvider上层第三方APP不可以使用；

# Settings的修改方式

## adb shell

通过adb shell进行修改，格式为：

```bash
adb shell settings get [global/system/secure] [key]
adb shell settings put [global/system/secure] [key] [value]
adb shell settings list [global/system/secure]
```

```
PS C:\Users\ThinkPad> adb shell
tb8788p1_64_wifi:/ # settings put global transition_animation_scale 0.5
tb8788p1_64_wifi:/ # settings get global transition_animation_scale
0.5
tb8788p1_64_wifi:/ # settings list system
accelerometer_rotation=0
alarm_alert=content://media/internal/audio/media/35?title=Cesium&canonical=1
alarm_alert_set=1
background_power_saving_enable=1
dim_screen=1
display_paper_mode_activated=0
display_paper_mode_texture=255
dtmf_tone=1
dtmf_tone_type=0
end_button_behavior=2
ethernet_state=1
gsensor_change_all_process=0
haptic_feedback_enabled=0
hearing_aid=0
hide_rotation_lock_toggle_for_accessibility=0
import_remove_running=false
isEthernetOpen=0
is_ambient_sensor_adjust=1
is_gravity_sensor_adjust=1
is_light_sensor_adjust=1
is_proximity_sensor_adjust=1
lockscreen_sounds_enabled=0
machine_serial_number=62210304000055
mode_ringer_streams_affected=166
mute_streams_affected=111
notification_light_pulse=1
notification_sound=content://media/internal/audio/media/56?title=Pixie%20Dust&canonical=1
notification_sound_set=1
notification_vibration_intensity=0
pointer_speed=0
radio.data.stall.recovery.action=0
ring_vibration_intensity=0
ringtone=content://media/internal/audio/media/177?title=Flutey%20Phone&canonical=1
ringtone_set=1
screen_brightness=204
screen_brightness_for_vr=86
screen_brightness_mode=0
screen_off_timeout=600000
s..wo_close_voice_call=1
show_password=0
sound_effects_enabled=1
stu_fingerprint=.........
system_s..wo_sensor_value_report=1
system_user_login_state=1
time_12_24=24
tty_mode=0
user_rotation=0
vibrate_when_ringing=0
voice_trigger_active_command_id=1
voice_wakeup_mode=1
volume_alarm=5
volume_bluetooth_sco=5
volume_music=5
volume_music_speaker=1
volume_music_usb_headset=2
volume_notification=5
volume_ring=5
volume_system=5
volume_voice=3
website_control_state=0
```

## 应用修改

```java
// 示例
import android.provider.Settings;//需要通过提供器中的settings类修改
Settings.Global.putString(mContext.getContentResolver(), "audio_test_result", value);//修改
Settings.Global.getString(mContext.getContentResolver(), "audio_test_result");//查询
```

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <Button
        android:id="@+id/change_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="ChangeAppArgs">
    </Button>
</LinearLayout>
```

```java
import android.provider.Settings;
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button button = (Button) findViewById(R.id.change_button);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Settings.Global.putString(getContentResolver(), "animator_duration_scale", "1.0");
                String ads = Settings.Global.getString(getContentResolver(), "animator_duration_scale");
                Log.d("MainActivity", "animator_duration_scale: " + ads);
            }
        });
    }
}
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.appchangeargtest">
    <!-- error: This Permission is only granted to system apps -->
    <uses-permission android:name="android.permission.WRITE_SECURE_SETTINGS"/>
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppChangeArgTest">
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

提示：`error: This Permission is only granted to system apps`。说明第三方App肯定是无法修改这个动画缩放的，而是系统应用才有权修改，因此也排除了三方应用直接修改动画时长缩放参数。

# 推测如何找到原生设置入口

通过Intent来测试

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.intenttosetting">
    <application ...>
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

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_height="match_parent"
    android:layout_width="match_parent">

    <Button
        android:id="@+id/settings"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="SETTINGS">
    </Button>
    <Button
        android:id="@+id/develop_settings"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="APPLICATION_DEVELOPMENT_SETTINGS">
    </Button>
    <Button
        android:id="@+id/accessibility_settings"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="ACCESSIBILITY_SETTINGS">
    </Button>
    <Button
        android:id="@+id/device_info"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="DEVICE_INFO">
    </Button>
    <Button
        android:id="@+id/top_settings"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="top_settings">
    </Button>

    <EditText
        android:id="@+id/editText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:ems="10"
        android:inputType="textPersonName"
        android:text="packageName" />

    <EditText
        android:id="@+id/editText2"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:ems="10"
        android:inputType="textPersonName"
        android:text="className" />

    <Button
        android:id="@+id/package_class"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="package_class" />

    <EditText
        android:id="@+id/editText3"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:ems="10"
        android:inputType="textPersonName"
        android:text="actionName" />

    <Button
        android:id="@+id/action"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="ACTION" />
</LinearLayout>
```

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button b1 = (Button) findViewById(R.id.settings);
        b1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent("android.settings.SETTINGS");
                startActivity(intent);
            }
        });
        Button b2 = (Button) findViewById(R.id.develop_settings);
        b2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent("com.android.settings.APPLICATION_DEVELOPMENT_SETTINGS");
                startActivity(intent);
            }
        });
        Button b3 = (Button) findViewById(R.id.accessibility_settings);
        b3.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent("android.settings.ACCESSIBILITY_SETTINGS");
                startActivity(intent);
            }
        });
        Button b4 = (Button) findViewById(R.id.device_info);
        b4.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent("android.settings.DEVICE_INFO_SETTINGS");
                startActivity(intent);
            }
        });
        Button b5 = (Button) findViewById(R.id.top_settings);
        b5.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent();
                intent.setClassName("com.android.settings", "com.android.settings.SubSettings");
                startActivity(intent);
            }
        });
        Button b_package_class = (Button) findViewById(R.id.package_class);
        b_package_class.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                EditText t1 = (EditText) findViewById(R.id.editText);
                EditText t2 = (EditText) findViewById(R.id.editText2);
                Intent intent = new Intent();
                intent.setClassName(t1.getText().toString(), t2.getText().toString());
                startActivity(intent);
            }
        });
        Button b_action = (Button) findViewById(R.id.action);
        b_action.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                EditText t3 = (EditText) findViewById(R.id.editText3);
                Intent intent = new Intent(t3.getText().toString());
                startActivity(intent);
            }
        });
    }
}
```

情况如下：

| intent                                                     | 某沃                                                         | 某度                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `android.settings.SETTINGS`                                | 直接进原生设置。                                             | 进厂家设置。                                                 |
| `com.android.settings.APPLICATION_DEVELOPMENT_SETTINGS`    | 提示请先启用开发者选项。如果处于开发者模式，则直接进。       | 提示请先启用开发者选项。如果处于开发者模式，则直接进。       |
| `android.settings.ACCESSIBILITY_SETTINGS`                  | 直接进无障碍设置。                                           | 直接进无障碍设置。                                           |
| `android.settings.DEVICE_INFO_SETTINGS`                    | 直接进关于本机信息，下拉菜单点击若干次版本号，可以进入开发者模式。 | 直接进关于本机信息，下拉菜单点击若干次版本号，可以进入开发者模式。 |
| `com.android.settings`, `com.android.settings.SubSettings` | 崩溃，日志报错，permission denial, id = 1000。疑似是此操作只有系统应用才可操作。 | 直接进原生设置顶层界面！                                     |
| 某一子设置项目的左上角返回按键流向                         | 可以返回到设置顶层界面                                       | 经过处理，统一返回到桌面，即不可返回到设置顶层界面。         |

由以上实验，推测，某沃用户可以通过各种第三方应用先转到原生设置下点按版本号进入开发者模式，后进入开发者选项调整动画时长缩放。

由此，为了避免用户进入原生设置进行配置，造成不可预测的异常现象，需要定制一个设置程序，屏蔽掉原生设置的活动入口。有以下几个方向：

1. 把原来action指向的原生设置程序指向厂家定制设置程序。

   ```xml
   // 以下信息来自.../vendor/mediatek/proprietary/packages/apps/MtkSettings/AndroidManifest.xml
           <activity android:name=".homepage.SettingsHomepageActivity"
                     android:label="@string/settings_label_launcher"
                     android:theme="@style/Theme.Settings.Home"
                     android:launchMode="singleTask">
               <intent-filter android:priority="1">
                   <action android:name="android.settings.SETTINGS" />
                   <category android:name="android.intent.category.DEFAULT" />
               </intent-filter>
               <meta-data android:name="com.android.settings.PRIMARY_PROFILE_CONTROLLED"
                          android:value="true" />
           </activity>
   ```


# 拦截进入开发者模式

进入版本号界面后，快速连续点按版本号，会显示信息：“现在只需再执行n步操作即可进入开发者模式”。以此为入口，检索这个字符串的代号。

```xml
~/myProject/DT15/alps/vendor/mediatek/proprietary/packages/apps/MtkSettings/src$ rg "步操作即可进入开发者模式" ..
../res/values-zh-rCN/strings.xml
26:      <item quantity="other">现在只需再执行 <xliff:g id="STEP_COUNT_1">%1$d</xliff:g> 步操作即可进入开发者模式。</item>
27:      <item quantity="one">现在只需再执行 <xliff:g id="STEP_COUNT_0">%1$d</xliff:g> 步操作即可进入开发者模式。</item>
```

可以看到，这个信息的字符串在`.../res/values-zh-rCN/strings.xml`中定义。

```xml
25     <plurals name="show_dev_countdown" formatted="false" msgid="7201398282729229649">
26       <item quantity="other">现在只需再执行 <xliff:g id="STEP_COUNT_1">%1$d</xliff:g> 步操作即可进入开发者模式。</item>
27       <item quantity="one">现在只需再执行 <xliff:g id="STEP_COUNT_0">%1$d</xliff:g> 步操作即可进入开发者模式。</item>
28     </plurals>
```

这个字符串的name为`show_dev_countdown`，顺藤摸瓜。

```
~/myProject/DT15/alps/vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com$ rg show_dev_countdown
android/settings/deviceinfo/BuildNumberPreferenceController.java
196:                                R.plurals.show_dev_countdown, mDevHitCountdown,
```

可以看到，输出此字符串的代码文件只有一个，`BuildNumberPreferenceController.java`，拦截进入开发者模式的解决之道则聚焦在了此文件上。

# 原生设置入口跳转

```java
public class SettingsHomepageActivity extends FragmentActivity {
    private static final String TAG = SettingsHomepageActivity.class.getSimpleName();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        String password = getIntent().getStringExtra("password");
        if(password != null && password.equals(".........")) {
            Log.w(TAG, "Origin Action");
            setContentView(R.layout.settings_homepage_container);
            final View root = findViewById(R.id.settings_homepage_container);
            root.setSystemUiVisibility(
                    View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);

            setHomepageContainerPaddingTop();

            final Toolbar toolbar = findViewById(R.id.search_action_bar);
            FeatureFactory.getFactory(this).getSearchFeatureProvider()
                    .initSearchToolbar(this /* activity */, toolbar, SettingsEnums.SETTINGS_HOMEPAGE);

            final ImageView avatarView = findViewById(R.id.account_avatar);
            final AvatarViewMixin avatarViewMixin = new AvatarViewMixin(this, avatarView);
            getLifecycle().addObserver(avatarViewMixin);

            if (!getSystemService(ActivityManager.class).isLowRamDevice()) {
                // Only allow contextual feature on high ram devices.
                showFragment(new ContextualCardsFragment(), R.id.contextual_cards_content);
            }
            showFragment(new TopLevelSettings(), R.id.main_content);
            ((FrameLayout) findViewById(R.id.main_content))
                    .getLayoutTransition().enableTransitionType(LayoutTransition.CHANGING);
        } else {
            Log.w(TAG, "s..wo's Action");
            Intent intent = new Intent();
            intent.setClassName("com.s..wo.eclass.settings", "com.s..wo.eclass.settings.SettingsActivity");
            startActivity(intent);
            finish();
        }
    }
    // ...
}
```

# 版本信息入口屏蔽

目的是隐藏设置主页面最下面的“关于平板电脑”子项目。

在源码目录中检索“关于平板电脑”一词。发现此属性名为"`about_settings`"

```
<string name="about_settings" product="tablet" msgid="593457295516533765">"关于平板电脑"</string>
<string name="about_settings" product="default" msgid="1743378368185371685">"关于手机"</string>
<string name="about_settings" product="device" msgid="6717640957897546887">"关于设备"</string>
```

在源码目录中检索“`about_settings`”一词。发现在AndroidManifest.xml中被引用

```
vendor/mediatek/proprietary/packages/apps/MtkSettings/AndroidManifest.xml
1009:                  android:label="@string/about_settings"
```

AndroidManifest.xml中涉及关于设备信息的完整部分：

```xml
        <activity android:name=".Settings$MyDeviceInfoActivity"
                  android:label="@string/about_settings"
                  android:icon="@drawable/ic_homepage_about">
            <intent-filter android:priority="1">
                <action android:name="android.settings.DEVICE_INFO_SETTINGS" />
                <action android:name="android.settings.DEVICE_NAME" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
            <intent-filter android:priority="71">
                <action android:name="android.intent.action.MAIN" />
                <category android:name="com.android.settings.SHORTCUT" />
            </intent-filter>
            <meta-data android:name="com.android.settings.FRAGMENT_CLASS"
                       android:value="com.android.settings.deviceinfo.aboutphone.MyDeviceInfoFragment" />
            <meta-data android:name="com.android.settings.PRIMARY_PROFILE_CONTROLLED"
                       android:value="true" />
        </activity>
```

说明，屏蔽版本信息入口后，依然可以通过“`adb shell am start -a android.settings.DEVICE_INFO_SETTINGS`”进入版本信息界面。

接着检索`android.settings.DEVICE_INFO_SETTINGS`。

```
frameworks/base/core/java/android/provider/Settings.java

    @SdkConstant(SdkConstantType.ACTIVITY_INTENT_ACTION)
    public static final String ACTION_DEVICE_INFO_SETTINGS =
        "android.settings.DEVICE_INFO_SETTINGS";
```

检索`ACTION_DEVICE_INFO_SETTINGS`。

```
frameworks/base/core/java/android/provider/Settings.java
1177:    public static final String ACTION_DEVICE_INFO_SETTINGS =

development/apps/BuildWidget/src/com/android/buildwidget/BuildWidget.java
69:                    new Intent(android.provider.Settings.ACTION_DEVICE_INFO_SETTINGS),

frameworks/base/core/tests/coretests/src/android/provider/SettingsProviderTest.java
339:        assertCanBeHandled(new Intent(Settings.ACTION_DEVICE_INFO_SETTINGS));

cts/apps/CtsVerifier/src/com/android/cts/verifier/managedprovisioning/IntentFiltersTestHelper.java
73:                new Intent(Settings.ACTION_DEVICE_INFO_SETTINGS),

cts/apps/CtsVerifier/src/com/android/cts/verifier/managedprovisioning/UserRestrictions.java
125:            Settings.ACTION_DEVICE_INFO_SETTINGS,
127:            Settings.ACTION_DEVICE_INFO_SETTINGS,

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/uitests/src/com/android/settings/ui/AboutPhoneSettingsTests.java
76:        launchAboutPhoneSettings(Settings.ACTION_DEVICE_INFO_SETTINGS);
```

```
vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/core/gateway/SettingsGateway.java
```



# 应用信息打开按钮屏蔽

全局搜索“打开”。

1. `mtk_strings.xml: vendor/mediatek/proprietary/packages/apps/Mms/res/values-zh-rCN`
2. `strings.xml: vendor/mediatek/proprietary/frameworks/base/res/res/values-zh-rCN`

```
<string name="launch_instant_app" msgid="391581144859010499">"打开"</string>
```

全局搜索“强行停止”

1. `strings.xml: vendor/mediatek/proprietary/frameworks/apps/MtkSettings/res/values-zh-rCN`
   1. 649958863744041872
   2. 7435006169872876756

# 修改的文件

`DT15/alps/vendor/mediatek/proprietary/packages/apps/MtkSettings`

进入android.settings.SETTINGS时会判断调用者或校验extra参数；进入设备信息页面时会判断调用者；去除应用信息页面中“打开”的按钮；

```git
[feature][SettingsEntryBlock] Intercept the entry of AOSP config

[what] 1. when intent to AOSP settings page, it will judge the caller or verify extra parameters; 2. when intent to the AOSP device info page, it will judge the caller; 3. remove the "Open" button in the ASOP app info page;
[why] avoid user mistakenly or deliberately configure parameters to cause exceptions
[how] modify intent logic; hide entries
```

## AppButtonsPreferenceController.java

`src/com/android/settings/applications/appinfo/AppButtonsPreferenceController.java`

```java
@@ -169,9 +169,10 @@ public class AppButtonsPreferenceController extends BasePreferenceController imp
         if (isAvailable()) {
             mButtonsPref = ((ActionButtonsPreference) screen.findPreference(
                     KEY_ACTION_BUTTONS))
-                    .setButton1Text(R.string.launch_instant_app)
-                    .setButton1Icon(R.drawable.ic_settings_open)
-                    .setButton1OnClickListener(v -> launchApplication())
+                    //.setButton1Text(R.string.launch_instant_app)
+                    //.setButton1Icon(R.drawable.ic_settings_open)
+                    //.setButton1OnClickListener(v -> launchApplication())
+                    .setButton1Visible(false)
                     .setButton2Text(R.string.uninstall_text)
                     .setButton2Icon(R.drawable.ic_settings_delete)
                     .setButton2OnClickListener(new UninstallAndDisableButtonListener())
```

## SettingsGateway.java【去除版本号入口】

`src/com/android/settings/core/gateway/SettingsGateway.java`

```java
@@ -190,7 +190,7 @@ public class SettingsGateway {
             UserDictionaryList.class.getName(),
             UserDictionarySettings.class.getName(),
             DisplaySettings.class.getName(),
-            MyDeviceInfoFragment.class.getName(),
+            //MyDeviceInfoFragment.class.getName(),
             ModuleLicensesDashboard.class.getName(),
             ManageApplications.class.getName(),
             FirmwareVersionSettings.class.getName(),
```

## SettingsHomepageActivity.java

`src/com/android/settings/homepage/SettingsHomepageActivity.java`

```java
import android.util.Log;

@@ -37,33 +39,42 @@ import com.android.settings.homepage.contextualcards.ContextualCardsFragment;
 import com.android.settings.overlay.FeatureFactory;
 
 public class SettingsHomepageActivity extends FragmentActivity {
-
+    private static final String TAG = SettingsHomepageActivity.class.getSimpleName();
     @Override
     protected void onCreate(Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
-
-        setContentView(R.layout.settings_homepage_container);
-        final View root = findViewById(R.id.settings_homepage_container);
-        root.setSystemUiVisibility(
-                View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
-
-        setHomepageContainerPaddingTop();
-
-        final Toolbar toolbar = findViewById(R.id.search_action_bar);
-        FeatureFactory.getFactory(this).getSearchFeatureProvider()
-                .initSearchToolbar(this /* activity */, toolbar, SettingsEnums.SETTINGS_HOMEPAGE);
-
-        final ImageView avatarView = findViewById(R.id.account_avatar);
-        final AvatarViewMixin avatarViewMixin = new AvatarViewMixin(this, avatarView);
-        getLifecycle().addObserver(avatarViewMixin);
-
-        if (!getSystemService(ActivityManager.class).isLowRamDevice()) {
-            // Only allow contextual feature on high ram devices.
-            showFragment(new ContextualCardsFragment(), R.id.contextual_cards_content);
+        String password = getIntent().getStringExtra("password");
+        if(password != null && password.equals(".........")) {
+            Log.w(TAG, "Origin Action");
+            setContentView(R.layout.settings_homepage_container);
+            final View root = findViewById(R.id.settings_homepage_container);
+            root.setSystemUiVisibility(
+                    View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
+
+            setHomepageContainerPaddingTop();
+
+            final Toolbar toolbar = findViewById(R.id.search_action_bar);
+            FeatureFactory.getFactory(this).getSearchFeatureProvider()
+                    .initSearchToolbar(this /* activity */, toolbar, SettingsEnums.SETTINGS_HOMEPAGE);
+
+            final ImageView avatarView = findViewById(R.id.account_avatar);
+            final AvatarViewMixin avatarViewMixin = new AvatarViewMixin(this, avatarView);
+            getLifecycle().addObserver(avatarViewMixin);
+
+            if (!getSystemService(ActivityManager.class).isLowRamDevice()) {
+                // Only allow contextual feature on high ram devices.
+                showFragment(new ContextualCardsFragment(), R.id.contextual_cards_content);
+            }
+            showFragment(new TopLevelSettings(), R.id.main_content);
+            ((FrameLayout) findViewById(R.id.main_content))
+                    .getLayoutTransition().enableTransitionType(LayoutTransition.CHANGING);
+        } else {
+            Log.w(TAG, "s..wo's Action");
+            Intent intent = new Intent();
+            intent.setClassName("com.s..wo.eclass.settings", "com.s..wo.eclass.settings.SettingsActivity");
+            startActivity(intent);
+            finish();
         }
-        showFragment(new TopLevelSettings(), R.id.main_content);
-        ((FrameLayout) findViewById(R.id.main_content))
-                .getLayoutTransition().enableTransitionType(LayoutTransition.CHANGING);
     }
```

## MyDeviceInfoFragment.java

`com/android/settings/deviceinfo/aboutphone/MyDeviceInfoFragment.java`

```java
import android.widget.Toast;
    @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        if("android-app://com.android.settings".equals(this.getActivity().getReferrer().toString()))
        {
            use(ImeiInfoPreferenceController.class).setHost(this /* parent */);
            use(DeviceNamePreferenceController.class).setHost(this /* parent */);
            mBuildNumberPreferenceController = use(BuildNumberPreferenceController.class);
            mBuildNumberPreferenceController.setHost(this /* parent */);
        } else {
            Toast.makeText(context, R.string.cannot_into_device_info, Toast.LENGTH_SHORT).show();
            finish();
        }
    }
```

## strings.xml

`vendor/mediatek/proprietary/packages/apps/MtkSettings/res/values/strings.xml`

```
<string name="cannot_into_device_info">Only debuggers can access the Device-Info Page</string>
```

`vendor/mediatek/proprietary/packages/apps/MtkSettings/res/values-zh-rCN/strings.xml`

```
<string name="cannot_into_device_info" msgid="2196318436963951285">非调试者无法进入版本信息页面</string>
```

## 编译冲突

`vendor/mediatek/proprietary/packages/apps/SettingsLib/src/com/android/settingslib/wifi/AccessPoint.java`

# 日志

```
development/apps/Fallback/AndroidManifest.xml
55:                             <action android:name="android.settings.SETTINGS" />

vendor/mediatek/proprietary/packages/apps/MtkSettings/AndroidManifest.xml
160:                <action android:name="android.settings.SETTINGS" />

frameworks/base/core/java/android/provider/Settings.java
122:    public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

cts/hostsidetests/devicepolicy/src/com/android/cts/devicepolicy/DeviceOwnerTest.java
633:            executeShellCommand("am start -a android.settings.SETTINGS");

development/samples/MultiWindow/src/com/example/android/multiwindow/LaunchingAdjacentActivity.java
53:                Intent intent = new Intent("android.settings.SETTINGS");

vendor/mediatek/proprietary/operator/packages/apps/Browser/OP07/src/com/mediatek/op07/browser/OP07NetworkStateHandlerExt.java
158:                Intent intent = new Intent("android.settings.SETTINGS");
```





```
frameworks/base/api/current.txt
38705:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

development/apps/Fallback/AndroidManifest.xml
55:                             <action android:name="android.settings.SETTINGS" />

prebuilts/sdk/5/public/api/android.txt
11335:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/7/public/api/android.txt
11389:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/2/public/api/android.txt
7654:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/6/public/api/android.txt
11342:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/12/public/api/android.txt
15925:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/10/public/api/android.txt
13613:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/22/public/api/android.txt
25214:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/22/system/api/android.txt
26813:    field public static final java.lang.String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/25/public/api/android.txt
32341:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/25/system/api/android.txt
35173:    field public static final java.lang.String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/8/public/api/android.txt
12573:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/11/public/api/android.txt
15541:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/17/public/api/android.txt
18754:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/20/public/api/android.txt
21186:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/28/public/api/android.txt
36463:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/15/public/api/android.txt
17382:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/21/public/api/android.txt
25141:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/14/public/api/android.txt
17255:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/3/public/api/android.txt
8708:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/1/public/api/android.txt
7650:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/13/public/api/android.txt
16020:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/19/public/api/android.txt
21028:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/18/public/api/android.txt
19874:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/4/public/api/android.txt
9945:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/27/public/api/android.txt
34744:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/24/public/api/android.txt
32236:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/27/system/api/android.txt
37925:    field public static final java.lang.String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/23/public/api/android.txt
26448:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/24/system/api/android.txt
35019:    field public static final java.lang.String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/23/system/api/android.txt
28511:    field public static final java.lang.String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/29/public/api/android.txt
38674:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/9/public/api/android.txt
13384:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/16/public/api/android.txt
18326:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/26/public/api/android.txt
34646:    field public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

prebuilts/sdk/26/system/api/android.txt
37772:    field public static final java.lang.String ACTION_SETTINGS = "android.settings.SETTINGS";

frameworks/opt/setupwizard/tools/docs/android-22.txt
25229:    field public static final java.lang.String ACTION_SETTINGS = "android.settings.SETTINGS";

vendor/mediatek/proprietary/packages/apps/MtkSettings/AndroidManifest.xml
160:                <action android:name="android.settings.SETTINGS" />

external/autotest/client/common_lib/cros/arc.py
657:    output = adb_shell('am start -W -a android.settings.SETTINGS')

frameworks/base/core/java/android/provider/Settings.java
122:    public static final String ACTION_SETTINGS = "android.settings.SETTINGS";

out/target/common/obj/APPS/MtkSettings_intermediates/manifest/AndroidManifest.xml.fixed
141:                <action android:name="android.settings.SETTINGS"/>

out/target/common/obj/APPS/MtkSettings_intermediates/manifest/AndroidManifest.xml
168:                <action android:name="android.settings.SETTINGS" />

cts/hostsidetests/devicepolicy/src/com/android/cts/devicepolicy/DeviceOwnerTest.java
633:            executeShellCommand("am start -a android.settings.SETTINGS");

development/samples/MultiWindow/src/com/example/android/multiwindow/LaunchingAdjacentActivity.java
53:                Intent intent = new Intent("android.settings.SETTINGS");

vendor/mediatek/proprietary/operator/packages/apps/Browser/OP07/src/com/mediatek/op07/browser/OP07NetworkStateHandlerExt.java
158:                Intent intent = new Intent("android.settings.SETTINGS");

```

```
xurui@ubuntu:~/myProject/DT15/alps$
rg 'package com.android.settings;' . -j56
./external/flatbuffers/.gitignore: line 114: error parsing glob '.corpus**': invalid use of **; must be one path component
./external/flatbuffers/.gitignore: line 115: error parsing glob '.seed**': invalid use of **; must be one path component
out/target/common/R/com/android/settings/Manifest.java
8:package com.android.settings;

out/target/common/R/com/android/settings/R.java
8:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/ProgressCategoryBase.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/CustomListPreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SummaryPreference.java
15:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/DateTimeSettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/LinkifyUtils.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/CancellablePreference.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/TestingSettingsBroadcastReceiver.java
1:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/EncryptionInterstitial.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/AccessiblePreferenceCategory.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SettingsDumpService.java
15:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/ProgressCategory.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/DisplaySettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/BrightnessPreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/TestingSettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/BandMode.java
1:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/HelpTrampoline.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SetFullBackupPassword.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/MasterClearConfirm.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SettingsLicenseActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/MonitoringCertInfoActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/AirplaneModeEnabler.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/ResetNetworkConfirm.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/ResetNetwork.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/TrustedCredentialsDialogBuilder.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/ActivityPicker.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/IccLockSettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/FallbackHome.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SetupWizardUtils.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/ManualDisplayActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/RingtonePreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/DialogCreatable.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/RestrictedCheckBox.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SetupRedactionInterstitial.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/RadioInfo.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/MasterClear.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/CryptKeeper.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/RegulatoryInfoDisplayActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SelfAvailablePreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/AirplaneModeVoiceActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/Utils.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/RestrictedSettingsFragment.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/TetherSettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/DefaultRingtonePreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SetupEncryptionInterstitial.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/UsageStatsActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/AppWidgetLoader.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/TouchBlockingFrameLayout.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/RestrictedRadioButton.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SubSettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/EditPinPreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/Settings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/UserCredentialsSettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SeekBarDialogPreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SettingsInitialize.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/PointerSpeedPreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SettingsActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/BugreportPreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/AllowBindAppWidgetActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/CheckableLinearLayout.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/CryptKeeperConfirm.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/ButtonBarHandler.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SettingsPreferenceFragment.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/SettingsTutorialDialogWrapperActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/RestrictedListPreference.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/RemoteBugreportActivity.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/LegalSettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/TrustedCredentialsSettings.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/AppWidgetPickActivity.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/ProxySelector.java
17:package com.android.settings;

out/target/common/obj/APPS/MtkSettings_intermediates/srcjars/com/android/settings/Manifest.java
8:package com.android.settings;

out/target/common/obj/APPS/MtkSettings_intermediates/srcjars/com/android/settings/R.java
8:package com.android.settings;

out/target/common/obj/APPS/MtkSettings_intermediates/aapt2/com/android/settings/Manifest.java
8:package com.android.settings;

out/target/common/obj/APPS/MtkSettings_intermediates/aapt2/com/android/settings/R.java
8:package com.android.settings;

out/target/common/obj/JAVA_LIBRARIES/settings-logtags_intermediates/logtags/src/com/android/settings/EventLogTags.java
5:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/HelpTrampolineTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/TetherSettingsTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/unit/src/com/android/settings/UserCredentialsTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/unit/src/com/android/settings/UtilsTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/RestrictedSettingsFragmentTest.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/unit/src/com/android/settings/RegulatoryInfoDisplayActivityTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/SettingsDumpServiceTest.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/SetupWizardUtilsTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/RestrictedListPreferenceTest.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/UtilsTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/SettingsActivityTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/TestUtils.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/LegalSettingsTest.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/ResetNetworkConfirmTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/SettingsPreferenceFragmentTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/RegulatoryInfoDisplayActivityTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/SettingsDialogFragmentTest.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/MasterClearConfirmTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/DisplaySettingsTest.java
1:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/ResetNetworkTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/SettingsLicenseActivityTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/SummaryPreferenceTest.java
16:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/SettingsInitializeTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/MasterClearTest.java
17:package com.android.settings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/unit/src/com/android/settings/notification/ZenModeSettingsIntegrationTest.java
1:package com.android.settings;

```

```
xurui@ubuntu:~/myProject/DT15/alps$
rg 'com.s..wo.eclass.settings' . -j56
./external/flatbuffers/.gitignore: line 114: error parsing glob '.corpus**': invalid use of **; must be one path component
./external/flatbuffers/.gitignore: line 115: error parsing glob '.seed**': invalid use of **; must be one path component
frameworks/base/services/core/java/com/android/server/pm/permission/DefaultPermissionGrantPolicy.java
624:                                      "com.s..wo.eclass.settings",

vendor/mediatek/proprietary/packages/apps/SystemUI/src/com/android/systemui/statusbar/custom/UtilsAc.java
37:                new ComponentName("com.s..wo.eclass.settings", "com.s..wo.eclass.settings.SettingsActivity"));
49:                new ComponentName("com.s..wo.eclass.settings", "com.s..wo.eclass.settings.SettingsActivity"));
61:                new ComponentName("com.s..wo.eclass.settings", "com.s..wo.eclass.settings.SettingsActivity"));
88:                new ComponentName("com.s..wo.eclass.settings", "com.s..wo.eclass.settings.SettingsActivity"));
98:                new ComponentName("com.s..wo.eclass.settings", "com.s..wo.eclass.settings.SettingsActivity"));
161:            intent.setClassName("com.s..wo.eclass.settings", "com.s..wo.eclass.settings.SettingsActivity");

```

```
out/target/common/obj/APPS/MtkSettings_intermediates/proguard_dictionary
21825:com.android.settings.SubSettings -> com.android.settings.SubSettings:

out/target/common/obj/APPS/MtkSettings_intermediates/manifest/AndroidManifest.xml
216:            android:name="com.android.settings.SubSettings"

vendor/mediatek/proprietary/hardware/power/config/mt6739/app_list/power_app_cfg.xml
25:            <Activity name="com.android.settings.SubSettings">

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/location/LocationSlice.java
38:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/core/SubSettingLauncher.java
31:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/core/SettingsBaseActivity.java
42:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/bluetooth/BluetoothSliceBuilder.java
37:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/panel/NfcPanel.java
25:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/search/SearchResultTrampoline.java
27:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/panel/WifiPanel.java
25:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/slices/SliceBuilderUtils.java
47:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/notification/ZenModeSliceBuilder.java
38:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/wifi/slice/WifiSlice.java
49:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/biometrics/fingerprint/FingerprintSettings.java
51:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/homepage/contextualcards/deviceinfo/DataUsageSlice.java
37:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/homepage/contextualcards/slices/NotificationChannelSlice.java
49:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/homepage/contextualcards/slices/BatteryFixSlice.java
45:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/homepage/contextualcards/slices/LowStorageSlice.java
34:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/homepage/contextualcards/deviceinfo/DeviceInfoSlice.java
37:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/homepage/contextualcards/slices/BluetoothDevicesSlice.java
37:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/homepage/contextualcards/deviceinfo/StorageSlice.java
34:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/datausage/DataUsageSummaryPreferenceTest.java
45:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/deviceinfo/storage/UserProfileControllerTest.java
39:import com.android.settings.SubSettings;

vendor/mediatek/proprietary/packages/apps/MtkSettings/tests/robotests/src/com/android/settings/deviceinfo/storage/StorageItemPreferenceControllerTest.java
52:import com.android.settings.SubSettings;

```

