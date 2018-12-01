---
title: 自定义View Layout过程
date: 2018-12-01 09:52:55
tags:
categories:
- ANDROID
---
#### 目录

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi27.webp)

#### 作用
计算视图（View）的位置 即计算View的四个顶点位置：Left、Top、Right 和 Bottom

#### layout过程详解
类似measure过程，layout过程根据View的类型分为2种情况：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi28.webp)

接下来，我将详细分析这2种情况下的layout过程

#### 单一View的layout过程
**应用场景**
在无现成的控件View满足需求、需自己实现时，则使用自定义单一View

>如：制作一个支持加载网络图片的ImageView控件
>注：自定义View在多数情况下都有替代方案：图片 / 组合动画，但二者可能会导致内存耗费过大，从而引起内存溢出等问题。

**具体使用**
继承自View、SurfaceView 或 其他View；不包含子View

#### 具体流程

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi29.webp)

下面我将一个个方法进行详细分析
#### 源码分析
layout过程的入口 = layout（），具体如下：

```
**
  * 源码分析：layout（）
  * 作用：确定View本身的位置，即设置View本身的四个顶点位置
  */ 
  public void layout(int l, int t, int r, int b) {  

    // 当前视图的四个顶点
    int oldL = mLeft;  
    int oldT = mTop;  
    int oldB = mBottom;  
    int oldR = mRight;  
      
    // 1. 确定View的位置：setFrame（） / setOpticalFrame（）
    // 即初始化四个顶点的值、判断当前View大小和位置是否发生了变化 & 返回 
    // ->>分析1、分析2
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    // 2. 若视图的大小 & 位置发生变化
    // 会重新确定该View所有的子View在父容器的位置：onLayout（）
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {  

        onLayout(changed, l, t, r, b);  
        // 对于单一View的laytou过程：由于单一View是没有子View的，故onLayout（）是一个空实现->>分析3
        // 对于ViewGroup的laytou过程：由于确定位置与具体布局有关，所以onLayout（）在ViewGroup为1个抽象方法，需重写实现（后面会详细说）
  ...

}  

/**
  * 分析1：setFrame（）
  * 作用：根据传入的4个位置值，设置View本身的四个顶点位置
  * 即：最终确定View本身的位置
  */ 
  protected boolean setFrame(int left, int top, int right, int bottom) {
        ...
    // 通过以下赋值语句记录下了视图的位置信息，即确定View的四个顶点
    // 从而确定了视图的位置
    mLeft = left;
    mTop = top;
    mRight = right;
    mBottom = bottom;

    mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);

    }

/**
  * 分析2：setOpticalFrame（）
  * 作用：根据传入的4个位置值，设置View本身的四个顶点位置
  * 即：最终确定View本身的位置
  */ 
  private boolean setOpticalFrame(int left, int top, int right, int bottom) {

        Insets parentInsets = mParent instanceof View ?
                ((View) mParent).getOpticalInsets() : Insets.NONE;

        Insets childInsets = getOpticalInsets();

        // 内部实际上是调用setFrame（）
        return setFrame(
                left   + parentInsets.left - childInsets.left,
                top    + parentInsets.top  - childInsets.top,
                right  + parentInsets.left + childInsets.right,
                bottom + parentInsets.top  + childInsets.bottom);
    }
    // 回到调用原处

/**
  * 分析3：onLayout（）
  * 注：对于单一View的laytou过程
  *    a. 由于单一View是没有子View的，故onLayout（）是一个空实现
  *    b. 由于在layout（）中已经对自身View进行了位置计算，所以单一View的layout过程在layout（）后就已完成了
  */ 
 protected void onLayout(boolean changed, int left, int top, int right, int bottom) {

   // 参数说明
   // changed 当前View的大小和位置改变了 
   // left 左部位置
   // top 顶部位置
   // right 右部位置
   // bottom 底部位置

}
```

至此，单一View的layout过程已分析完毕。

#### 总结
单一View的layout过程解析如下：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi30.webp)

#### ViewGroup的layout过程

**应用场景**
利用现有的组件根据特定的布局方式来组成新的组件

**具体使用**
继承自ViewGroup 或 各种Layout；含有子 View

**原理（步骤）**

>计算自身ViewGroup的位置：layout（）
>遍历子View & 确定自身子View在ViewGroup的位置（调用子View 的 layout（））：onLayout（）
>自上而下、一层层地传递下去，直到完成整个View树的layout（）过程

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi31.webp)

