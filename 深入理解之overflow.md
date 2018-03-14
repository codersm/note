---
title: "深入理解之overflow"
date: 2016-7-6 10:48:10
tags: css
---
深入理解之overflow
<!--more-->

#### 1、body/html与滚动条
无论什么浏览器，默认滚动条均来自html,
##### 2、获取页面滚动高度
  - Chrome浏览器：document.body.scrollTop;
  - 其他浏览器：document.documentElement.scrollTop
目前，两者不会同时存在，因此，流行这种写法：
```javascript
  var st = document.body.scrollTop || document.documentElement.scrollTop
```    
> **滚动条会占容器尺寸的高度和宽度**。

##### **3、自定义滚动条**
实际开发常用的几个：
```css
  ::-webkit-scrollbar{ /* 血槽宽度 */
      width: 8px; height: 8px;
  }
  ::-webkit-scrollbar-thumb{ /* 拖动条 */
    background-color: rgba(0,0,0,.3);
    border-radius: 6px;
  }
  ::-webkit-scrollbar-track{ /* 背景糟 */
    background-color: #ddd;
    border-radius: 6px;
  }
```
<font color="red">IE浏览器无法自定义滚动条，可以借助jquery scroll插件</font>

##### 4、块级格式化上下文
  - overflow：visible <font color="red">×</font>
  - overflow：auto    <font color="green">√</font>
  - overflow：scroll  <font color="green">√</font>    
  - overflow：hidden  <font color="green">√</font>

  1、清除浮动影响
  2、避免margin穿透问题
  3、两栏自适应布局

```css
  .cell{
    display: table-cell;,width: 2000px;
  }
```



### overflow与绝对定位
>绝对定位元素不总是被父级overflow属相剪裁，尤其当overflow在绝对定位元素及其包含块之间的时候。

    包含块：含position:relative/absolute/fixed声明的父级元素，没有则body元素。



### resize拉伸


##### 6、锚链与锚点
