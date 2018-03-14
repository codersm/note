---
title: Java字符串详解
date: 2017-09-12 09:09:41
tags: Java
---
# Java基础系列—字符串

> **可以证明，字符串操作是计算机程序设计中最常见的行为。**

## 不可变String

String对象是不可变的。查看JDK文档你就会发现，String类中每一个看起来会修改String值的方法，实际上都是创建了一个全新的String对象，以包含修改后的字符串内容，而最初的String对象则丝毫未变。

```java
public class Immutable {

    public static String upcase(String s){
        return  s.toUpperCase();
    }

    public static void main(String[] args) {
        String q = "howdy";
        System.out.println(q);
        String qq = upcase(q);
        System.out.println(qq);
        System.out.println(q);
    }
}
```

## 重载 “+” 与StringBuilder

不可变性会带来一定的效率问题。用于String的“+”与“+=”是Java中仅有的两个重载过的操作符，而Java并不允许程序重载任何操作符。

操作符 “+” 可以用来连接String：

```java
public class Concatenation {

    public static void main(String[] args) {
        String mango = "mango";
        String s = "abc" + mango;
        System.out.println(s);
    }
}
```

想看看以上代码到底是如何工作的吗，可以用JDK自带的工具javap来反编译以上代码，可以得到以下的字节码：

