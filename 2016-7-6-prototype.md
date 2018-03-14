---
title: JavaScript中的prototype
date: 2016-07-06T14:50:34.000Z
tags: JavaScript
---

对于初学 JavaScript 的人来说 prototype 是一种很神奇的特性，而事实上，prototype 对于 JavaScript 的意义重大，prototype 不仅仅是一种管理对象继承的机制，更是一种出色的设计思想。 <!-- more -->

# 1、什么是prototype

JavaScript中对象的prototype属性，可以返回对象类型原型的引用。这是一个相当拗口的解释，要理解它，先要正确理解对象类型(Type)以及原型(prototype)的概念。

在现实生活中，我们常常说，某个东西是以另一个东西为原型创作的。这两个东西可以是同一个类型，也可以是不同类型。习语"依葫芦画瓢"，这里的葫芦就是原型，而瓢就是类型，用JavaScript的prototype来表示就是"瓢.prototype =某个葫芦"或者"瓢.prototype= new 葫芦()"。

# 2、propertype使用技巧

JavaScript为每一个类型(Type)都提供了一个prototype属性，将这个属性指向一个对象，这个对象就成为了这个类型的"原型"，这意味着由这个类型所创建的所有对象都具有这个原型的特性。另外，JavaScript的对象是动态的，原型也不例外，给prototype增加或者减少属性，将改变这个类型的原型，这种改变将直接作用到由这个原型创建的所有对象上。

- 设定对象的属性默认值 如果给某个对象的类型的原型添加了某个名为a的属性，而这个对象本身又有一个名为a的同名属性，则在访问这个对象的属性a时，对象本身的属性"覆盖"了原型属性，但是原型属性并没有消失，当你用delete运算符将对象本身的属性a删除时，对象的原型属性就恢复了可见性。

  ```javascript
  function Point(x,y){
  this.x = x;
  this.y = y;
  }
  var p1 = new Point(1,2);
  var p2 = new Point(3,4);
  Point.prototype.z = 0; //动态为Point的原型添加了属性
  alert(p1.z);
  alert(p2.z); //同时作用于Point类型创建的所有对象
  ```

- 设置对象的属性为只读，从而避免它被改写。示例如下：

```javascript
function Point(x, y)
{
if(x) this.x = x;
if(y) this.y = y;
}
Point.prototype.x = 0;
Point.prototype.y = 0;

function LineSegment(p1, p2)
{
//私有成员
var m_firstPoint = p1;
var m_lastPoint = p2;
var m_width = {
valueOf : function(){return Math.abs(p1.x - p2.x)},
toString : function(){return Math.abs(p1.x - p2.x)}
}
var m_height = {
valueOf : function(){return Math.abs(p1.y - p2.y)},
toString : function(){return Math.abs(p1.y - p2.y)}
}
//getter
this.getFirstPoint = function()
{
return m_firstPoint;
}
this.getLastPoint = function()
{
return m_lastPoint;
}

this.length = {
  valueOf : function(){return Math.sqrt(m_width*m_width + m_height*m_height)},
  toString : function(){return Math.sqrt(m_width*m_width + m_height*m_height)}
  }
}
var p1 = new Point;
var p2 = new Point(2,3);
var line1 = new LineSegment(p1, p2);
var lp = line1.getFirstPoint();
lp.x = 100; //不小心改写了lp的值，破坏了lp的原始值而且不可恢复
alert(line1.getFirstPoint().x);
alert(line1.length); //就连line1.lenght都发生了改变
```

将this.getFirstPoint()改写为下面这个样子：

```javascript
  this.getFirstPoint = function()
  {
    function GETTER(){};
    GETTER.prototype = m_firstPoint;
    return new GETTER();
  }
```

实际上，将一个对象设置为一个类型的原型，相当于通过实例化这个类型，为对象建立只读副本，在任何时候对副本进行改变，都不会影响到原始对象，而对原始对象进行改变，则会影响到副本，除非被改变的属性已经被副本自己的同名属性覆盖。

> 则可以避免这个问题，保证了m_firstPoint属性的只读性。

- 利用prototype来大量的创建复杂对象，要比用其他任何方法来copy对象快得多。 注意到，用一个对象为原型，来创建大量的新对象，这正是prototype pattern的本质。

```javascript
  var p1 = new Point(1,2);
  var points = [];
  var PointPrototype = function(){};
  PointPrototype.prototype = p1;
  for(var i = 0; i < 10000; i++)
  {
  points[i] = new PointPrototype();
  //由于PointPrototype的构造函数是空函数，因此它的构造要比直接构造//p1副本快得多。
  }
```

# 3、propertype实质

prototype的行为类似于C++中的静态域，将一个属性添加为prototype的属性，这个属性将被该类型创建的所有实例所共享，但是这种共享是只读的。在任何一个实例中只能够用自己的同名属性覆盖这个属性，而不能够改变它。换句话说，对象在读取某个属性时，总是先检查自身域的属性表，如果有这个属性，则会返回这个属性，否则就去读取prototype域，返回protoype域上的属性。另外，JavaScript允许protoype域引用任何类型的对象，因此，如果对protoype域的读取依然没有找到这个属性，则JavaScript将递归地查找prototype域所指向对象的prototype域，直到这个对象的prototype域为它本身或者出现循环为止。
