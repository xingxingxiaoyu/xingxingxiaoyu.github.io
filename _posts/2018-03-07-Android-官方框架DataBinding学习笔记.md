---
layout:       post                                          # 使用的布局（不需要改）
title:        Android-官方框架DataBinding学习笔记           # 标题 
subtitle:     Android-官方框架DataBinding学习笔记           # 副标题
date:         2018-03-07                                    # 时间
author:       xingxingxiaoyu                                # 作者
header-img:   "img/IMG_054.jpg"                             # 这篇文章标题背景图片
catalog:      true                                          # 是否归档
tags:                                                       # 标签
    - Android
    - DataBinding
---
DataBinding是谷歌官方发布的一个框架，它的目的是降低布局和逻辑的耦合性，使代码的逻辑更清晰。它能够很简单的省去```findViewById()```的步骤，大量减少```Activity```的代码，数据直接能写在```layout```文件上，而且它能自动进行空检测，很多地方对象为空不会引起空指针异常。

![DataBinding](http://upload-images.jianshu.io/upload_images/2292129-c9d121a318a93452.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面我将从以下几个方面介绍DataBinding框架：
1. DataBinding在AndroidStudio下的环境搭建
2. DataBinding的简单使用
3. DataBinding的事件处理
4. layout文件细节
5. 观察者对象
6. 生成Binding
7. 属性setters
###DataBinding在AndroidStudio下的环境搭建
由于DataBinding是谷歌的官方框架，所以环境搭建很简单，只需在model下的build.gradle文件上加上如下代码：
```
android {
    ···
    dataBinding {
        enabled = true
    }
    ···
}
```
不过这要求你的Gradle是 1.5.0-alpha1或者更新的版本，AndroidStudio1.3或更新的版本才可以。DataBinding是一个兼容库，他能运行在Android2.1以上（Api level7+）。

###DataBinding的简单使用
DataBinding的```layout```文件与我们一般写的```layout```文件不一样，它包含数据和视图两方面，所以其的```layout```文件如下:
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        tools:context="com.example.databindingtest.MainActivity">
    <data>
        <variable
            name="user"
            type="com.example.databindingtest.User"/>
    </data>
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{user.firstName}"/>
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{user.lastName}"/>
    </LinearLayout>
</layout>
```
```user```是一个java对象，它是被用于这个```layout```的一个变量。
```User```类如下
```
public class User 
{
   public String firstName;
   public String lastName;
   public User(String firstName, String lastName) 
   {
       this.firstName = firstName;
       this.lastName = lastName;
   }
}
```
当在```layout```文件中写```@{user.firstName}```时，会使用```user```对象的```firstName```属性。
一个用DataBinding的方式写出的```layout```文件会产生一个类，类名为```layout```文件名的驼峰式写法加上```Binding```,所以```activity_main.xml```会对应于```ActivityMainBinding```类，这个类包含了```layout```文件的性能，包括数据和视图两部分，可以用如下方式绑定```Activity```和布局文件：
```
DataBindingUtil.setContentView(MainActivity.this, R.layout.main_activity);
```
完整的Activity文件如下：
```
public class MainActivity extends AppCompatActivity
{
    private ActivityMainBinding mBinding;
    private User mUser;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
        mUser = new User("李", "晓峰");
        mBinding.setUser(mUser);
        mBinding.setHandler(new MyHandlers());
        mBinding.setPresenter(new Presenter());
    }
}
```
运行效果：
![DataBinding学习笔记](http://upload-images.jianshu.io/upload_images/2292129-a67d8e4c25c834eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如此就完成了DataBinding的最简单体验。

也可以把User类写成这样：
```
public class User
{
    public String getFirstName()
    {
        return firstName;
    }

    public void setFirstName(String firstName)
    {
        this.firstName = firstName;
    }

    public String getLastName()
    {
        return lastName;
    }

    public void setLastName(String lastName)
    {
        this.lastName = lastName;
    }

    private String firstName;
    private String lastName;

