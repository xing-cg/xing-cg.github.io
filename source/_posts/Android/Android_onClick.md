---
title: Android_onClick
categories:
  - - Android
tags: 
date: 2022/8/20
updated: 
comments: 
published:
---

# 不同的绑定点击回调方式

第一种方式：匿名函数

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button button = (Button) findViewById(R.id.button);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                // ...
            }
        });
        
        ImageView imageView = (ImageView) findViewById(R.id.image_view);
        imageView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                // ...
            }
        });
    }
}
```

第二种方式：继承OnClickListener接口，实现onClick函数。使组件直接绑定this。每当被点击时，统一调用onClick，里面进行case逻辑分流。依据View的id与其R.id对比进行。

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener{
    private EditText editText;
    private ImageView imageView;
    private ProgressBar progressBar;
    private static int COUNT;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button button = (Button) findViewById(R.id.button);
        imageView = (ImageView) findViewById(R.id.image_view);
        progressBar = (ProgressBar) findViewById(R.id.progress_bar);
        button.setOnClickListener(this);
        imageView.setOnClickListener(this);
    }
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.button:
                if (progressBar.getVisibility() == View.GONE) {
                    progressBar.setVisibility(View.VISIBLE);
                } else {
                    progressBar.setVisibility(View.GONE);
                }
                break;
            case R.id.image_view:
                if(COUNT == 1) {
                    imageView.setImageResource(R.drawable.img_2);
                    COUNT = 0;
                } else {
                    imageView.setImageResource(R.drawable.img_1);
                    COUNT = 1;
                }
                break;
            default:
                break;
        }
    }
}
```


