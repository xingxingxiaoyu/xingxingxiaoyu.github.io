---
layout:       post                                          # 使用的布局（不需要改）
title:        Android-Notification点击按钮改变内容方案      # 标题 
subtitle:     Android-Notification点击按钮改变内容方案      # 副标题
date:         2018-03-07                                    # 时间
author:       xingxingxiaoyu                                # 作者
header-img:   "img/IMG_078.jpg"                             # 这篇文章标题背景图片
catalog:      true                                          # 是否归档
tags:                                                       # 标签
    - Android
---


最近做项目的时候，遇到一个需求：发送一个带加减按钮和一个数字的通知，点击按钮后，通知的数字相应的要改变。

![通知样式](http://upload-images.jianshu.io/upload_images/2292129-323b0829530ff312.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

听起来很简单，不就是给两个按钮添加点击事件就可以了吗。可是，通知并不属于App的进程，而是Android系统维护的进程，我们在通知里面自定义的布局要通过```RemoteViews```来实现：
```
RemoteViews remoteViews = new RemoteViews(getPackageName(), R.layout.layout_number);
NotificationCompat.Builder.setContent(remoteViews);
```
```RemoteViews```是用来处理跨进程显示的布局方案，一般用于通知和桌面小部件。由于进程的不同，所以```RemoteViews```所应用的布局并不能直接使用```findViewById()```来查找控件，它封装了一些方法来更新```UI```，例如：
```
void setTextViewText(int viewId, CharSequence text);
void setViewVisibility(int viewId, int visibility);
void setProgressBar(int viewId, int max, int progress, boolean indeterminate);
void setImageViewResource(int viewId, int srcId);
```
更多方法可以参考[官方Api文档](https://developer.android.google.cn/reference/android/widget/RemoteViews.html)
它没有提供直接给控件添加点击事件的方法。所以遇到了困境。

那么可不可以直接将整个通知的布局写成自定义控件，把按钮的点击在自定义控件里面实现呢？很遗憾是不可以的，当发送通知时，会直接报下列的错误：
```
android.app.RemoteServiceException: Bad notification posted from package 
com.example.xujiafeng.notificationrefresh
```
其实仔细想想也很正常，如果能这么简单的设置点击事件，```RemoteViews```不可能不暴露给控件设置点击事件的方法给我们。

那么还有什么办法呢？

```RemoteViews```虽然没有直接给控件设置点击事件的方法，可是提供了以下方法：
```
void setOnClickPendingIntent (int viewId, PendingIntent pendingIntent)
```
官方文档对这个方法的解释是：相当于调用```setOnClickListener(android.view.View.OnClickListener)```来启动提供的```PendingIntent```。

而```PendingIntent```是一个延时```Intent```，它内部包含了一个```Intent```，当某种条件满足时他才会启动```Intent```，它有四个方法来获取对象：
```
getActivity(Context, int, Intent, int)；
getActivities(Context, int, Intent[], int)；
getBroadcast(Context, int, Intent, int)；
getService(Context, int, Intent, int)；
```
从方法名就能明白，在```Intent```被启动时，会启动```Activity```，发送广播，启动服务。

所以如果我们想要实现点击```RemoteViews```的某个按钮后来启动```Activity```，代码就如下：
```
Intent intent = new Intent(this, MainActivity.class);
PendingIntent pendingIntent = PendingIntent.getActivity(this, 21, intent, PendingIntent.FLAG_UPDATE_CURRENT);
remoteViews.setOnClickPendingIntent(R.id.iv_add, pendingIntent);
```

那么回到我们之前的问题，怎么实现点击按钮来实现通知上数字的变化呢？

方法是控制点击按钮后发送一条广播，然后App监听这个广播，收到广播后再次发送相同```id```的通知，新的通知是更改了数字了的。这就是具体的思路，下面是具体的代码实现。

发送通知的代码：
```
    private void sendNotification()
    {
        RemoteViews remoteViews = new RemoteViews(getPackageName(), R.layout.layout_number);

        Intent addIntent = new Intent();
        addIntent.setAction("com.example.xujiafeng.notificationrefresh.add");
        remoteViews.setOnClickPendingIntent(R.id.iv_add, PendingIntent.getBroadcast(MainActivity.this, 10, addIntent, PendingIntent.FLAG_UPDATE_CURRENT));

        Intent minusIntent = new Intent();
        addIntent.setAction("com.example.xujiafeng.notificationrefresh.minus");
        remoteViews.setOnClickPendingIntent(R.id.iv_add, PendingIntent.getBroadcast(MainActivity.this, 10, minusIntent, PendingIntent.FLAG_UPDATE_CURRENT));

        remoteViews.setTextViewText(R.id.tv_value, String.valueOf(value));

        notification = new NotificationCompat.Builder(MainActivity.this)
                .setSmallIcon(R.mipmap.ic_launcher)
                .setContentTitle("ContentTitle")
                .setContentText("ContentText")
                .setWhen(System.currentTimeMillis())
                .setTicker("Ticker")
                .setContent(remoteViews)
                .build();
        mNotificationManager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
        mNotificationManager.notify(12, notification);

    }
```
广播接收器：
```
    public class NotificationReceiver extends BroadcastReceiver
    {

        @Override
        public void onReceive(Context context, Intent intent)
        {
            String action = intent.getAction();
            switch (action)
            {
                case "com.example.xujiafeng.notificationrefresh.add":
                    value++;
                    sendNotification();
                    break;
                case "com.example.xujiafeng.notificationrefresh.minus":
                    value--;
                    sendNotification();
                    break;
            }
        }
    }
```
注册广播：
```
        IntentFilter filter = new IntentFilter();
        filter.addAction(ADD_ACTION);
        filter.addAction(MINUS_ACTION);
        mReceiver = new NotificationReceiver();
        registerReceiver(mReceiver, filter);
```
这样就完成了，运行效果如下：
![运行效果](http://upload-images.jianshu.io/upload_images/2292129-727583b6e0be1bbd.gif?imageMogr2/auto-orient/strip)

[项目的GitHub地址](https://github.com/xingxingxiaoyu/NotificationRefresh)
