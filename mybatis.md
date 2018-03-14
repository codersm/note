---
title: mybatis
date: 2016-11-07 19:46:00
tags: mybatis
---

# mybatis中使用map类型参数，其中key为列名，value为列值
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" " http://mybatis.org/dtd/mybatis-3-mapper.dtd">  

<mapper namespace="us.codecraft.webmagic.dao.CrawDao">  

   <insert id="saveNewNews" parameterType="java.util.Map">  
         insert ignore into tb_news   
         <foreach collection="params.keys" item="key" open="(" close=")" separator="," >  
            ${key}  
         </foreach>  
         values   
         <foreach collection="params.keys"  item="key" open="(" close=")" separator=",">  
            #{params[${key}]}  
         </foreach>  
   </insert>  
</mapper>
```

# mybatis like查询%
```xml
--all 用$不能防sql注入  
select * from user where name like '%${name}%'  

--mysql,oracle （db2的concat函数只支持2个参数）  
select * from user where name like concat('%',#{name},'%')   

--oracle,db2  
select * from user where name like '%'||#{name}||'%'  

--SQL Server  
select * from user where name like '%'+#{name}+'%'  

--据说这种是预编译，有空测下  
select * from user where name like "%"#{name}"%"  
```
