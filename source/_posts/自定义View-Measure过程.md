---
title: 自定义View Measure过程
date: 2018-12-01 08:48:56
tags:
categories:
- ANDROID
---
#### 目录

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi11.webp)

#### 作用
测量View的宽 / 高

>在某些情况下，需要多次测量（measure）才能确定View最终的宽/高；
>该情况下，measure过程后得到的宽 / 高可能不准确；
>此处建议：在layout过程中onLayout()去获取最终的宽 / 高

#### 储备知识
了解measure过程前，需要先了解传递尺寸（宽 / 高测量值）的2个类：

>ViewGroup.LayoutParams类（）
>MeasureSpecs 类（父视图对子视图的测量要求）

#### ViewGroup.LayoutParams
布局参数类

>ViewGroup 的子类（RelativeLayout、LinearLayout）有其对应的 ViewGroup.LayoutParams 子类
>如：RelativeLayout的 ViewGroup.LayoutParams子类= RelativeLayoutParams

作用
指定视图View 的高度（height） 和 宽度（width）等布局参数。

具体使用
通过以下参数指定

参数|解释
---|:--:|---:
具体值|dp / px
fill_parent|强制性使子视图的大小扩展至与父视图大小相等（不含 padding )
match_parent|与fill_parent相同，用于Android 2.3 & 之后版本
wrap_content|自适应大小，强制性地使视图扩展以便显示其全部内容(含 padding )

```
android:layout_height="wrap_content"   //自适应大小  
android:layout_height="match_parent"   //与父视图等高  
android:layout_height="fill_parent"    //与父视图等高  
android:layout_height="100dip"         //精确设置高度值为 100dip
```

**构造函数**
构造函数 = View的入口，可用于初始化 & 获取自定义属性

```
// View的构造函数有四种重载
    public DIY_View(Context context){
        super(context);
    }

    public DIY_View(Context context,AttributeSet attrs){
        super(context, attrs);
    }

    public DIY_View(Context context,AttributeSet attrs,int defStyleAttr ){
        super(context, attrs,defStyleAttr);

// 第三个参数：默认Style
// 默认Style：指在当前Application或Activity所用的Theme中的默认Style
// 且只有在明确调用的时候才会生效，
    }
    
    public DIY_View(Context context,AttributeSet attrs,int defStyleAttr ，int defStyleRes){
        super(context, attrs，defStyleAttr，defStyleRes);
    }

// 最常用的是1和2
}
```

#### MeasureSpec
**简介**

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi12.webp)

**组成**
测量规格（MeasureSpec） = 测量模式（mode） + 测量大小（size）

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi13.webp)

其中，测量模式（Mode）的类型有3种：UNSPECIFIED、EXACTLY 和 AT_MOST。具体如下：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi14.webp)

**具体使用**
>MeasureSpec 被封装在View类中的一个内部类里：MeasureSpec类
>MeasureSpec类 用1个变量封装了2个数据（size，mode）：通过使用二进制，将测量模式（mode） & 测量大小(size）打包成一个int值来，并提供了打包 & 解包的方法

**该措施的目的 = 减少对象内存分配**

**实际使用**

```
/**
  * MeasureSpec类的具体使用
  **/

    // 1. 获取测量模式（Mode）
    int specMode = MeasureSpec.getMode(measureSpec)

    // 2. 获取测量大小（Size）
    int specSize = MeasureSpec.getSize(measureSpec)

    // 3. 通过Mode 和 Size 生成新的SpecMode
    int measureSpec=MeasureSpec.makeMeasureSpec(size, mode);

```

**源码分析**

```
**
  * MeasureSpec类的源码分析
  **/
    public class MeasureSpec {

        // 进位大小 = 2的30次方
        // int的大小为32位，所以进位30位 = 使用int的32和31位做标志位
        private static final int MODE_SHIFT = 30;  
          
        // 运算遮罩：0x3为16进制，10进制为3，二进制为11
        // 3向左进位30 = 11 00000000000(11后跟30个0)  
        // 作用：用1标注需要的值，0标注不要的值。因1与任何数做与运算都得任何数、0与任何数做与运算都得0
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;  
  
        // UNSPECIFIED的模式设置：0向左进位30 = 00后跟30个0，即00 00000000000
        // 通过高2位
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;  
        
        // EXACTLY的模式设置：1向左进位30 = 01后跟30个0 ，即01 00000000000
        public static final int EXACTLY = 1 << MODE_SHIFT;  

        // AT_MOST的模式设置：2向左进位30 = 10后跟30个0，即10 00000000000
        public static final int AT_MOST = 2 << MODE_SHIFT;  
  
        /**
          * makeMeasureSpec（）方法
          * 作用：根据提供的size和mode得到一个详细的测量结果吗，即measureSpec
          **/ 
            public static int makeMeasureSpec(int size, int mode) {  
            
                return size + mode;  
            // measureSpec = size + mode；此为二进制的加法 而不是十进制
            // 设计目的：使用一个32位的二进制数，其中：32和31位代表测量模式（mode）、后30位代表测量大小（size）
            // 例如size=100(4)，mode=AT_MOST，则measureSpec=100+10000...00=10000..00100  

            }  
      
        /**
          * getMode（）方法
          * 作用：通过measureSpec获得测量模式（mode）
          **/    

            public static int getMode(int measureSpec) {  
             
                return (measureSpec & MODE_MASK);  
                // 即：测量模式（mode） = measureSpec & MODE_MASK;  
                // MODE_MASK = 运算遮罩 = 11 00000000000(11后跟30个0)
                //原理：保留measureSpec的高2位（即测量模式）、使用0替换后30位
                // 例如10 00..00100 & 11 00..00(11后跟30个0) = 10 00..00(AT_MOST)，这样就得到了mode的值

            }  
        /**
          * getSize方法
          * 作用：通过measureSpec获得测量大小size
          **/       
            public static int getSize(int measureSpec) {  
             
                return (measureSpec & ~MODE_MASK);  
                // size = measureSpec & ~MODE_MASK;  
               // 原理类似上面，即 将MODE_MASK取反，也就是变成了00 111111(00后跟30个1)，将32,31替换成0也就是去掉mode，保留后30位的size  
            } 

    }

```

**MeasureSpec值的计算**
>上面讲了那么久MeasureSpec，那么MeasureSpec值到底是如何计算得来?
>结论：子View的MeasureSpec值根据子View的布局参数（LayoutParams）和父容器的MeasureSpec值计算得来的，具体计算逻辑封装在getChildMeasureSpec()里。如下图：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi15.webp)