流程

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi32.webp)

**此处需注意：**
ViewGroup 和 View 同样拥有layout（）和onLayout()，但二者不同的：

>一开始计算ViewGroup位置时，调用的是ViewGroup的layout（）和onLayout()；
>当开始遍历子View & 计算子View位置时，调用的是子View的layout（）和onLayout()

下面我将一个个方法进行详细分析：layout过程入口为layout（）

```
/**
  * 源码分析：layout（）
  * 作用：确定View本身的位置，即设置View本身的四个顶点位置
  * 注：与单一View的layout（）源码一致
  */ 
  public void layout(int l, int t, int r, int b) {  

    // 当前视图的四个顶点
    int oldL = mLeft;  
    int oldT = mTop;  
    int oldB = mBottom;  
    int oldR = mRight;  
      
    // 1. 确定View的位置：setFrame（） / setOpticalFrame（）
    // 即初始化四个顶点的值、判断当前View大小和位置是否发生了变化 & 返回 
    // ->>分析1、分析2
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    // 2. 若视图的大小 & 位置发生变化
    // 会重新确定该View所有的子View在父容器的位置：onLayout（）
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {  

        onLayout(changed, l, t, r, b);  
        // 对于单一View的laytou过程：由于单一View是没有子View的，故onLayout（）是一个空实现（上面已分析完毕）
        // 对于ViewGroup的laytou过程：由于确定位置与具体布局有关，所以onLayout（）在ViewGroup为1个抽象方法，需重写实现 ->>分析3
  ...

}  

/**
  * 分析1：setFrame（）
  * 作用：确定View本身的位置，即设置View本身的四个顶点位置
  */ 
  protected boolean setFrame(int left, int top, int right, int bottom) {
        ...
    // 通过以下赋值语句记录下了视图的位置信息，即确定View的四个顶点
    // 从而确定了视图的位置
    mLeft = left;
    mTop = top;
    mRight = right;
    mBottom = bottom;

    mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);

    }

/**
  * 分析2：setOpticalFrame（）
  * 作用：确定View本身的位置，即设置View本身的四个顶点位置
  */ 
  private boolean setOpticalFrame(int left, int top, int right, int bottom) {

        Insets parentInsets = mParent instanceof View ?
                ((View) mParent).getOpticalInsets() : Insets.NONE;

        Insets childInsets = getOpticalInsets();

        // 内部实际上是调用setFrame（）
        return setFrame(
                left   + parentInsets.left - childInsets.left,
                top    + parentInsets.top  - childInsets.top,
                right  + parentInsets.left + childInsets.right,
                bottom + parentInsets.top  + childInsets.bottom);
    }
    // 回到调用原处

/**
  * 分析3：onLayout（）
  * 作用：计算该ViewGroup包含所有的子View在父容器的位置（）
  * 注： 
  *      a. 定义为抽象方法，需重写，因：子View的确定位置与具体布局有关，所以onLayout（）在ViewGroup没有实现
  *      b. 在自定义ViewGroup时必须复写onLayout（）！！！！！
  *      c. 复写原理：遍历子View 、计算当前子View的四个位置值 & 确定自身子View的位置（调用子View layout（））
  */ 
  protected void onLayout(boolean changed, int left, int top, int right, int bottom) {

     // 参数说明
     // changed 当前View的大小和位置改变了 
     // left 左部位置
     // top 顶部位置
     // right 右部位置
     // bottom 底部位置

     // 1. 遍历子View：循环所有子View
          for (int i=0; i<getChildCount(); i++) {
              View child = getChildAt(i);   

              // 2. 计算当前子View的四个位置值
                // 2.1 位置的计算逻辑
                ...// 需自己实现，也是自定义View的关键

                // 2.2 对计算后的位置值进行赋值
                int mLeft  = Left
                int mTop  = Top
                int mRight = Right
                int mBottom = Bottom

              // 3. 根据上述4个位置的计算值，设置子View的4个顶点：调用子view的layout() & 传递计算过的参数
              // 即确定了子View在父容器的位置
              child.layout(mLeft, mTop, mRight, mBottom);
              // 该过程类似于单一View的layout过程中的layout（）和onLayout（），此处不作过多描述
          }
      }
  }
```

#### 总结
对于ViewGroup的layout过程，如下：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi33.webp)

至此，ViewGroup的 layout过程已讲解完毕。

