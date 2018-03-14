---
title: Javascript
date: 2016-12-07 12:59:25
tags: Javascript
---
# JavaScript面向对象基础

## _proto、prototye和constructor

- 类的静态属性和实例属性

## call()和apply()

1、每个函数都包含两个非继承而来的方法：apply()和call()。
2、用途相同，都是在特定的作用域中调用函数。
3、接收参数方面不同，apply()接收两个参数，一个是函数运行的作用域(this)，另一个是参数数组。
call()方法第一个参数与apply()方法相同，但传递给函数的参数必须列举出来。

**示例一：call()和apply()基本使用**:

```javascript
function sum(num1, num2) {
   return num1 + num2;
}

console.log(sum.call(window, 10, 10)); //20
console.log(sum.apply(window,[10,20])); //30
```

**示例二：更改函数的作用域**:

```javascript

window.firstName = "diz";
window.lastName = "song";
var myObject = { firstName: "my", lastName: "Object" };
function HelloName() {
　　console.log("Hello " + this.firstName + " " + this.lastName, " glad to meet you!");
}
HelloName.call(window); //huo .call(this);
HelloName.call(myObject);

```

## Array.prototype.slice.call(arguments,0)

> 把类数组对象转换成一个真正的数组。

在Array.prototype.slice.call(arguments,0)中，Array.prototype.slice调用的是Array的原型方法，对于正真的数组是有slice()方法，但是对于像arguments或者自己定义的一些类数组对象虽然存在length等若干属性，但是并没有slice()方法，所以对于这种类数组对象就得使用原型方法来使用slice()方法，即Array.prototype.slice（如果在自定义中的类数组对象中自定义了slice()方法，那么自然可以直接调用）。

Array.prototype.slice.call(arguments)能将具有length属性的对象转成数组，除了IE下的节点集合（因为ie下的dom对象是以com对象的形式实现的，js对象与com对象不能进行转换）。

## 属性与方法封装

JavaScript函数级作用域，声明在函数内部的变量以及方法在外界是访问不到的。通过此特性即可创建类的私有变量以及私有方法。

## 私有方法

## 类的静态方法

## 闭包

## 创建对象安全模式

## 继承

### 类式继承——子类的原型对象

1、判断对象与类之间的继承关系
instanceof是通过判断对象的prototype链来确定这个对象是否是某个类的实例，而不关心对象与类的自身结构。

**缺点**：

1、如果父类中的共有属性是引用类型，就会在子类中被所有实例共用，因此一个子类的实例更改子类原型从父类构造函数中继承来的共有属性就会直接影响到其他子类。
2、由于子类实现的继承是靠其原型prototype对父类的实例化实现的，因此在创建父类的时候，是无法向父类传递参数的，因而在实例化父类的时候也无法对父类构造函数内属性进行初始化。

### 构造函数继承

```javascript
  SuperClass.call(this,id);
```

- call()方法

可以更改函数的作用环境。

> 这种类型的继承没有涉及原型prototype，所以父类的原型方法自然不会被子类继承。    

# 单例模式

- 命名空间

- 模块分明

# 观察者模式
	也被称为发布-订阅者模式或消息机制，定义了以一种依赖关系，解决了主体对象与观察者之间功能的耦合。  
  - 观察者
  	消息容器、订阅消息方法、取消订阅的消息方法、发送订阅的消息方法。
  - 主体对象
> 数组splice()


# 状态模式
  当一个对象的内部状态发生变化时，会导致其行为的改变，这看起来像是改变了对象。


# 策略模式
  将定义的一组算法封装起来，使其相互之间可以替换。封装的算法具有一定独立性，不会随客服端变化而变化。


> 状态模式与策略模式的区别


策略模式应用：
1、jQuery.animate();
2、easing.js


# 职责链模式
解决请求的发送者与请求的接受者之间的耦合，通过职责链上的多个对象分解请求流程，实现请求在多个对象之间的传递，直到最后一个对象完成请求的处理。

# 命令模式
将请求与实现解耦并封装成独立的对象，从而使不同的请求对客户端的实现参数化。

> apply()方法

# 访问者模式
针对于对象结构中的元素，定义在不改变该对象的前提下访问结构中元素的新方法。

