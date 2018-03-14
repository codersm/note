---
title: "关于java中Double类型的运算精度问题"
date: 2016-07-05 18:25:18
tags: Java
---

1.[问题提出](#问题提出)

2.[使用BigDecimal对double进行运算](#使用BigDecimal对double进行运算)

3.[Double保留两位小数的方法](#Double保留两位小数的方法)

<!--more-->

# Double运算

在Java中简单浮点数类型float和double不能进行运算,float和double只能用来做科学计算或者是
工程计算，在商业计算中我们要用java.math.BigDecimal。对于货币精度已知的情况，可以使用
整型。

## 问题提出

```java
public class DoubleFormat {

  public static void main(String[] args) {
      double d1 = 1.0;
      double d2 = 0.58;
      System.out.println(d1 - d2);
  }
}
```

> 执行结果：0.42000000000000004

## 使用BigDecimal对double进行运算

```java
public class Arith {

  //默认除法运算精度
  private static final int DEF_DIV_SCALE = 10;

  //这个类不能实例化
  private Arith() {}

  /**
   * 提供精确的加法运算。
   *
   * @param v1 被加数
   * @param v2 加数
   * @return 两个参数的和
   */
  public static double add(double v1, double v2) {
      BigDecimal b1 = new BigDecimal(Double.toString(v1));
      BigDecimal b2 = new BigDecimal(Double.toString(v2));
      return b1.add(b2).doubleValue();
  }

  /**
   * 提供精确的减法运算。
   *
   * @param v1 被减数
   * @param v2 减数
   * @return 两个参数的差
   */
  public static double sub(double v1, double v2) {
      BigDecimal b1 = new BigDecimal(Double.toString(v1));
      BigDecimal b2 = new BigDecimal(Double.toString(v2));
      return b1.subtract(b2).doubleValue();
  }

  /**
   * 提供精确的乘法运算。
   *
   * @param v1 被乘数
   * @param v2 乘数
   * @return 两个参数的积
   */
  public static double mul(double v1, double v2) {
      BigDecimal b1 = new BigDecimal(Double.toString(v1));
      BigDecimal b2 = new BigDecimal(Double.toString(v2));
      return b1.multiply(b2).doubleValue();
  }

  /**
   * 提供（相对）精确的除法运算，当发生除不尽的情况时，精确到
   * 小数点以后10位，以后的数字四舍五入。
   *
   * @param v1 被除数
   * @param v2 除数
   * @return 两个参数的商
   */
  public static double div(double v1, double v2) {
      return div(v1, v2, DEF_DIV_SCALE);
  }

  /**
   * 提供（相对）精确的除法运算。当发生除不尽的情况时，由scale参数指
   * 定精度，以后的数字四舍五入。
   *
   * @param v1    被除数
   * @param v2    除数
   * @param scale 表示表示需要精确到小数点以后几位。
   * @return 两个参数的商
   */
  public static double div(double v1, double v2, int scale) {
      if (scale < 0) {
          throw new IllegalArgumentException(
                  "The scale must be a positive integer or zero");
      }
      BigDecimal b1 = new BigDecimal(Double.toString(v1));
      BigDecimal b2 = new BigDecimal(Double.toString(v2));
      return b1.divide(b2, scale, BigDecimal.ROUND_HALF_UP).doubleValue();
  }

  /**
   * 提供精确的小数位四舍五入处理。
   *
   * @param v     需要四舍五入的数字
   * @param scale 小数点后保留几位
   * @return 四舍五入后的结果
   */
  public static double round(double v, int scale) {
      if (scale < 0) {
          throw new IllegalArgumentException(
                  "The scale must be a positive integer or zero");
      }
      BigDecimal b = new BigDecimal(Double.toString(v));
      BigDecimal one = new BigDecimal("1");
      return b.divide(one, scale, BigDecimal.ROUND_HALF_UP).doubleValue();
  }
}
```

## Double保留两位小数的方法

```java
       double d1 = 1.0;

       DecimalFormat df = new DecimalFormat("#.00");
       System.out.println(df.format(d1));

       System.out.println(String.format("%.2f", d1));
```