    public User(String firstName, String lastName)
    {
        this.firstName = firstName;
        this.lastName = lastName;
    }
}
```
此时布局文件不能直接调用```user```的```firstName```属性，如果你Ctrl+左键点击```ActivityMainBinding```类，会跳到对应的布局文件，这也就能理解为什么布局文件不能访问```user```的除```public```权限外的其余的属性和方法了。
但是其实布局文件不用做任何修改，也能达到同样的效果，这是因为```@{user.firstName}```会去调用```user```的```getFirstName()```方法,也可以把布局文件写成```@{user.getFirstName}```,将会直接调用```getFirstName()```方法。

如果布局文件写成```@{user.firstName}```，```User```类同时包含```firstName()```方法和```getFirstName()```方法，会优先调用哪个方法呢？
将```User```类改成
```
public class User
{
    ···
    public String getFirstName()
    {
        Log.e("User", "getFirstName");
        return firstName;
    }
    public String firstName()
    {
        Log.e("User", "firstName");
        return "12132";
    }
    ···
}
```
打印日志如下：
```
com.example.databindingtest E/User: getFirstName
```
所以说```getFirstName()```方法被调用了，而```firstName()```方法并没有被调用。

###DataBinding的事件处理
DataBinding允许我们直接将事件写在控件上，事件的属性名决定于```Listener```的方法名，例如长按事件``` View.OnLongClickListener```有方法```onLongClick()```,那么其对应的属性为```android:onLongClick```。
有两种处理事件的方式：
- Method References 事件为一个对象及其的方法
- Listener Bindings 事件为一个Lambda表达式

######Method References
Method References设置事件和使用```android:onClick```属性，将方法写在```Activity```是很相似的，主要的区别是DataBinding的Method References对表达式的检验是在编译期，因此如果方法为空或者不正确，会在编译期被发现。且它的方法不必写在```Activity```里。

事件处理类：
```
public class MyHandlers
{
    public void click(View view)
    {
        Log.e("MyHandlers","onClick");
    }
    public boolean longClick(View view)
    {
        Log.e("MyHandlers","onLongClick");
        return true;
    }
}
```
注意，方法的输入参数和返回值必须和事件监听器的方法一样。
```xml```文件：
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        tools:context="com.example.databindingtest.MainActivity">
        <variable
            name="handler"
            type="com.example.databindingtest.MyHandlers"/>
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:onClick="@{handler::click}"
            android:onLongClick="@{handler::longClick}"
            android:text="button"/>
    </LinearLayout>
</layout>
```
```Activity```加上
```
mBinding.setHandler(new MyHandlers());
```
分别点击和长按按钮，打印日志如下：
```
com.example.databindingtest E/MyHandlers: onClick
com.example.databindingtest E/MyHandlers: onLongClick
```
######Listener Bindings
Listener Bindings是当事件发生的时候绑定表达式，它和Method References是类似的，但是它能绑定任意的输入类型方法，而不必和事件监听器的一样，不过返回值要和事件监听器一样。它的表达式要写成Lambda表达式。

事件处理类：
```
public class Presenter
{
    public void click()
    {
        Log.e( "Presenter"," onClick");
    }
    public boolean longClick()
    {
        Log.e( "Presenter","onLongClick");
        return true;
    }
}
```
```xml```文件：
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        tools:context="com.example.databindingtest.MainActivity">

    <data>
        <variable
            name="presenter"
            type="com.example.databindingtest.Presenter"/>
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:onClick="@{()->presenter.click()}"
            android:onLongClick="@{()->presenter.longClick()}"
            android:text="button"/>
    </LinearLayout>
</layout>

```
```Activity```加上
```
mBinding.setPresenter(new Presenter());
```
日志如下：
```
com.example.databindingtest E/Presenter:  onClick
com.example.databindingtest E/Presenter: onLongClick
```
所以我们在使用MVP模式的时候，就可以不必去```Activity```里去绑定控件的事件与```Presenter```里的方法了。

在这里我们没有传递```view```属性到```View.onClick```方法中, Listener bindings提供给我们两种选择：忽略方法的所有输入参数或者为方法的所有输入参数命名。
例如我们可以写成：
```
android:onClick="@{(view)->presenter.click()}"
```
如果我们想要在表达式中使用view，那么可以这样写：
```
    public void click(View view)
    {
        if (view instanceof Button)
        {
            Log.e("Presenter", " onClick");
        }
    }

android:onClick="@{(button)->presenter.click(button)}"
```
表达式也可以有自己的输入参数：
```
public void click(Task task)

android:onClick="@{()->presenter.click(task)}"
```
```
public void click(View view,Task task)

