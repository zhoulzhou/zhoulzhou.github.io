---
title: 自定义View Measure过程详解
date: 2018-12-01 09:17:26
tags:
categories:
- ANDROID
---
接上一篇
#### measure过程详解
measure过程 根据View的类型分为2种情况：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi19.webp)

接下来，我将详细分析这两种measure过程

#### 单一View的measure过程
**应用场景**
在无现成的控件View满足需求、需自己实现时，则使用自定义单一View

>如：制作一个支持加载网络图片的ImageView控件
>注：自定义View在多数情况下都有替代方案：图片 / 组合动画，但二者可能会导致内存耗费过大，从而引起内存溢出等问题。

**具体使用**
继承自View、SurfaceView 或 其他View；不包含子View

#### 具体流程

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi20.webp)

下面我将一个个方法进行详细分析：入口 = measure（）

```
/**
  * 源码分析：measure（）
  * 定义：Measure过程的入口；属于View.java类 & final类型，即子类不能重写此方法
  * 作用：基本测量逻辑的判断
  **/ 

    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

        // 参数说明：View的宽 / 高测量规格

        ...

        int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                mMeasureCache.indexOfKey(key);

        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            // 计算视图大小 ->>分析1

        } else {
            ...
      
    }

/**
  * 分析1：onMeasure（）
  * 作用：a. 根据View宽/高的测量规格计算View的宽/高值：getDefaultSize()
  *      b. 存储测量后的View宽 / 高：setMeasuredDimension()
  **/ 
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
    // 参数说明：View的宽 / 高测量规格

    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
                         getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
    // setMeasuredDimension() ：获得View宽/高的测量值 ->>分析2
    // 传入的参数通过getDefaultSize()获得 ->>分析3
}

/**
  * 分析2：setMeasuredDimension()
  * 作用：存储测量后的View宽 / 高
  * 注：该方法即为我们重写onMeasure()所要实现的最终目的
  **/
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {  

    //参数说明：测量后子View的宽 / 高值

        // 将测量后子View的宽 / 高值进行传递
            mMeasuredWidth = measuredWidth;  
            mMeasuredHeight = measuredHeight;  
          
            mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;  
        } 
    // 由于setMeasuredDimension（）的参数是从getDefaultSize()获得的
    // 下面我们继续看getDefaultSize()的介绍

/**
  * 分析3：getDefaultSize()
  * 作用：根据View宽/高的测量规格计算View的宽/高值
  **/
  public static int getDefaultSize(int size, int measureSpec) {  

        // 参数说明：
        // size：提供的默认大小
        // measureSpec：宽/高的测量规格（含模式 & 测量大小）

            // 设置默认大小
            int result = size; 
            
            // 获取宽/高测量规格的模式 & 测量大小
            int specMode = MeasureSpec.getMode(measureSpec);  
            int specSize = MeasureSpec.getSize(measureSpec);  
          
            switch (specMode) {  
                // 模式为UNSPECIFIED时，使用提供的默认大小 = 参数Size
                case MeasureSpec.UNSPECIFIED:  
                    result = size;  
                    break;  

                // 模式为AT_MOST,EXACTLY时，使用View测量后的宽/高值 = measureSpec中的Size
                case MeasureSpec.AT_MOST:  
                case MeasureSpec.EXACTLY:  
                    result = specSize;  
                    break;  
            }  

         // 返回View的宽/高值
            return result;  
        }

```

>上面提到，当模式是UNSPECIFIED时，使用的是提供的默认大小（即第一个参数size）；那么，提供的默认大小具体是多少呢？
>答：在onMeasure（）方法中，getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec)中传入的默认大小是getSuggestedMinimumWidth()。

接下来我们继续看getSuggestedMinimumWidth()的源码分析

>由于getSuggestedMinimumHeight()类似，所以此处仅分析getSuggestedMinimumWidth()

源码分析如下：

```
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth,mBackground.getMinimumWidth());
}

//getSuggestedMinimumHeight()同理

```

从代码可以看出：
**若 View 无设置背景，那么View的宽度 = mMinWidth**
·mMinWidth· = android:minWidth属性所指定的值；
若android:minWidth没指定，则默认为0

**若 View设置了背景，View的宽度为mMinWidth和mBackground.getMinimumWidth()中的最大值**

那么，mBackground.getMinimumWidth()的大小具体指多少？继续看getMinimumWidth()的源码分析：

```
public int getMinimumWidth() {
    final int intrinsicWidth = getIntrinsicWidth();
    //返回背景图Drawable的原始宽度
    return intrinsicWidth > 0 ? intrinsicWidth :0 ;
}

// 由源码可知：mBackground.getMinimumWidth()的大小 = 背景图Drawable的原始宽度
// 若无原始宽度，则为0；
// 注：BitmapDrawable有原始宽度，而ShapeDrawable没有
```

