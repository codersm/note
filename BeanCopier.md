---
title: BeanCopier
date: 2016-08-17 15:16:21
tags: Java
---

在做业务的时候，我们有时为了隔离变化，会将DAO查询出来的Entity，和对外提供的DTO隔离开来。大概90%的时候，它们的结构都是类似的，但是我们很不喜欢写很多冗长的b.setF1(a.getF1())这样的代码，于是我们需要BeanCopier来帮助我们。

<!-- more -->

### 1、使用Cglib的BeanCopier完成bean对象拷贝

|   | 条件                           | 结果   |
|:--|:-------------------------------|:-------|
| 1 | 属性名相同，并且属性类型相同   | ok     |
| 2 | 属性名相同，并且属性类型不相同 | no     |
| 2 | target的setter不规范           | 抛异常 |

> <font color='red'>注意</font>：即使源类型是原始类型(int, short和char等)，目标类型是其包装类型(Integer, Short和Character等)，或反之：都不会被拷贝。

### 2、性能

BeanCopier拷贝速度快，性能瓶颈出现在创建BeanCopier实例的过程中。所以，把创建过的BeanCopier实例放到缓存中，下次可以直接获取，提升性能。

### 3、代码
```java
static final Map<String, BeanCopier> BEAN_COPIERS = new HashMap<String, BeanCopier>();

  public static void copy(Object from, Object to, Converter converter) {
      String key = genKey(from.getClass(), to.getClass());
      boolean isConvert = false;
      if (converter != null) {
          isConvert = true;
      }
      BeanCopier copier = null;
      synchronized (BeanCopierUtils.class) {
          if (!BEAN_COPIERS.containsKey(key)) {
              copier = BeanCopier.create(from.getClass(), to.getClass(), isConvert);
          } else {
              copier = BEAN_COPIERS.get(key);
          }
      }
      copier.copy(from, to, converter);
  }

  private static String genKey(Class<?> srcClazz, Class<?> destClazz) {
      return srcClazz.getName() + destClazz.getName();
  }
```
### 4、参考资料
 - http://czj4451.iteye.com/blog/2044150  
 - http://m.blog.itpub.net/28912557/viewspace-1449672/