android:onClick="@{(view)->presenter.click(view,task)}"
```
有一些点击事件与```android:onClick```有冲突，DataBinding给他们分配了一些特别的属性名：

|View| Listener Setter | Attribute|
| :-------------: |:-------------:| :-----:|
|SearchView|setOnSearchClickListener(View.OnClickListener)| android:onSearchClick|
|ZoomControls| setOnZoomInClickListener(View.OnClickListener)|android:onZoomIn|
|ZoomControls|setOnZoomOutClickListener(View.OnClickListener)|android:onZoomOut|

###layout文件细节
前面我们对于Databinding的布局文件做了简单的介绍，现在详细介绍布局文件细节。
######类的导入
就像Java一样，类的使用可以是带包名的类名，也支持导入：
```
<data>
    <import type="com.example.databindingtest.User"/>
    <variable
        name="user"
        type="com.example.databindingtest.User"/>
</data>
```
```java.lang```包下的类不需要导入，可以直接使用
```
<variable
    name="str"
    type="String"/>
```
在布局文件中可以直接使用类的静态变量和方法，不需要调用```ViewDataBinding.setXxx()```
```
public class StaticClass
{
    public static String name = "StaticClass";

    public static void printf(View v)
    {
        Log.e("StaticClass", "printf");
    }
}
```
```
<Button
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      android:text="@{StaticClass.name}"
      android:onClick="@{StaticClass::printf}"/>
```
注意，此时使用Listener Bindings引用方法会报错。

类名支持设置别名：
```
<import
      alias="V"
      type="android.view.View"/>
```
```
<TextView
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      android:text="@{user.lastName}"
      android:visibility="@{user.adult?V.VISIBLE:V.GONE}"/>
```
这样能防止出现类名相同的情况而造成的类名无法识别。

定义和使用集合：
```
<data>
    <import type="com.example.User"/>
    <import type="java.util.List"/>
    <variable name="userList" type="List<User>"/>
</data>
```
```
<TextView
      android:id="@+id/name"
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      android:text="@{userList[0].firstName}"/>
```
其中"<"和">"要使用Html的转义代替，此时AndroidStudio可能会爆红，可是是正确的，可以正确运行。

```Map```集合与其类似
```
android:text="@{map["key"]}"
```
但此时引号有冲突了，所以将外层引号改成单引号：
```
android:text='@{map["key"]}'
```
######Binding类名自定义
```Binding```类的类名可以自定义：
```
<data class="ContactItem">
    ...
</data>
```
指定包名：
```
<data class="com.example.ContactItem">
    ...
</data>
```
######DataBinding运算符和空检测
```DataBinding```的表达式是支持简单的运算符的
- Mathematical （数学计算符） + - / * %
- String concatenation（字符串拼接符） +
- Logical（逻辑运算符） && ||
- Binary（位运算符） & | ^
- Unary（单目运算符） + - ! ~
- Shift（位移运算符） >> >>> <<
- Comparison（比较运算符） == > < >= <=
- instanceof
- Grouping ()
- Literals - character, String, numeric, null
- Cast
- Method calls
- Field access
- Array access []
- Ternary operator（三元运算符） ?:

基本上和Java保持一致。

举例：
```
android:text="@{String.valueOf(index + 1)}"
android:visibility="@{age < 13 ? View.GONE : View.VISIBLE}"
android:transitionName='@{"image_" + id}'
```
不支持```this,super,new```三个关键词。
空检测：
```
android:text="@{user.displayName ?? user.lastName}"
```
它会根据？？运算符左边的对象是否为空来选择值，左边为空现在左边，否则选择右边。
等价于：
```
android:text="@{user.displayName != null ? user.displayName : user.lastName}"
```
注意，```DataBinding```自动进行了很多空指针检测，对象为空调用它的属性或方法不会引起程序崩溃，而是赋予默认值。例如对于这个
```
<TextView
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      android:text="@{user.lastName}"
      android:onClick="@{(button)->user.click(button)}"/>
