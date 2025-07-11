---
title: Android_原生设置入口对接
categories:
  - - Android
tags: 
date: 2022/8/31
updated: 
comments: 
published:
---

# 背景

经过调整原生setting activity onCreate的跳转逻辑后，intent to setting需要put一个extra参数才可以成功进入。

调整的代码：`vendor/mediatek/proprietary/packages/apps/MtkSettings/src/com/android/settings/homepage/SettingsHomepageActivity.java`

```java
public class SettingsHomepageActivity extends FragmentActivity {
    private static final String TAG = SettingsHomepageActivity.class.getSimpleName();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        String password = getIntent().getStringExtra("password");
        if(password != null && password.equals("xbs002841")) {
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
            Log.w(TAG, "seewo's Action");
            Intent intent = new Intent();
            intent.setClassName("com.seewo.eclass.settings", "com.seewo.eclass.settings.SettingsActivity");
            startActivity(intent);
            finish();
        }
    }
    // ...
}
```

经跳转逻辑修改后，正确进入activity的方式：

| 参数     | 值        |
| -------- | --------- |
| password | xbs002841 |

```java
Intent intent = new Intent("android.settings.SETTINGS");
intent.putExtra("password", "xbs002841");
startActivity(intent);
```

```
android-app://com.seewo.eclass.settings
android-app://com.example.intenttosetting
android-app://com.seewo.studystation.launcher
```

