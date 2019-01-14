---
title: Android抽象布局-include merge viewstub
date: 2019-01-14 16:24:38
tags:
- include
- merge
- viewstub
categories:
- ANDROID
---
在布局优化中，Androi的官方提到了这三种布局<include />、<merge />、<ViewStub />，并介绍了这三种布局各有的优势，下面也是简单说一下他们的优势，以及怎么使用，记下来权当做笔记。

#### 1、布局重用 include
<include />标签能够重用布局文件，简单的使用如下：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" 
    android:layout_width=”match_parent”
    android:layout_height=”match_parent”
    android:background="@color/app_bg"
    android:gravity="center_horizontal">
 
    <include layout="@layout/titlebar"/>
 
    <TextView android:layout_width=”match_parent”
              android:layout_height="wrap_content"
              android:text="@string/hello"
              android:padding="10dp" />
 
    ...
 
</LinearLayout>
```

1)<include />标签可以使用单独的layout属性，这个也是必须使用的。

2)可以使用其他属性。<include />标签若指定了ID属性，而你的layout也定义了ID，则你的layout的ID会被覆盖，解决方案。

3)在include标签中所有的android:layout_*都是有效的，前提是必须要写layout_width和layout_height两个属性。

4)布局中可以包含两个相同的include标签，引用时可以使用如下方法解决（参考）:

```
View bookmarks_container_2 = findViewById(R.id.bookmarks_favourite); 
 
bookmarks_container_2.findViewById(R.id.bookmarks_list);
```

#### 2、减少视图层级 merge
<merge/>标签在UI的结构优化中起着非常重要的作用，它可以删减多余的层级，优化UI。<merge/>多用于替换FrameLayout或者当一个布局包含另一个时，<merge/>标签消除视图层次结构中多余的视图组。例如你的主布局文件是垂直布局，引入了一个垂直布局的include，这是如果include布局使用的LinearLayout就没意义了，使用的话反而减慢你的UI表现。这时可以使用<merge/>标签优化。

```
<merge xmlns:android="http://schemas.android.com/apk/res/android">
 
    <Button
        android:layout_width="fill_parent" 
        android:layout_height="wrap_content"
        android:text="@string/add"/>
 
    <Button
        android:layout_width="fill_parent" 
        android:layout_height="wrap_content"
        android:text="@string/delete"/>
 
</merge>
```

现在，当你添加该布局文件时(使用<include />标签)，系统忽略<merge />节点并且直接添加两个Button。

#### 3、需要时使用 ViewStub
<ViewStub />标签最大的优点是当你需要时才会加载，使用他并不会影响UI初始化时的性能。各种不常用的布局想进度条、显示错误消息等可以使用<ViewStub />标签，以减少内存使用量，加快渲染速度。<ViewStub />是一个不可见的，大小为0的View。<ViewStub />标签使用如下：

在开发应用程序的时候，经常会在运行时动态根据条件来决定显示哪个View或某个布局。大多数开发者会把可能用到的View都写在上面，先把它们的可见性都设为View.GONE，然后在代码中动态的更改它的可见性。这样做的优点是逻辑简单而且控制起来比较灵活。但它的缺点很明显，耗费资源。虽然把View的初始可见View.GONE，但在Inflate布局的时候View仍然会被Inflate，也就是说仍然会创建对象，会被实例化，会被设置属性，导致耗费内存等资源。标签最大的优点是当你需要时才会加载，使用他并不会影响UI初始化时的性能。各种不常用的布局想进度条、显示错误消息等可以使用标签，以减少内存使用量，加快渲染速度。是一个不可见的，大小为0的View。ViewStub是一个轻量级的View，它一个看不见的，不占布局位置，占用资源非常小的控件。可以为ViewStub指定一个布局，在Inflate布局的时候，只有ViewStub会被初始化，然后当ViewStub被设置为可见的时候，或是调用了ViewStub.inflate()的时候，ViewStub所向的布局就会被Inflate和实例化，然后ViewStub的布局属性都会传给它所指向的布局。这样，就可以使用ViewStub来方便的在运行时控制是否显示某个布局。

```
<ViewStub
    android:id="@+id/stub_import"
    android:inflatedId="@+id/panel_import"
    android:layout="@layout/progress_overlay"
    android:layout_width="fill_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="bottom" />
```

当你想加载布局时，可以使用下面其中一种方法：

```
((ViewStub) findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);
// or
View importPanel = ((ViewStub) findViewById(R.id.stub_import)).inflate();
```

当调用inflate()函数的时候，ViewStub被引用的资源替代，并且返回引用的view。 这样程序可以直接得到引用的view而不用再次调用函数findViewById()来查找了。
注：ViewStub目前有个缺陷就是还不支持 <merge /> 标签。

但ViewStub也不是万能的，下面总结下ViewStub能做的事儿和什么时候该用ViewStub，什么时候该用可见性的控制。
1）ViewStub只能Inflate一次，之后ViewStub对象会被置为空。按句话说，某个被ViewStub指定的布局被Inflate后，就不会够再通过ViewStub来控制它了。

2） ViewStub只能用来Inflate一个布局文件，而不是某个具体的View，当然也可以把View写在某个布局文件中。

基于以上的特点，那么可以考虑使用ViewStub的情况有：
1）在程序的运行期间，某个布局在Inflate后，就不会有变化，除非重新启动。因为ViewStub只能Inflate一次，之后会被置空，所以无法指望后面接着使用ViewStub来控制布局。所以当需要在运行时不止一次的显示和隐藏某个布局，那么ViewStub是做不到的。这时就只能使用View的可见性来控制了。

2） 想要控制显示与隐藏的是一个布局文件，而非某个View。因为设置给ViewStub的只能是某个布局文件的Id，所以无法让它来控制某个View。所以，如果想要控制某个View(如Button或TextView)的显示与隐藏，或者想要在运行时不断的显示与隐藏某个布局或View，只能使用View的可见性来控制。使用的时候的注意事项：某些布局属性要加在ViewStub而不是实际的布局上面，而ViewStub的属性在inflate()后会都传给相应的布局。