```
如果```user```对象为空，那么```user.lastName```会被分配它的默认值，对象是```null```，```int```是```0```等等；TextView的点击事件会相当于没有设置。
######资源使用
在表达式中使用xml文件定义的资源是可以的：
```
android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
```
常用的资源和其对应注解如下：

|Type（类型）	|Normal Reference（普通引用）	|Expression Reference（表达式引用）|
|:----:|:----:|:----:|
|String[]|	@array	|@stringArray|
|int[]	|@array|	@intArray|
|TypedArray|	@array|	@typedArray|
|Animator|	@animator|	@animator|
|StateListAnimator	|@animator	|@stateListAnimator|
|color int	|@color	|@color|
|ColorStateList	|@color	|@colorStateList|

###观察者对象
在我们目前的代码中，如果对象改变了某个属性，```UI```是无法自动更新的，其实很好理解，在```User.setFirstName()```方法中，它只是改变了```user```类的属性，而没有通知```UI```层，```DataBinding```已经封装好了通知```UI```层的方法：
```
public class User extends BaseObservable {
   private String firstName;
   private String lastName;
   @Bindable
   public String getFirstName() {
       return this.firstName;
   }
   @Bindable
   public String getLastName() {
       return this.lastName;
   }
   public void setFirstName(String firstName) {
       this.firstName = firstName;
       notifyPropertyChanged(BR.firstName);
   }
   public void setLastName(String lastName) {
       this.lastName = lastName;
       notifyPropertyChanged(BR.lastName);
   }
}
```
在字段的```get```方法或定义处添加```@Bindable```注解，就可以在```BR```类生成对应的字段，然后继承```BaseObservable```类，就可以调用通知```UI```层具体属性修改的方法了。
在按钮的点击事件上加上如下方法实现，就能看到```UI```的更改动画：
```
valueAnimator = ValueAnimator.ofInt(0, 100);
valueAnimator.setDuration(10000);
valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener()
{
      @Override
      public void onAnimationUpdate(ValueAnimator valueAnimator)
      {
          int animatedValue = (int) valueAnimator.getAnimatedValue();
          mUser.setFirstName("李"+animatedValue);
      }
});
valueAnimator.start();
```
凡是在```layout```文件里面出现了的属性，均可以在```BR```类里面找到其的对应。

######ObservableFields
使用```@Bindable```注解并调用```notifyPropertyChanged(BR.xxx)```方法能达到自动更新```UI```的目的，可是过于繁琐，所以```DataBinding```为我们封装好了更简单易用的类```
ObservableFields```，它的源代码很简单：
```
public class ObservableField<T> extends BaseObservable implements Serializable {
    private T mValue;
    public ObservableField(T value) {
        mValue = value;
    }
    public ObservableField() {
    }
    public T get() {
        return mValue;
    }
    public void set(T value) {
        if (value != mValue) {
            mValue = value;
            notifyChange();
        }
    }
}
```
在每次调用```set()```方法的时候，均会调用```notifyChange()```方法，这个方法也是```BaseObservable```提供的，效果等同于```notifyPropertyChanged(BR._all)```。

使用举例：
实体类
```
public class Person
{
    public ObservableField<String> name = new ObservableField<>();
    public ObservableField<String> address = new ObservableField<>();
    public ObservableInt age = new ObservableInt();
}
```
布局文件
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        xmlns:app="http://schemas.android.com/apk/res-auto">

    <data>
        <variable
            name="person"
            type="com.example.databindingtest.Person"/>
    </data>

    <LinearLayout
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context="com.example.databindingtest.SecondActivity">
        <TextView
            android:text='@{person.name}'
            android:id="@+id/tv_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>
    </LinearLayout>
</layout>
```
改变属性的方法调用
```
Person person = new Person();
mBinding.setPerson(person);
person.name.set("lei");
```
当数据是集合时，使用```ObservableArrayMap```
```
ObservableArrayMap<String, Object> user = new ObservableArrayMap<>();
mBinding.setUser(user);
user.put("firstName","1437");
user.put("lastName","dufjklsg");
```
```
    <data>
        <variable
            name="user"
            type="android.databinding.ObservableArrayMap<String,Object>"/>
    </data>
    ···
    <LinearLayout
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context="com.example.databindingtest.SecondActivity">
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text='@{user["firstName"]}'/>
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text='@{user["lastName"]}'/>
    </LinearLayout>
```
在布局文件和java代码上的写法和普通集合基本一致。
如果```key```不是字符串而是数字下标，则使用```ObservableArrayList```和之前```ArrayList```的用法在形式上基本上一致。