总结：getDefaultSize()计算View的宽/高值的逻辑

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi21.webp)

**至此，单一View的宽/高值已经测量完成，即对于单一View的measure过程已经完成。**

#### 总结
对于单一View的measure过程，如下：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi22.webp)

>实际作用的方法：getDefaultSize() = 计算View的宽/高值、setMeasuredDimension（） = 存储测量后的View宽 / 高

#### ViewGroup的measure过程
**应用场景**
利用现有的组件根据特定的布局方式来组成新的组件

**具体使用**
继承自ViewGroup 或 各种Layout；含有子 View

#### 原理
>遍历 测量所有子View的尺寸
>合并将所有子View的尺寸进行，最终得到ViewGroup父视图的测量值
>自上而下、一层层地传递下去，直到完成整个View树的measure（）过程

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi23.webp)

#### 流程

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi24.webp)

下面我将一个个方法进行详细分析：入口 = measure（）
若需进行自定义ViewGroup，则需重写onMeasure()，下文会提到

```
/**
  * 源码分析：measure()
  * 作用：基本测量逻辑的判断；调用onMeasure()
  * 注：与单一View measure过程中讲的measure()一致
  **/ 
  public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
            mMeasureCache.indexOfKey(key);
    if (cacheIndex < 0 || sIgnoreMeasureCache) {

        // 调用onMeasure()计算视图大小
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    } else {
        ...
}

/**
  * 分析1：onMeasure()
  * 作用：遍历子View & 测量
  * 注：ViewGroup = 一个抽象类 = 无重写View的onMeasure（），需自身复写
  **/

```

**为什么ViewGroup的measure过程不像单一View的measure过程那样对onMeasure（）做统一的实现？**（如下代码）

```
/**
  * 分析：子View的onMeasure()
  * 作用：a. 根据View宽/高的测量规格计算View的宽/高值：getDefaultSize()
  *      b. 存储测量后的View宽 / 高：setMeasuredDimension()
  * 注：与单一View measure过程中讲的onMeasure()一致
  **/ 
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
    // 参数说明：View的宽 / 高测量规格

    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
                         getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
    // setMeasuredDimension() ：获得View宽/高的测量值 
    // 传入的参数通过getDefaultSize()获得
}

```

答：因为不同的ViewGroup子类（LinearLayout、RelativeLayout / 自定义ViewGroup子类等）具备不同的布局特性，这导致他们子View的测量方法各有不同

>而onMeasure（）的作用 = 测量View的宽/高值

**因此，ViewGroup无法对onMeasure（）作统一实现。这个也是单一View的measure过程与ViewGroup过程最大的不同。**

>即 单一View measure过程的onMeasure（）具有统一实现，而ViewGroup则没有
>注：其实，在单一View measure过程中，getDefaultSize()只是简单的测量了宽高值，在实际使用时有时需更精细的测量。所以有时候也需重写onMeasure（）

**在自定义ViewGroup中，关键在于：根据需求复写onMeasure()从而实现你的子View测量逻辑。复写onMeasure()的套路如下：**

```
/**
  * 根据自身的测量逻辑复写onMeasure（），分为3步
  * 1. 遍历所有子View & 测量：measureChildren（）
  * 2. 合并所有子View的尺寸大小,最终得到ViewGroup父视图的测量值（自身实现）
  * 3. 存储测量后View宽/高的值：调用setMeasuredDimension()  
  **/ 

  @Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  

        // 定义存放测量后的View宽/高的变量
        int widthMeasure ;
        int heightMeasure ;

        // 1. 遍历所有子View & 测量(measureChildren（）)
        // ->> 分析1
        measureChildren(widthMeasureSpec, heightMeasureSpec)；

        // 2. 合并所有子View的尺寸大小，最终得到ViewGroup父视图的测量值
         void measureCarson{
             ... // 自身实现
         }

        // 3. 存储测量后View宽/高的值：调用setMeasuredDimension()  
        // 类似单一View的过程，此处不作过多描述
        setMeasuredDimension(widthMeasure,  heightMeasure);  
  }
  // 从上可看出：
  // 复写onMeasure（）有三步，其中2步直接调用系统方法
  // 需自身实现的功能实际仅为步骤2：合并所有子View的尺寸大小

/**
  * 分析1：measureChildren()
  * 作用：遍历子View & 调用measureChild()进行下一步测量
  **/ 

    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        // 参数说明：父视图的测量规格（MeasureSpec）

                final int size = mChildrenCount;
                final View[] children = mChildren;

                // 遍历所有子view
                for (int i = 0; i < size; ++i) {
                    final View child = children[i];
                     // 调用measureChild()进行下一步的测量 ->>分析1
                    if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                        measureChild(child, widthMeasureSpec, heightMeasureSpec);
                    }
                }
            }

/**
  * 分析2：measureChild()
  * 作用：a. 计算单个子View的MeasureSpec
  *      b. 测量每个子View最后的宽 / 高：调用子View的measure()
  **/ 
  protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {

        // 1. 获取子视图的布局参数
        final LayoutParams lp = child.getLayoutParams();

        // 2. 根据父视图的MeasureSpec & 布局参数LayoutParams，计算单个子View的MeasureSpec
        // getChildMeasureSpec() 请看上面第2节储备知识处
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,// 获取 ChildView 的 widthMeasureSpec
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,// 获取 ChildView 的 heightMeasureSpec
                mPaddingTop + mPaddingBottom, lp.height);

        // 3. 将计算好的子View的MeasureSpec值传入measure()，进行最后的测量
        // 下面的流程即类似单一View的过程，此处不作过多描述
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
    // 回到调用原处

```