**即：子view的大小由父view的MeasureSpec值 和 子view的LayoutParams属性 共同决定**

下面，我们来看getChildMeasureSpec()的源码分析：

```
/**
  * 源码分析：getChildMeasureSpec（）
  * 作用：根据父视图的MeasureSpec & 布局参数LayoutParams，计算单个子View的MeasureSpec
  * 注：子view的大小由父view的MeasureSpec值 和 子view的LayoutParams属性 共同决定
  **/

    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {  

         //参数说明
         * @param spec 父view的详细测量值(MeasureSpec) 
         * @param padding view当前尺寸的的内边距和外边距(padding,margin) 
         * @param childDimension 子视图的布局参数（宽/高）

            //父view的测量模式
            int specMode = MeasureSpec.getMode(spec);     

            //父view的大小
            int specSize = MeasureSpec.getSize(spec);     
          
            //通过父view计算出的子view = 父大小-边距（父要求的大小，但子view不一定用这个值）   
            int size = Math.max(0, specSize - padding);  
          
            //子view想要的实际大小和模式（需要计算）  
            int resultSize = 0;  
            int resultMode = 0;  
          
            //通过父view的MeasureSpec和子view的LayoutParams确定子view的大小  


            // 当父view的模式为EXACITY时，父view强加给子view确切的值
           //一般是父view设置为match_parent或者固定值的ViewGroup 
            switch (specMode) {  
            case MeasureSpec.EXACTLY:  
                // 当子view的LayoutParams>0，即有确切的值  
                if (childDimension >= 0) {  
                    //子view大小为子自身所赋的值，模式大小为EXACTLY  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  

                // 当子view的LayoutParams为MATCH_PARENT时(-1)  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    //子view大小为父view大小，模式为EXACTLY  
                    resultSize = size;  
                    resultMode = MeasureSpec.EXACTLY;  

                // 当子view的LayoutParams为WRAP_CONTENT时(-2)      
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    //子view决定自己的大小，但最大不能超过父view，模式为AT_MOST  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                }  
                break;  
          
            // 当父view的模式为AT_MOST时，父view强加给子view一个最大的值。（一般是父view设置为wrap_content）  
            case MeasureSpec.AT_MOST:  
                // 道理同上  
                if (childDimension >= 0) {  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    resultSize = size;  
                    resultMode = MeasureSpec.AT_MOST;  
                }  
                break;  
          
            // 当父view的模式为UNSPECIFIED时，父容器不对view有任何限制，要多大给多大
            // 多见于ListView、GridView  
            case MeasureSpec.UNSPECIFIED:  
                if (childDimension >= 0) {  
                    // 子view大小为子自身所赋的值  
                    resultSize = childDimension;  
                    resultMode = MeasureSpec.EXACTLY;  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {  
                    // 因为父view为UNSPECIFIED，所以MATCH_PARENT的话子类大小为0  
                    resultSize = 0;  
                    resultMode = MeasureSpec.UNSPECIFIED;  
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
                    // 因为父view为UNSPECIFIED，所以WRAP_CONTENT的话子类大小为0  
                    resultSize = 0;  
                    resultMode = MeasureSpec.UNSPECIFIED;  
                }  
                break;  
            }  
            return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
        }

```

关于getChildMeasureSpec()里对子View的测量模式 & 大小的判断逻辑有点复杂；
别担心，我已帮大家总结好。具体请看下表：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi16.webp)

其中的规律总结：（以子View为标准，横向观察）

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi17.webp)

**由于UNSPECIFIED模式适用于系统内部多次measure情况，很少用到，故此处不讨论**

**区别于顶级View（即DecorView）的测量规格MeasureSpec计算逻辑：取决于 自身布局参数 & 窗口尺寸**

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/zi18.webp)