###生成Binding
之前我们已经展示了一种生成```Binding```类绑定```Activity```的方法
```
 mBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
```
还有另一种方式：
```
 mBinding = ActivitySecondBinding.inflate(getLayoutInflater());
 setContentView(mBinding.getRoot());
```
第一行生成```Binding```类，第二行绑定```Activity```，这种方式在使用```RecyclerView```的时候会很有用，能够拿到```Binding```类，还可以通过```ViewDataBinding.getRoot()```获取根布局。

适配器的例子：
```
public class MessageAdapter extends RecyclerView.Adapter<MessageAdapter.MyViewHolder>
{

    private LayoutInflater mInflater;
    private ArrayList<Message> messageArrayList;
    private Context context;

    public MessageAdapter(ArrayList<Message> messageArrayList, Context context)
    {
        this.messageArrayList = messageArrayList;
        this.context = context;
        mInflater = LayoutInflater.from(context);
    }

    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType)
    {
        return MyViewHolder.create(mInflater);
    }

    @Override
    public void onBindViewHolder(MyViewHolder holder, int position)
    {
        holder.mBinding.setMessage(messageArrayList.get(position));
    }

    @Override
    public int getItemCount()
    {
        return messageArrayList == null ? 0 : messageArrayList.size();
    }

    static class MyViewHolder extends RecyclerView.ViewHolder
    {

        private ThirdBinding mBinding;

        private MyViewHolder(ThirdBinding binding)
        {
            super(binding.getRoot());
            mBinding = binding;
        }
        private static MyViewHolder create(LayoutInflater inflater)
        {
            ThirdBinding mBinding = ThirdBinding.inflate(inflater);
            return new MyViewHolder(mBinding);
        }
        private void bindData(Message message)
        {
            mBinding.setMessage(message);
        }
    }
}
```

######自动生成控件对象
在生成```Binding```类的同时，```DataBinding```会根据我们在布局文件中设置的```id```自动生成对应的字段：
```
<Button
    android:id="@+id/button_test"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="12312"/>
```
在具体的```Binding```类上的字段
```
public final Button buttonTest;
```
依旧像之前自动转换成驼峰式命名，使用直接调用这个字段就可以：
```
mBinding.buttonTest.setText("");
```
这就是我们完全没必要使用```findViewById()```了。
######变量的set，get方法
正如我们之前所看到的那样，我们在```data```标签下所申明的变量会生成对应的```set```，```get```方法:
```
<variable
    name="user"
    type="com.example.databindingtest.User"/>
```
```
mBinding.setUser(mUser);
mBinding.getUser();
```
###属性setters
对于在布局文件中控件的每一个用表达式描述的属性，```DataBinding```会试着寻找方法来设置属性。属性的名称空间不必匹配，仅仅根据属性名本身。例如，用表达式关联的```TextView```的属性```android:text```会寻找方法```setText(String)```,如果表达式返回的是```int```，则会寻找方法```setText(int)```。如果在已给出的属性中没有某一个属性名，但是有```set```方法,那么我们能很简单的设置属性。例如对于```DrawerLayout```，他有方法```public void setScrimColor(@ColorInt int color)```但是没有属性```android:scrimColor```，我们可以自动的设置上这个属性：
```
<android.support.v4.widget.DrawerLayout
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:scrimColor="@{@color/scrim}"
    app:drawerListener="@{fragment.drawerListener}"/>
```
基于此，我们就能很简单的自定义控件，而不必去写属性值的```xml```文件，例如写一个能设置头的TextView：
```
public class HeadTextView extends android.support.v7.widget.AppCompatTextView
{

    public HeadTextView(Context context, @Nullable AttributeSet attrs)
    {
        super(context, attrs);
    }

    public void setStartText(String startText)
    {
        String text = startText + getText().toString();
        setText(text);
    }
}
```
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:bind="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools"
        tools:context="com.example.databindingtest.MainActivity">

    <data>
        <variable
            name="head"
            type="String"/>
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <com.example.databindingtest.HeadTextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="183****0038"
            app:startText="@{head}"/>

    </LinearLayout>
</layout>