至此，ViewGroup的measure过程分析完毕

#### 总结
ViewGroup的measure过程如下：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi25.webp)

#### 实例说明
为了让大家更好地理解ViewGroup的measure过程（特别是复写onMeasure()），下面，我将用ViewGroup的子类LinearLayout来分析下ViewGroup的measure过程

#### ViewGroup的measure过程实例解析（LinearLayout）
此处直接进入LinearLayout复写的onMeasure（）代码分析

>详细分析请看代码注释

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

      // 根据不同的布局属性进行不同的计算
      // 此处只选垂直方向的测量过程，即measureVertical()->>分析1
      if (mOrientation == VERTICAL) {
          measureVertical(widthMeasureSpec, heightMeasureSpec);
      } else {
          measureHorizontal(widthMeasureSpec, heightMeasureSpec);
      }

      
}

  /**
    * 分析1：measureVertical()
    * 作用：测量LinearLayout垂直方向的测量尺寸
    **/ 
  void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
      
      /**
       *  其余测量逻辑
       **/
          // 获取垂直方向上的子View个数
          final int count = getVirtualChildCount();

          // 遍历子View获取其高度，并记录下子View中最高的高度数值
          for (int i = 0; i < count; ++i) {
              final View child = getVirtualChildAt(i);

              // 子View不可见，直接跳过该View的measure过程，getChildrenSkipCount()返回值恒为0
              // 注：若view的可见属性设置为VIEW.INVISIBLE，还是会计算该view大小
              if (child.getVisibility() == View.GONE) {
                 i += getChildrenSkipCount(child, i);
                 continue;
              }

              // 记录子View是否有weight属性设置，用于后面判断是否需要二次measure
              totalWeight += lp.weight;

              if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
                  // 如果LinearLayout的specMode为EXACTLY且子View设置了weight属性，在这里会跳过子View的measure过程
                  // 同时标记skippedMeasure属性为true，后面会根据该属性决定是否进行第二次measure
                // 若LinearLayout的子View设置了weight，会进行两次measure计算，比较耗时
                  // 这就是为什么LinearLayout的子View需要使用weight属性时候，最好替换成RelativeLayout布局
                
                  final int totalLength = mTotalLength;
                  mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
                  skippedMeasure = true;
              } else {
                  int oldHeight = Integer.MIN_VALUE;
     /**
       *  步骤1：遍历所有子View & 测量：measureChildren（）
       *  注：该方法内部，最终会调用measureChildren（），从而 遍历所有子View & 测量
       **/
            measureChildBeforeLayout(

                   child, i, widthMeasureSpec, 0, heightMeasureSpec,
                   totalWeight == 0 ? mTotalLength : 0);
                   ...
            }

      /**
       *  步骤2：合并所有子View的尺寸大小,最终得到ViewGroup父视图的测量值（自身实现）
       **/        
              final int childHeight = child.getMeasuredHeight();

              // 1. mTotalLength用于存储LinearLayout在竖直方向的高度
              final int totalLength = mTotalLength;

              // 2. 每测量一个子View的高度， mTotalLength就会增加
              mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                     lp.bottomMargin + getNextLocationOffset(child));
      
              // 3. 记录LinearLayout占用的总高度
              // 即除了子View的高度，还有本身的padding属性值
              mTotalLength += mPaddingTop + mPaddingBottom;
              int heightSize = mTotalLength;

      /**
       *  步骤3：存储测量后View宽/高的值：调用setMeasuredDimension()  
       **/ 
       setMeasureDimension(resolveSizeAndState(maxWidth,width))

    ...

  }
```

至此，自定义View的中最重要、最复杂的measure过程讲解完毕。

#### 总结
本文对自定义View中最重要、最复杂的measure过程进行了详细分析，具体如下图：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi26.webp)


