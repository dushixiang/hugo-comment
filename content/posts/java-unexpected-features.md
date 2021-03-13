---
title: "Java的奇技淫巧"
categories: [ "Java" ]
tags: [ "Java","开发" ]
draft: false
slug: "java-unexpected-features"
date: "2021-03-13T13:49:38+08:00"
---



**Java**是一种广泛使用的计算机编程语言、面向对象、泛型编程的特性，广泛应用于企业级Web应用开发和移动应用开发。

1995年3月23日**Sun**公司发布了**Java**，至今已有近26年，可以说是一门十分成熟的开发语言了，但在某些不为人知的地方存在着一些意料之外的特性。

### Java的保留关键字 goto和const 

在**Java**里面没有`goto`这个功能，但它作为保留字是无法当做变量来使用的，`const`也是同样。

```java
int goto = 0;
int const = 0;
```

上面这两行代码的写法存在问题，无法正常编译通过。

### Java标签Label

上面说了在**Java**里面没有`goto`这个功能，但为了处理多重循环引入了Label，目的是为了在多重循环中方便的使用 break 和coutinue ，但好像在其他地方也可以用。

```java
outerLoop:
while (true) {
	System.out.println("I'm the outer loop");
	int i = 0;
	while (true) {
		System.out.println("I am the inner loop");
		i++;
		if (i >= 3) {
			break outerLoop;
		}
	}
}
System.out.println("Complete the loop");

// 输出
I'm the outer loop
I am the inner loop
I am the inner loop
I am the inner loop
Complete the loop
```

```java
test:
{
	System.out.println("hello");
	if (true) {
    break test; // works
  }
  System.out.println("world");
}

// 输出
hello
```

```java
test:
if (true) {
	System.out.println("hello");
	if (true) {
		break test; // works
	}
	System.out.println("world");
}
// 输出
hello
```

```java
test:
try {
	System.out.println("hello");
	if (true) {
		break test; // works
	}
	System.out.println("world");
} finally {
}
// 输出
hello
```

### Integer的是否相等问题

日常开发使用到Java基本数据类型是不可避免的一件事，但它却包含了一些很容易犯错的点，踩过一些坑的同学可能了解Java基本包装类型的常量池技术，例如`Integer`就具有数值`[-128，127] `的相应类型的缓存数据，但下面定义的4个变量是否相等你是否能说的出来呢？

```java
Integer i1 = 127;
Integer i2 = Integer.valueOf(127);
Integer i3 = Integer.parseInt("127");

Integer i4 = new Integer(127);

System.out.println(i1 == i2);
// true
System.out.println(i1 == i3);
// true
System.out.println(i2 == i3);
// true

System.out.println(i4 == i1);
// false
System.out.println(i4 == i2);
// false
System.out.println(i4 == i3);
// false
```



1. `Integer i1 = 127;` **Java** 在编译的时候会直接将代码优化为 `Integer i1=Integer.valueOf(127);`，从而使用常量池中的对象。

2. `Integer i2 = Integer.valueOf(127);` 会从Integer缓存中获取。

   `Integer.valueOf`源码如下。

   ```java
   /**
    * 会从缓存数组中获取范围为[-128, 127]的值，如果没有就创建一个新的对象。
    */
   public static Integer valueOf(int i) {
   	if (i >= IntegerCache.low && i <= IntegerCache.high)
   		return IntegerCache.cache[i + (-IntegerCache.low)];
   	return new Integer(i);
   }
   ```

3. `Integer.parseInt("127");` 返回值是 int数字，在`[-128, 127]`范围的会从常量池中获取。

4. `new Integer(127);` 创建一个新的对象。

Integer对象的缓存值的大小范围是在`[-128 127]`区间。这意味着当我们在此数值范围内操作时，`“==”`比较能正常返回结果。但当数值不在此范围，对象相等的比较是否正常返回结果就很难说了。所以为了安全，在使用对象数值进行比较相等时，请使用`“.equals()”`，而不是依赖于`“==”`比较相等，除非你非常肯定这种做法没有错误。

思考一个问题，当变量的数值均为`128`时会输出什么结果呢？

### Double.MIN_VALUE 竟然不是负数？

我们可能是受了太多`Integer`和`Long`的影响，理所当然的认为`Double.MIN_VALUE`应该是一个负数，但实际上`Double.MIN_VALUE`是非0非负的最小值**4.9E-324**，`Float`也是同样。如果你想要最小的Double类型的数字，请使用`-Double.MAX_VALUE`。

```java
System.out.println(Double.MAX_VALUE);
// 1.7976931348623157E308
System.out.println(Double.MIN_VALUE);
// 4.9E-324
System.out.println(Float.MAX_VALUE);
// 3.4028235E38
System.out.println(Float.MIN_VALUE);
// 1.4E-45
```

### 使用 for 做死循环？

提到死循环大家都会想到`while(true)`，但还有另一种写法 `for(;;)`，在字节码层面他们会被编译为同样的内容，一些上古`C/C++`程序员写Java的时候还会使用`for(;;)`做死循环，遇到这样的朋友一定要珍惜哦。

```java
while (true){
  // do something
}
```

```java
for(;;){
	// do something
}
```

### 下划线也能当做变量？

```java
Object _ = new Object();
System.out.println(_.toString());
```

在Java8中还能使用下划线当做变量，但在Java9之后就标记为不再支持，请珍惜和它的最后这段时光吧。

### WTF!!! finally还能返回内容？

```java
public class Finally {

    public int fun() {
        try {
            return 0;
        } finally {
            return 1;
        }
    }

    public static void main(String[] args) {
        Finally aFinally = new Finally();
        System.out.println(aFinally.fun());
      	// 1
    }
}
```

面试的时候finally返回值坑了多少英雄好汉？

### 一个类可以有几个static代码块？

```java
public class Static {

    static {
        System.out.println("hello");
    }

    static {
        System.out.println("world");
    }

    static {
        System.out.println("java");
    }

    public static void main(String[] args) {

    }
}

// 输出
hello
world
java
```

Java🐂🍺

### final 类型一定要声明的时候进行初始化吗？

关于final类型的变量不可修改是每一个Java开发的常识，声明变量的时候就要对其进行初始化，但真的是这样的吗？

```java
public class Final {
    public static void main(String[] args) {
        final int a;
        if (args != null) {
            a = 0;
        } else {
            a = 1;
        }
        System.out.println(a);
        // 0 
    }
}
```

是否有点违背认知了？