```
```
mBinding.setHead("电话号码：");
```

![简单自定义控件](http://upload-images.jianshu.io/upload_images/2292129-a51adea995ded609.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当然，这个自定义控件还有更多细节需要完善。

注意，```app:startText```的属性值必须要是表达式，而不能像常规的那样写成```app:startText="电话号码："```，只有表达式才能引起```DataBinding```的机制。不过可以写成```app:startText="@{@string/phone}"```，因为表达式里是可以引用资源的。

一些属性有```set```方法但是名称不匹配，可以使用```BindingMethods```注解来联系起方法和属性，例如对于```android:hint```属性，它对应的方法是``` setImageTintList(ColorStateList)```，而不是```setTint```,可以在类的上面加上以下注解来完成匹配：
```
@BindingMethods({
       @BindingMethod(type = ImageView.class,
                      attribute = "android:tint",
                      method = "setImageTintList"),
})
```
注意，此处官方文档写的有误。

Android的框架已经帮我们把```framework```层的属性做了匹配。
对于```HeadTextView```,可以改成：
```
@BindingMethods({
        @BindingMethod(type = TextView.class,
                attribute = "app:startText",
                method = "setStartText111"),
})
public class HeadTextView extends android.support.v7.widget.AppCompatTextView
{

    public HeadTextView(Context context, @Nullable AttributeSet attrs)
    {
        super(context, attrs);
    }

    public void setStartText111(String startText)
    {
        String text = startText + getText().toString();
        setText(text);
    }
}
```
我们可以自己为属性写set方法，例如对于```android:paddingLeft```属性，并没有对应的方法，而有方法```setPadding(left, top, right, bottom)```存在，使用```BindingAdapter```注解能定制属性的```set```方法。
```
@BindingAdapter("android:paddingLeft")
public static void setPaddingLeft(View view, int padding) {
   view.setPadding(padding,
                   view.getPaddingTop(),
                   view.getPaddingRight(),
                   view.getPaddingBottom());
}
```
之前的自定义控件可以这样修改：
```
public class AttrAdapter
{
    @BindingAdapter("app:startText")
    public static void setHead(TextView textView, String head)
    {
        textView.setText(head + textView.getText());
    }
}
```
```
<import type="com.example.databindingtest.AttrAdapter"/>
```
```
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="183****0038"
    app:startText="@{@string/phone}"/>
```
也就是说我们可以在已有的控件上任意的拓展属性！

我们也可以用适配器来接受多个属性:
```
@BindingAdapter({"bind:imageUrl", "bind:error"})
public static void loadImage(ImageView view, String url, Drawable error) {
   Picasso.with(view.getContext()).load(url).error(error).into(view);
}
```
```
<ImageView app:imageUrl="@{venue.imageUrl}"
      app:error="@{@drawable/venueError}"/>
```
当```app:imageUrl```和```app:error```两个属性都被设置了的时候，会调用```loadImage```方法。

适配器里也可以接受之前的属性：
```
@BindingAdapter("android:paddingLeft")
    public static void setPaddingLeft(View view, int oldPadding, int newPadding)
    {
        Log.e("AttrAdapter", "oldPadding=" + oldPadding + " newPadding=" + newPadding);
        if (oldPadding != newPadding)
        {
            view.setPadding(newPadding,
                    newPadding,
                    view.getPaddingRight(),
                    view.getPaddingBottom());
        }
    }
```
```
<variable
    name="left"
    type="int"/>
```
```
<TextView
    android:paddingLeft="@{left}"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="183****0038"
    app:startText="@{@string/phone}"/>
```
```
mBinding.setLeft(10);
```
打印Log如下：
```
com.example.databindingtest E/AttrAdapter: oldPadding=0 newPadding=10
```

表达式的输入值可以与属性要求的值不一样，它会自动寻找要求的方法：
```
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="@{user}"/>
```
```
    @BindingAdapter("android:text")
    public static void setText(TextView view, User s)
    {
        view.setText(s.toString());
    }
```

![DataBinding学习笔记](http://upload-images.jianshu.io/upload_images/2292129-df2800f20c6b575b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###结语
至此，```DataBinding```的主要特性已经学习完毕了，绝大部分内容只是对[官方文档](https://developer.android.google.cn/topic/libraries/data-binding/index.html#custom_setters)的简单翻译。
在这里我想说一句，学习新的知识最正确的方式是直接看官方文档，因为那是第一手的资料，如果英文实在是太差，才考虑去看别人写的相关文章。

[测试代码的GitHub地址](https://github.com/xingxingxiaoyu/SkinPeeler)
