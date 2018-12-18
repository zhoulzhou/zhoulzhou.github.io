---
title: 代码中设置view的layoutparams
date: 2018-12-18 21:02:20
tags:
- LayoutParams
categories:
- ANDROID
---
#### LayoutParams分类和作用
 LayoutParams是ViewGroup类中的子类,而ViewGroup我们都知道,它是容纳组件的容器。比如说:LinearLayout,ListView都是继承ViewGroup的。LayoutParams主要设置的是子view在父类布局的参数,有如下两种方式:
 
 **(1)通过XML文件定义**
 
 ```
 <relativelayout xmlns:android="http://schemas.android.com/apk/res/android" 
 xmlns:tools="http://schemas.android.com/tools" 
 android:layout_width="match_parent" 
 android:layout_height="match_parent">
 
    <textview 
	android:layout_width="wrap_content" 
	android:layout_height="wrap_content" 
	android:text="xiaotang">
 
    </textview>
</relativelayout>
```

可以看到,TextView需要设置layout_width和layout_height参数,才能在父布局中显示出来。

**(2)通过java代码添加View**

```
protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		RelativeLayout contentView = (RelativeLayout) View.inflate(this,
				R.layout.activity_main, null);
		TextView tv = new TextView(this);
		RelativeLayout.LayoutParams lp = new RelativeLayout.LayoutParams(
				new LayoutParams(-1, -1));
		tv.setText("xiaotang");
		contentView.addView(tv);
	}
```

可以看到在java代码中添加view,需要设置其在父布局中的布局参数,才能顺利显示出来.当然在这里,你不申明lp布局参数也可,contentView.addView(tv)会判断你是否设置view的布局参数，若没设置,则默认提供Warp_content类的布局参数.那么View的LayoutParams为什么要设置成RelativeLayout类型呢？后面我们会详细讨论的.

#### View的LayoutParams与父类的关系

**View的LayoutParams设置在父类里面才能起到效果.**大家应该重视这句话,你真正理解了这句话了吗？

```
protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		RelativeLayout contentView = (RelativeLayout) View.inflate(this,
				R.layout.activity_main, null);
		View view = View.inflate(this, R.layout.textview, null);
		RelativeLayout.LayoutParams lp = new RelativeLayout.LayoutParams(
				new LayoutParams(LayoutParams.FILL_PARENT,
						LayoutParams.FILL_PARENT));
		view.setLayoutParams(lp);
		contentView.addView(view);
		setContentView(contentView);
	}
```

#### 为什么view的LayoutParams要是父容器的的LayoutParams
大家看我上面的举的实例中,细心的你就会发现。为什么view放在contentView中，申明的lp(布局参数)都是RelativeLayout类的？原因就是装载该View的容器就是RelativeLayout.
我们再看看下面这个例子:

```
Display display = this.getWindowManager().getDefaultDisplay();
		int width = display.getWidth(); // 获取屏幕宽度
		int height = display.getHeight(); // 获取屏幕高度
		View head = View.inflate(this, R.layout.head, null);
		AbsListView.LayoutParams layoutParams = new AbsListView.LayoutParams(
				AbsListView.LayoutParams.MATCH_PARENT, this.getResources()
						.getDimensionPixelOffset(R.dimen.top_bg_height));
		head.setLayoutParams(layoutParams);
		listView.addHeaderView(head);
```

是ListView添加View的例子,可以看到view的LayoutParams的类型就是AbsListView(ListView继承AbsListView并且在ListView中没有修改该LayoutParams).

**原因分析:**
     我们都知道,每一个视图的绘制过程都必须经历三个最主要的阶段，即onMeasure()、onLayout()和onDraw(),父容器不仅要负责自己的显示，而且还要遍历取负责子组件的显示.我们查看容器的onMeasure()方法就能看到有如下操作:
这里以RelativeLayout为例:

```
LayoutParams params = (LayoutParams) child.getLayoutParams();  //这里的LayoutParams是RelativeLayout类型
```
