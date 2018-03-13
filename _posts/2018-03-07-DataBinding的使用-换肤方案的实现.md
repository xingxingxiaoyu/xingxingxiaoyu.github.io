---
layout:       post                                          # 使用的布局（不需要改）
title:        DataBinding的使用—换肤方案的实现             # 标题 
subtitle:     DataBinding的使用—换肤方案的实现             # 副标题
date:         2018-03-07                                    # 时间
author:       xingxingxiaoyu                                # 作者
header-img:   "img/home_background.png"                             # 这篇文章标题背景图片
catalog:      true                                          # 是否归档
tags:                                                       # 标签
    - Android
    - DataBinding
---
在学习了```DataBinding```框架后，就想着这个方案在一些场景下的运用，于是根据这个框架的特性，写出了一个换肤的方案，先看效果图：

![效果图](https://note.youdao.com/yws/public/resource/b2be8c9fae448267c62916307f84aeb4/xmlnote/2E344C07B41642C39BC102D4AB8C0CC7/984)
在看下面的实现之前，希望你对```DataBinding```框架有基本的认识。如果你不够了解，可以参考我之前的一片文章[Android 官方框架DataBinding学习笔记](http://www.jianshu.com/p/1d9c6a1f6d0d)，或者参考[官方文档](https://developer.android.google.cn/topic/libraries/data-binding/index.html)。

这个换肤的方案支持字体大小、颜色，背景色以及图片的切换，所以先创建实体类：
```
public class StyleBean extends BaseObservable
{
    /**
     * 字体大小的级别 0-5
     */
    @Bindable
    private int textSizeLivel;
    /**
     * 主题风格 白天或者黑夜
     */
    @Bindable
    private boolean isNight;

    public int getTextSizeLivel()
    {
        return textSizeLivel;
    }

    public void setTextSizeLivel(int textSizeLivel)
    {
        this.textSizeLivel = textSizeLivel;
        notifyPropertyChanged(BR.textSizeLivel);
    }

    public boolean isNight()
    {
        return isNight;
    }

    public void setNight(boolean night)
    {
        isNight = night;
        notifyPropertyChanged(BR._all);
    }

    public StyleBean(int textSizeLivel, boolean isNight)
    {
        this.textSizeLivel = textSizeLivel;
        this.isNight = isNight;
    }
}
```
继承```BaseObservable```类是为了之后修改属性能自动更新```UI```。
然后准备好字体大小以及颜色值的```xml```文件：
```
<?xml version="1.0" encoding="utf-8"?>
<resources>

    <dimen name="size_level_0">10sp</dimen>

    <dimen name="size_level_1">12sp</dimen>

    <dimen name="size_level_2">14sp</dimen>

    <dimen name="size_level_3">16sp</dimen>

    <dimen name="size_level_4">18sp</dimen>

    <dimen name="size_level_5">20sp</dimen>

    <color name="textColorNight">#222222</color>

    <color name="textColorDay">#aaaaaa</color>

    <color name="backgroundNight">#999999</color>

    <color name="backgroundDay">#eeeeee</color>

</resources>
```
直接在实体类里面去初始化这些资源：
```
    public StyleBean(int textSizeLivel, boolean isNight)
    {
        ···
        initResource();
    }

    private void initResource()
    {
        dimens = new float[6];
        Resources resources = BaseApplication.getContext().getResources();
        dimens[0] = resources.getDimension(R.dimen.size_level_0);
        dimens[1] = resources.getDimension(R.dimen.size_level_1);
        dimens[2] = resources.getDimension(R.dimen.size_level_2);
        dimens[3] = resources.getDimension(R.dimen.size_level_3);
        dimens[4] = resources.getDimension(R.dimen.size_level_4);
        dimens[5] = resources.getDimension(R.dimen.size_level_5);

        mColorDay = resources.getColor(R.color.textColorDay);
        mColorNight = resources.getColor(R.color.textColorNight);

        mBackgroundDay = resources.getColor(R.color.backgroundDay);
        mBackgroundNight = resources.getColor(R.color.backgroundNight);
    }
```
然后添加自定义```setter```方法：
```
    @BindingAdapter("android:textSize")
    public static void setTextSize(TextView textView, int level)
    {
        textView.setTextSize(TypedValue.COMPLEX_UNIT_PX, StyleBean.dimens[level]);
    }


    @BindingAdapter("android:textColor")
    public static void setTextColor(TextView textView, boolean isNight)
    {
        textView.setTextColor(isNight ? StyleBean.mColorNight : StyleBean.mColorDay);
    }

    @BindingAdapter("android:background")
    public static void setBackground(View view, boolean isNight)
    {
        view.setBackgroundColor(isNight ? mBackgroundNight : mBackgroundDay);
    }
```
这样，当我们在控件的属性使用表达式设置值，切表达式返回类型兼容我们的方法时，就会调用我们的方法。

然后就可以在```layout```文件里面书写代码了：
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools">

    <data>

        <import type="com.example.xujiafeng.skinpeeler.StyleBean"/>

        <variable
            name="style"
            type="StyleBean"/>
    </data>

    <RelativeLayout
        android:background="@{style.night}"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:padding="20dp"
        tools:context="com.example.xujiafeng.skinpeeler.activity.MainActivity">

        <LinearLayout
            android:id="@+id/layout1"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content">

            <TextView
                android:id="@+id/text"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="待设置的字体"
                android:textColor="@{style.night}"
                android:textSize="@{style.textSizeLivel}"/>
        </LinearLayout>

        <ImageView
            android:id="@+id/image"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_below="@+id/layout"
            android:layout_marginTop="10dp"
            android:src="@{style.night ? @drawable/icon_vip_check:@drawable/icon_vip_uncheck}"/>

        <Button
            android:id="@+id/button_text_size"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_alignParentBottom="true"
            android:text="设置字体大小"/>

        <Button
            android:id="@+id/button_day"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_alignParentBottom="true"
            android:layout_alignParentRight="true"
            android:text="风格"/>
    </RelativeLayout>
</layout>
```
由于我是把自定义```setter```写在```StyleBean```里面，所以必须要导入这个类。

为了实现```App```风格的一致，所以将```StyleBean```的实现类放在```Application```里面，并添加设置风格的方法：
```
public class BaseApplication extends Application
{
    @Override
    public void onCreate()
    {
        super.onCreate();
        sContext = this;
        style = new StyleBean(2, false);
    }

    private static StyleBean style;

    public static StyleBean getStyle()
    {
        return style;
    }

    public static Context sContext;

    public static Context getContext()
    {
        return sContext;
    }

    public static void setTextSize(int level)
    {
        style.setTextSizeLivel(level);
    }

    public static void setNight(boolean isNight)
    {
        style.setNight(isNight);
    }

    public static int getTextSize()
    {
        return style.getTextSizeLivel();
    }

    public static boolean getNight()
    {
        return style.isNight();
    }
}
```
然后在```Activity```里面设置风格的时候，使用```Application```里面的全局```style```，完整的代码如下:
```
public class MainActivity extends AppCompatActivity
{

    private ActivityMainBinding mBinding;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
        mBinding.setStyle(BaseApplication.getStyle());
    }
}
```
添加修改```style```的方法在```StyleBean```中，供按钮去添加事件：
```
public class StyleBean extends BaseObservable
{

    public void changeTextSize(View view)
    {
        final String[] items = {"0", "1", "2", "3", "4", "5"};
        new AlertDialog.Builder(view.getContext()).setItems(items, new DialogInterface.OnClickListener()
        {
            @Override
            public void onClick(DialogInterface dialogInterface, int i)
            {
                BaseApplication.setTextSize(Integer.valueOf(items[i]));
            }
        }).create().show();
    }
    public void changeNight(View view)
    {
        final String[] items = {"夜间", "白天"};
        new AlertDialog.Builder(view.getContext()).setItems(items, new DialogInterface.OnClickListener()
        {
            @Override
            public void onClick(DialogInterface dialogInterface, int i)
            {
                BaseApplication.setNight(i == 0);
            }
        }).create().show();
    }
}

```
```
        <Button
            android:id="@+id/button_text_size"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_alignParentBottom="true"
            android:onClick="@{(view)->style.changeTextSize(view)}"
            android:text="设置字体大小"/>

        <Button
            android:id="@+id/button_day"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_alignParentBottom="true"
            android:layout_alignParentRight="true"
            android:onClick="@{(view)->style.changeNight(view)}"
            android:text="风格"/>
```
这样点击按钮就能够弹窗选择字体大小或者白天黑夜了。

[项目的GitHub地址](https://github.com/xingxingxiaoyu/SkinPeeler)
