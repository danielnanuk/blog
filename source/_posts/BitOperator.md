---
title: Java中的位操作
date: 2018-06-08 13:58:21
tags:
---
## 位操作
Java中的位操作有:
1. 取反操作:``~``
2. 或操作: ``|``
3. 与操作: ``&``
4. 异或操作: ``^``
3. 移位操作:
    1. 左移: ``<<``
    2. 右移: ``>>``
    3. 无符号右移： ``>>>``

Java中的位操作只支持整型(`byte`,`short`,`int`,`long`)

### 最大值与最小值
开始看位操作之前, 我们先来看看最大值与最小值, 以Integer为例:
```java
    /**
     * A constant holding the minimum value an {@code int} can
     * have, -2<sup>31</sup>.
     */
    @Native public static final int   MIN_VALUE = 0x80000000;

    /**
     * A constant holding the maximum value an {@code int} can
     * have, 2<sup>31</sup>-1.
     */
    @Native public static final int   MAX_VALUE = 0x7fffffff;
```

Java中的Integer为32位, 其中要表示正数, 负数以及0, 第32位为符号位.
```java
    Function<Integer, String> toBinaryString = (integer) -> {
          StringBuilder builder = new StringBuilder(Integer.toBinaryString(integer));
          int length = builder.length();
          for (int i = length; i < 32; i++) {
            builder.insert(0, "0");
          }
          return builder.toString();
    };
    
    System.out.println(toBinaryString.apply(Integer.MAX_VALUE)); // 01111111111111111111111111111111
    
    System.out.println(toBinaryString.apply(Integer.MIN_VALUE)); // 10000000000000000000000000000000
    
    System.out.println(toBinaryString.apply(0)); // 00000000000000000000000000000000
    
    System.out.println(toBinaryString.apply(-1)); // 11111111111111111111111111111111
```
对于Integer来说, 取值范围-2<sup>31</sup> ~ 2<sup>31</sup> - 1, 由于0的存在, Integer.MAX_VALUE 的绝对值比 Integer
.MIN_VALUE的绝对值小1.

### 取反操作

```java
    int a = 10;
    System.out.println(toBinaryString.apply(a)); // 00000000000000000000000000001010
    int b = ~10;
    System.out.println(toBinaryString.apply(b)); // 11111111111111111111111111110101
    System.out.println(b); // -11
```
操作数的第n位为1, 那么结果的第n位为0, 反之. 对于任何任何数a都有: ``a + ~a = -1``.

### 或操作
```java
    int a = 10;
    System.out.println(toBinaryString.apply(a)); // 00000000000000000000000000001010
    int b = 3;
    System.out.println(toBinaryString.apply(b)); // 00000000000000000000000000000011
    int c = a | b;
    System.out.println(toBinaryString.apply(c)); // 00000000000000000000000000001011
``` 
第一个操作数的的第n位于第二个操作数的第n位只要有一个是1, 那么结果的第n为也为1, 否则为0.

### 与操作
```java
    int a = 10;
    System.out.println(toBinaryString.apply(a)); // 00000000000000000000000000001010
    int b = 3;
    System.out.println(toBinaryString.apply(b)); // 00000000000000000000000000000011
    int c = a & b;
    System.out.println(toBinaryString.apply(c)); // 00000000000000000000000000000010
```
第一个操作数的的第n位于第二个操作数的第n位如果都是1, 那么结果的第n为也为1, 否则为0.

### 异或操作

```java
    int a = 10;
    System.out.println(toBinaryString.apply(a)); // 00000000000000000000000000001010
    int b = 3;
    System.out.println(toBinaryString.apply(b)); // 00000000000000000000000000000011
    int c = a & b;
    System.out.println(toBinaryString.apply(c)); // 00000000000000000000000000001001
```
第一个操作数的的第n位于第二个操作数的第n位相反, 那么结果的第n为也为1, 否则为0.

### 左移操作
```javas
    int a = 10;
    System.out.println(toBinaryString.apply(a)); // 00000000000000000000000000001010
    int b = a << 1;
    System.out.println(toBinaryString.apply(b)); // 00000000000000000000000000010100
    System.out.println(b); // 20
```
对于正数来说, 左移时最高位始终为0, 低位补0.

位于负数来说, 左移时最高位始终为1, 低位补0.

在未出现溢出的情况下, 左移1位相当于原来的值乘以2.

要注意, 在Java中进行左移操作时, 移动的位数若大于数据的最大位数减1时, 会对操作次数进行精度取余, 如下
```java
    int a = 10;
    int b = 32; // 若b值大于31, 实际左移位数是b对32求余的结果, 这里实际不进行位移, 结果仍是10
    System.out.println(a << b);
    System.out.println(a << b % 32);
```

### 右移操作
```java
    int a = 10;
    System.out.println(toBinaryString.apply(a)); // 00000000000000000000000000001010
    int b = a >> 1;
    System.out.println(toBinaryString.apply(b)); // 00000000000000000000000000000101
    System.out.println(b); // 5
```
对于正数来说, 右移时最高位始终为0, 高位补0.

位于负数来说, 右移时最高位始终为1, 高位补0.

在未过界情况下, 对于偶数右移a左移等于`a >> 1 = a / 2`, 对于奇数b右移等于`b >> 1 = (b - 1) / 2`.

### 无符号右移
```java
    int a = -10;
    System.out.println(toBinaryString.apply(a));
    int b = a >>> 1;
    System.out.println(toBinaryString.apply(b));
    System.out.println(b);
```
不管最高位符号位向右移动, 高位补0.