![字节码](https://upload-images.jianshu.io/upload_images/2889214-8ab86b0285d9dcbf.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编译器自动引入了java.lang.StringBuilder类（因为它更高效），通过调用append方法将字符串连接起来，最后调用toString方法生成最终结果。

### 避免循环体内使用 "+="

```java
public String implicit(String[] fields) {
    String result = "";
    for (int i = 0; i < fields.length; i++) {
        result += fields[i];
    }
    return result;
}
```

通过反编译得如下字节码：

![Jietu20180311-174853.jpg](https://upload-images.jianshu.io/upload_images/2889214-8cd5414821484264.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

StringBuilder是在循环体内构造的，这意味着每经过循环一次，就会创建一个新的StringBuilder对象。

### 无意识的递归

```java
public class InfiniteRecursion {

    @Override
    public String toString() {
        // 打印InfiniteRecursion对象的内存地址
        return " InfiniteRecursion adress: " + this + "\n";
    }

    public static void main(String[] args) {
        InfiniteRecursion infiniteRecursion  = new InfiniteRecursion();
        System.out.println(infiniteRecursion);
    }
}
```

运行以上程序出现如下结果：

![运行结果](https://upload-images.jianshu.io/upload_images/2889214-b41c42bd105ac064.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`" InfiniteRecursion adress: " + this`这里发生了自动类型转换，由InfiniteRecursion类型转换成String类型。因为编译器发现String对象后面跟着一个 “+”，而后面的对象不是String，编译器调用this.toString()方法进行类型转换，因此发生了递归调用。

如果你真的想要打印出对象的内存地址，应该调用Object.toString()方法。所以不该使用this，而是应该调用super.toString()方法。

## 格式化输出

Java SE5推出了C语言中printf()风格的格式化输出这一功能，不需要使用重载的 “+”操作符来连接引用号内的字符串或者字符串常量，而是使用特殊的占位符来表示数据将来的位置。

```java
public static void main(String[] args) {
    int x = 5;
    double y = 5.332;

    // the old way
    System.out.println("Row 1：[" + x + " " + y + "]");

    // the new way
    System.out.printf("Row 1：[%d %f]\n", x, y);
    
    // or
    System.out.format("Row 1：[%d %f]\n", x, y);
}
```
运行以上程序，首先将x的值插入到%d的位置，然后将y的值插入%f的位置。这些占位符被称为**格式修饰符**，它们不但说明了将插入什么类型的变量，以及如何对其格式化。

### Formatter类

在Java中，所有新的格式化功能都由java.util.Formatter类处理。可以将Formatter类看作一个翻译器，它将你的格式化字符串与数据翻译成需要的结果。

```java
Formatter formatter = new Formatter(System.out);
formatter.format("Row 1：[%d %f]\n", x, y);
```

### String.format()

String.format()是一个static方法，它接受与Formatter.format()方法一样的参数，但返回一个String对象。

### 格式化说明符

在插入数据时，如果想要控制空格与对齐，你需要更精细复杂的格式修饰符。以下是其抽象的语法：

```
%[argument_index$][flags][width][.precision]conversion
```

| 字段             | 说明                                                         |
| -------------- | ---------------------------------------------------------- |
| argument_index | 需要将参数列表中第几个参数进行格式化                                         |
| flags          | 一些特殊的格式，比如‘-’代表向左对齐                                        |
| width          | 输出的最小的宽度，适用于所有的参数类型                                        |
| [.precision]   | 参数为String，则表示打印String输出字符的最大数量；参数为float，则表示小数点最大位数。不使用于int |
| conversion     | 接受的参数类型，如s代表后面接String类型的参数；d代表接int型的参数                     |

[参数详细说明](http://blog.csdn.net/XIAXIA__/article/details/41720485)

## 正则表达式

正则表达式是一种强大而灵活的文本处理工具。使用正则表达式，我们能够以编程的方式，构造复杂的文本模式，并对输入的字符串进行搜索。一旦找到了匹配这些模式的部分，你就能够随心所欲地对它们进行处理。

> **Java语言与其他语言相比对反斜杠`\`有不同的处理**：
> 
> 在其他语言中，`\\`表示“我想要在正则表达式中插入一个普通的（字面上的）反斜杠，请不要给它做任何特殊的意义。” 而在Java中，`\\`的意思是“我要插入一个正则表达式的反斜杠，所以其后的字符具有特殊的意义。” 
> 
> 例如，如果你想表示一位数字，那么正则表达式应该是`\\d`。如果你想插入一个普通的反斜杠，则应该这样`\\\\`。不过换行和制表符之类的东西只需使用单斜杠线：`\n\t`。


### String类支持正则表达式的方法

```java
public String[] split(String regex)

public String[] split(String regex, int limit) 

public String replaceFirst(String regex, String replacement)

public String replaceAll(String regex, String replacement) 

public boolean matches(String regex) 
```

### Pattern和Matcher

一般来说，比起功能有限的String类，我们更愿意构造功能强大的正则表达式对象。通过`Pattern.complie()`方法来编译你的正则表达式即可。它会根据你的String类型的正则表达式生成一个Pattern对象。接下来，把你想要检索的字符串传入Pattern对象的matcher()方法，matcher()方法会生成一个Matcher对象，它有很多功能可用。

```java
public class RegexExpression {

    public static void main(String[] args) {
        String phone = "18926119073";
        Pattern pattern = Pattern.compile("1([358][0-9]|4[579]|66
                |7[0135678]|9[89])[0-9]{8}");
        Matcher matcher = pattern.matcher(phone);
        System.out.println(matcher.matches());
    }
}
```

#### 组

组是用括号划分的正则表达式，可以根据组的编号来引用某个组。组号为0表示整个表达式，组号1表示被第一对括号括起的组，依次类推。因此，在下面这个表达式：

```
A(B(C))D
```
中有三个组：组0是ABCD，组1是BC，组2是C。

Mather提供了很多有用的方法具体使用查看API,使用Mather的替换方法可以实现隐藏手机号中间数字、隐藏用户名等。

```java
String phone = "18926119073";
Pattern pattern = Pattern.compile("(\\d{3})\\d{4}(\\d{4})");
Matcher matcher = pattern.matcher(phone);
phone = matcher.replaceAll("$1****$2");
System.out.println(phone);
```

### 用正则表达式扫描

Java SE5新增类Scanner类，它可以大大减轻扫描输入的工作。

```java
public class ScannerRead {

    public static void main(String[] args) {
        Scanner scanner = new Scanner("Sir Robin of Camelot
        \n22 1.61803"));
        System.out.println("What is your name?");
        String name = scanner.nextLine();
        System.out.println(name);
        System.out.println("(input: <age> <double>)");
        System.out.println(scanner.nextInt()+" "+ scanner.nextDouble());
    }
}
```

> Scanner的构造期可以接受任何类型的输入对象，包括File对象、InputStream、String或者Readable对象。
> 
> Scanner所有的输入、分词以及翻译的操作都隐藏在不同类型的next方法中，普通的next()方法返回下一个String，所有的基本类型（除char之外）都有对应的next方法，包括BigDecimal和BigInteger。所有的next方法，只有在找到一个完整的分词之后才会返回，hasNext方法用以判断下一个输入分词是否所需的类型。


在默认的情况下，Scanner根据空白符对输入进行分词，但是你可以用正则表达式指定自己所需的定界符：

```java
public class ScannerDelimiter {

    public static void main(String[] args) {
        Scanner scanner = new Scanner("12,42,78,99");
        scanner.useDelimiter(",");
        while (scanner.hasNextInt()){
            System.out.println(scanner.nextInt());
        }
    }
}
```

除了能够扫描基本类型之外，你还可以使用自定义的正则表达式进行扫描。如下所示：

```java
public class ThreatAnalyzer {

    static String threatData =
            "58.27.82.161@02/10/2005\n" +
            "124.45.82.161@02/10/2005\n" +
            "58.27.82.161@02/10/2005\n" +
            "72.27.82.161@02/10/2005\n" +
             "[Next log section with different data format]";

    public static void main(String[] args) {
        Scanner scanner = new Scanner(threatData);
        String pattern = "(\\d+[.]\\d+[.]\\d+[.]\\d+)@(\\d{2}/\\d{2}/\\d{4})";
        while (scanner.hasNext(pattern)) {
            scanner.next(pattern);
            MatchResult match = scanner.match();
            String ip = match.group(1);
            String date = match.group(2);
            System.out.format("Threat on %s from %s\n", date, ip);
        }
    }
}
```