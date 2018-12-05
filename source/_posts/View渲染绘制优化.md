---
title: View渲染绘制优化
date: 2018-12-05 09:25:03
tags:
- 渲染
- 绘制
categories:
- 性能优化
---
#### Hidden Cost of Transparency
这小节会介绍如何减少透明区域对性能的影响。通常来说，对于不透明的View，显示它只需要渲染一次即可，可是如果这个View设置了alpha值，会至少需要渲染两次。原因是包含alpha的view需要事先知道混合View的下一层元素是什么，然后再结合上层的View进行Blend混色处理。

在某些情况下，一个包含alpha的View有可能会触发改View在HierarchyView上的父View都被额外重绘一次。下面我们看一个例子，下图演示的ListView中的图片与二级标题都有设置透明度。

{% asset_img xuan.png %}

大多数情况下，屏幕上的元素都是由后向前进行渲染的。在上面的图示中，会先渲染背景图(蓝，绿，红)，然后渲染人物头像图。如果后渲染的元素有设置alpha值，那么这个元素就会和屏幕上已经渲染好的元素做blend处理。很多时候，我们会给整个View设置alpha的来达到fading的动画效果，如果我们图示中的ListView做alpha逐渐减小的处理，我们可以看到ListView上的TextView等等组件会逐渐融合到背景色上。但是在这个过程中，我们无法观察到它其实已经触发了额外的绘制任务，我们的目标是让整个View逐渐透明，可是期间ListView在不停的做Blending的操作，这样会导致不少性能问题。

如何渲染才能够得到我们想要的效果呢？我们可以先按照通常的方式把View上的元素按照从后到前的方式绘制出来，但是不直接显示到屏幕上，而是使用GPU预处理之后，再又GPU渲染到屏幕上，GPU可以对界面上的原始数据直接做旋转，设置透明度等等操作。使用GPU进行渲染，虽然第一次操作相比起直接绘制到屏幕上更加耗时，可是一旦原始纹理数据生成之后，接下去的操作就比较省时省力。

{% asset_img xuan1.png %}

何才能够让GPU来渲染某个View呢？我们可以通过setLayerType的方法来指定View应该如何进行渲染，从SDK 16开始，我们还可以使用ViewPropertyAnimator.alpha().withLayer()来指定。如下图所示：

{% asset_img xuan2.png %}

另外一个例子是包含阴影区域的View，这种类型的View并不会出现我们前面提到的问题，因为他们并不存在层叠的关系。

{% asset_img xuan3.png %}

为了能够让渲染器知道这种情况，避免为这种View占用额外的GPU内存空间，我们可以做下面的设置。

{% asset_img xuan4.png %}

通过上面的设置以后，性能可以得到显著的提升，如下图所示：

{% asset_img xuan5.png %}

#### Avoiding Allocations in onDraw()
我们都知道应该避免在onDraw()方法里面执行导致内存分配的操作，下面讲解下为何需要这样做。

首先onDraw()方法是执行在UI线程的，在UI线程尽量避免做任何可能影响到性能的操作。虽然分配内存的操作并不需要花费太多系统资源，但是这并不意味着是免费无代价的。设备有一定的刷新频率，导致View的onDraw方法会被频繁的调用，如果onDraw方法效率低下，在频繁刷新累积的效应下，效率低的问题会被扩大，然后会对性能有严重的影响。

{% asset_img xuan6.png %}

如果在onDraw里面执行内存分配的操作，会容易导致内存抖动，GC频繁被触发，虽然GC后来被改进为执行在另外一个后台线程(GC操作在2.3以前是同步的，之后是并发)，可是频繁的GC的操作还是会影响到CPU，影响到电量的消耗。

那么简单解决频繁分配内存的方法就是把分配操作移动到onDraw()方法外面，通常情况下，我们会把onDraw()里面new Paint的操作移动到外面，如下面所示：

{% asset_img xuan7.png %}

#### Custom Views and Performance
Android系统有提供超过70多种标准的View，例如TextView，ImageView，Button等等。在某些时候，这些标准的View无法满足我们的需要，那么就需要我们自己来实现一个View，这节会介绍如何优化自定义View的性能。

通常来说，针对自定义View，我们可能犯下面三个错误：
>Useless calls to onDraw()：我们知道调用View.invalidate()会触发View的重绘，有两个原则需要遵守，第1个是仅仅在View的内容发生改变的时候才去触发invalidate方法，第2个是尽量使用ClipRect等方法来提高绘制的性能。
>Useless pixels：减少绘制时不必要的绘制元素，对于那些不可见的元素，我们需要尽量避免重绘。
>Wasted CPU cycles：对于不在屏幕上的元素，可以使用Canvas.quickReject把他们给剔除，避免浪费CPU资源。另外尽量使用GPU来进行UI的渲染，这样能够极大的提高程序的整体表现性能。

最后请时刻牢记，尽量提高View的绘制性能，这样才能保证界面的刷新帧率尽量的高

#### Double Layout Taxation
布局中的任何一个View一旦发生一些属性变化，都可能引起很大的连锁反应。例如某个button的大小突然增加一倍，有可能会导致兄弟视图的位置变化，也有可能导致父视图的大小发生改变。当大量的layout()操作被频繁调用执行的时候，就很可能引起丢帧的现象。

{% asset_img shi17.png %}

例如，在RelativeLayout中，我们通常会定义一些类似alignTop，alignBelow等等属性，如图所示：

{% asset_img shi18.png %}

为了获得视图的准确位置，需要经过下面几个阶段。首先子视图会触发计算自身位置的操作，然后RelativeLayout使用前面计算出来的位置信息做边界的调整的操作，如下面两张图所示：

{% asset_img shi19.png %}

{% asset_img shi20.png %}

经历过上面2个步骤，relativeLayout会立即触发第二次layout()的操作来确定所有子视图的最终位置与大小信息。

除了RelativeLayout会发生两次layout操作之外，LinearLayout也有可能触发两次layout操作，通常情况下LinearLayout只会发生一次layout操作，可是一旦调用了measureWithLargetChild()方法就会导致触发两次layout的操作。另外，通常来说，GridLayout会自动预处理子视图的关系来避免两次layout，可是如果GridLayout里面的某些子视图使用了weight等复杂的属性，还是会导致重复的layout操作。

如果只是少量的重复layout本身并不会引起严重的性能问题，但是如果它们发生在布局的根节点，或者是ListView里面的某个ListItem，这样就会引起比较严重的性能问题。如下图所示：

{% asset_img shi21.png %}

我们可以使用Systrace来跟踪特定的某段操作，如果发现了疑似丢帧的现象，可能就是因为重复layout引起的。通常我们无法避免重复layout，在这种情况下，我们应该尽量保持View Hierarchy的层级比较浅，这样即使发生重复layout，也不会因为布局的层级比较深而增大了重复layout的倍数。另外还有一点需要特别注意，在任何时候都请避免调用requestLayout()的方法，因为一旦调用了requestLayout，会导致该layout的所有父节点都发生重新layout的操作。

{% asset_img shi22.png %}


