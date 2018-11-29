---
title: 两个对象的equals hashcode的关系
date: 2018-11-29 14:58:59
tags:
categories:
- JAVA
---
HashSet和HashMap一直都是JDK中最常用的两个类，**HashSet要求不能存储相同的对象，HashMap要求不能存储相同的键。** 
那么Java运行时环境是如何判断HashSet中相同对象、HashMap中相同键的呢？当存储了“相同的东西”之后Java运行时环境又将如何来维护呢？ 

在研究这个问题之前，首先说明一下JDK对equals(Object obj)和hashcode()这两个方法的定义和规范： 
**在Java中任何一个对象都具备equals(Object obj)和hashcode()这两个方法，因为他们是在Object类中定义的。 
equals(Object obj)方法用来判断两个对象是否“相同”，如果“相同”则返回true，否则返回false。 
hashcode()方法返回一个int数，在Object类中的默认实现是“将该对象的内部地址转换成一个整数返回”。** 
接下来有两个个关于这两个方法的重要规范(我只是抽取了最重要的两个,其实不止两个)： 
#### 规范1：
**若重写equals(Object obj)方法，有必要重写hashcode()方法，确保通过equals(Object obj)方法判断结果为true的两个对象具备相等的hashcode()返回值。**
说得简单点就是：**“如果两个对象相同，那么他们的hashcode应该相等”。**不过请注意：这个只是规范，如果你非要写一个类让equals(Object obj)返回true而hashcode()返回两个不相等的值，编译和运行都是不会报错的。不过这样违反了Java规范，程序也就埋下了BUG。 
#### 规范2：
**如果equals(Object obj)返回false，即两个对象“不相同”，并不要求对这两个对象调用hashcode()方法得到两个不相同的数。**
说的简单点就是：**“如果两个对象不相同，他们的hashcode可能相同”。 **
**根据这两个规范，可以得到如下推论： 
1、如果两个对象equals，Java运行时环境会认为他们的hashcode一定相等。 
2、如果两个对象不equals，他们的hashcode有可能相等。 
3、如果两个对象hashcode相等，他们不一定equals。 
4、如果两个对象hashcode不相等，他们一定不equals。 **

这样我们就可以推断Java运行时环境是怎样判断HashSet和HastMap中的两个对象相同或不同了。
我的推断是：**先判断hashcode是否相等，再判断是否equals。 **

#### 测试程序
首先我们定义一个类，重写hashCode()和equals(Object obj)方法 

```
class A {  
  
    @Override  
    public boolean equals(Object obj) {  
        System.out.println("判断equals");  
        return false;  
    }  
  
    @Override  
    public int hashCode() {  
        System.out.println("判断hashcode");  
        return 1;  
    }  
}  
```

然后写一个测试类，代码如下： 

```
public class Test {    
    
    public static void main(String[] args) {    
        Map<A,Object> map = new HashMap<A, Object>();    
        map.put(new A(), new Object());    
        map.put(new A(), new Object());    
            
        System.out.println(map.size());    
    }    
        
}    
```

运行之后打印结果是： 

判断hashcode 
判断hashcode 
判断equals 
2 

可以看出，Java运行时环境会调用new A()这个对象的hashcode()方法。其中： 
打印出的第一行“判断hashcode”是第一次map.put(new A(), new Object())所打印出的。 
接下来的“判断hashcode”和“判断equals”是第二次map.put(new A(), new Object())所打印出来的。 

那么为什么会是这样一个打印结果呢？我是这样分析的： 
1、当第一次map.put(new A(), new Object())的时候，Java运行时环境就会判断这个map里面有没有和现在添加的 new A()对象相同的键，判断方法：调用new A()对象的hashcode()方法，判断map中当前是不是存在和new A()对象相同的HashCode。显然，这时候没有相同的，因为这个map中都还没有东西。所以这时候hashcode不相等，则没有必要再调用 equals(Object obj)方法了。参见推论4（如果两个对象hashcode不相等，他们一定不equals） 
2、当第二次map.put(new A(), new Object())的时候，Java运行时环境再次判断，这时候发现了map中有两个相同的hashcode（因为我重写了A类的hashcode()方 法永远都返回1），所以有必要调用equals(Object obj)方法进行判断了。参见推论3（如果两个对象hashcode相等，他们不一定equals），然后发现两个对象不equals（因为我重写了equals(Object obj)方法，永远都返回false）。 
3、这时候判断结束，判断结果：两次存入的对象不是相同的对象。所以最后打印map的长度的时候显示结果是：2。 

改写程序如下：

```
class A {  
  
    @Override  
    public boolean equals(Object obj) {  
        System.out.println("判断equals");  
        return true;  
    }  
  
    @Override  
    public int hashCode() {  
        System.out.println("判断hashcode");  
        return 1;  
    }  
}  
  
  
public class Test {  
  
    public static void main(String[] args) {  
        Map<A,Object> map = new HashMap<A, Object>();  
        map.put(new A(), new Object());  
        map.put(new A(), new Object());  
          
        System.out.println(map.size());  
    }  
      
} 
```

运行之后打印结果是： 

判断hashcode 
判断hashcode 
判断equals 
1 

显然这时候map的长度已经变成1了，因为Java运行时环境认为存入了两个相同的对象。原因可根据上述分析方式进行分析。 

以上分析的是HashMap，其实HashSet的底层本身就是通过HashMap来实现的，所以他的判断原理和HashMap是一样的，也是先判断hashcode再判断equals。 

所以：写程序的时候应尽可能的按规范来，不然在不知不觉中就埋下了bug！  

