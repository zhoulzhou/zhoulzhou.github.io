---
title: 自定义View Layout过程实例详解
date: 2018-12-01 10:10:12
tags:
categories:
- 自定义View
---
#### 实例讲解
为了更好理解ViewGroup的layout过程（特别是复写onLayout（））
下面，我将用2个实例来加深对ViewGroup layout过程的理解

>系统提供的ViewGroup的子类：LinearLayout
>自定义View（继承了ViewGroup类）

#### 实例解析1（LinearLayout）

#### 原理
计算出LinearLayout本身在父布局的位置
计算出LinearLayout中所有子View在容器中的位置

具体流程

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi34.webp)

#### 源码分析
在上述流程中，对于LinearLayout的layout（）的实现与上面所说是一样的，此处不作过多阐述
故直接进入LinearLayout复写的onLayout（）分析

```
/**
  * 源码分析：LinearLayout复写的onLayout（）
  * 注：复写的逻辑 和 LinearLayout measure过程的 onMeasure()类似
  */ 
  @Override
  protected void onLayout(boolean changed, int l, int t, int r, int b) {

      // 根据自身方向属性，而选择不同的处理方式
      if (mOrientation == VERTICAL) {
          layoutVertical(l, t, r, b);
      } else {
          layoutHorizontal(l, t, r, b);
      }
  }
      // 由于垂直 / 水平方向类似，所以此处仅分析垂直方向（Vertical）的处理过程 ->>分析1

/**
  * 分析1：layoutVertical(l, t, r, b)
  */
    void layoutVertical(int left, int top, int right, int bottom) {
       
        // 子View的数量
        final int count = getVirtualChildCount();

        // 1. 遍历子View
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {

                // 2. 计算子View的测量宽 / 高值
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();

                // 3. 确定自身子View的位置
                // 即：递归调用子View的setChildFrame()，实际上是调用了子View的layout() ->>分析2
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);

                // childTop逐渐增大，即后面的子元素会被放置在靠下的位置
                // 这符合垂直方向的LinearLayout的特性
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

                i += getChildrenSkipCount(child, i);
            }
        }
    }

/**
  * 分析2：setChildFrame()
  */
    private void setChildFrame( View child, int left, int top, int width, int height){
        
        // setChildFrame（）仅仅只是调用了子View的layout（）而已
        child.layout(left, top, left ++ width, top + height);

        }
    // 在子View的layout（）又通过调用setFrame（）确定View的四个顶点
    // 即确定了子View的位置
    // 如此不断循环确定所有子View的位置，最终确定ViewGroup的位置
```

#### 实例解析2：自定义View
上面讲的例子是系统提供的、已经封装好的ViewGroup子类：LinearLayout
但是，一般来说我们使用的都是自定义View；
接下来，我用一个简单的例子讲下自定义View的layout（）过程

#### 实例视图说明
实例视图 = 1个ViewGroup（灰色视图），包含1个黄色的子View，如下图：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi35.webp)

#### 原理
计算出ViewGroup在父布局的位置
计算出ViewGroup中子View在容器中的位置

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi36.webp)

#### 具体计算逻辑
具体计算逻辑是指计算子View的位置，即计算四顶点位置 = 计算Left、Top、Right和Bottom；
主要是写在复写的onLayout（）
计算公式如下：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi37.webp)

```
r = Left + width + Left；// 因左右间距一样
b = Top + height + Top；// 因上下间距一样

Left = (r - width) / 2；
Top = (b - height) / 2；

Right = width + Left;
Bottom = height + Top;

```

#### 代码分析
因为其余方法同上，这里不作过多描述，所以这里只分析复写的onLayout（）

```
**
  * 源码分析：LinearLayout复写的onLayout（）
  * 注：复写的逻辑 和 LinearLayout measure过程的 onMeasure()类似
  */ 
  @Override  
protected void onLayout(boolean changed, int l, int t, int r, int b) {  

     // 参数说明
     // changed 当前View的大小和位置改变了 
     // left 左部位置
     // top 顶部位置
     // right 右部位置
     // bottom 底部位置

        // 1. 遍历子View：循环所有子View
        // 注：本例中其实只有一个
        for (int i=0; i<getChildCount(); i++) {
            View child = getChildAt(i);

            // 取出当前子View宽 / 高
            int width = child.getMeasuredWidth();
            int height = child.getMeasuredHeight();

            // 2. 计算当前子View的四个位置值
                // 2.1 位置的计算逻辑
                int mLeft = (r - width) / 2;
                int mTop = (b - height) / 2;
                int mRight =  mLeft + width；
                int mBottom =  mLeft + width；

            // 3. 根据上述4个位置的计算值，设置子View的4个顶点
            // 即确定了子View在父容器的位置
            child.layout(mLeft, mTop, mRight,mBottom);
        }
    }
}
```

布局文件如下：

```
<?xml version="1.0" encoding="utf-8"?>
<scut.carson_ho.layout_demo.Demo_ViewGroup xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:background="#eee998"
    tools:context="scut.carson_ho.layout_demo.MainActivity">

    <Button
        android:text="ChildView"
        android:layout_width="200dip"
        android:layout_height="200dip"
        android:background="#333444"
        android:id="@+id/ChildView" />
</scut.carson_ho.layout_demo.Demo_ViewGroup >
```

效果图

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi38.webp)

细节问题：getWidth() （ getHeight()）与 getMeasuredWidth() （getMeasuredHeight()）获取的宽 （高）有什么区别？
答：首先明确定义：

getWidth() / getHeight()：获得View最终的宽 / 高
getMeasuredWidth() / getMeasuredHeight()：获得 View测量的宽 / 高
先看下各自的源码：

```
// 获得View测量的宽 / 高
  public final int getMeasuredWidth() {  
      return mMeasuredWidth & MEASURED_SIZE_MASK;  
      // measure过程中返回的mMeasuredWidth
  }  

  public final int getMeasuredHeight() {  
      return mMeasuredHeight & MEASURED_SIZE_MASK;  
      // measure过程中返回的mMeasuredHeight
  }  


// 获得View最终的宽 / 高
  public final int getWidth() {  
      return mRight - mLeft;  
      // View最终的宽 = 子View的右边界 - 子view的左边界。
  }  

  public final int getHeight() {  
      return mBottom - mTop;  
     // View最终的高 = 子View的下边界 - 子view的上边界。
  }
```

：一般情况下，二者获取的宽 / 高是相等的

**结论**
在非人为设置的情况下，View的最终宽/高（getWidth() / getHeight()）
与 View的测量宽/高 （getMeasuredWidth() /  getMeasuredHeight()）永远是相等
