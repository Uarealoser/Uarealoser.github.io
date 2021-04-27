---
layout:     post
title:      springboot使用数据库
subtitle:   简单介绍springBoot使用数据库的方法
date:       2021-04-27
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
---

# springBoot使用Mysql数据库

Mysql使用C和C++开发，提供多种存储引擎，提供多种连接途径，例如：ODBC，JDBC，TCP/IP等，并且支持多线程。
在SpringBoot中使用Mysql非常简单，大体分为2步：
1. 在pom.xml中加入依赖

> pom.xml

```xml
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
```

2. 在配置文件中配置数据库信息

> application.properties

```
# 数据库地址
spring.datasource.url=jdbc:mysql://localhost:3306/tests?characterEncoding=utf8&useSSL=false
# 数据库用户名
spring.datasource.username=root
# 数据库密码
spring.datasource.password=1234
# 数据库驱动
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

# springBoot使用redis数据库

1. pom.xml中添加依赖

> pom.xml

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

2. 在配置文件中加入配置

```
# redis
# Redis数据库索引，默认为0
spring.redis.database=0
# Redis服务器地址
spring.redis.host=localhost
# Redis服务器端口
spring.redis.port=6379
# Redis服务器连接密码(默认为空)
#spring.redis.password=
# 连接超时时间(毫秒)
spring.redis.timeout=10
```

# 使用JDBC操作数据库

JDBC的全名是Java DataBase Connectivity，简单理解，它就是一个可以执行SQL语句的Java API

1. JDBC依赖

> pom.xml

```xml
         <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
```

前提需要引入上面的mysql依赖

2. 配置数据库信息

> application.properties

```
# mysql
# 数据库地址
spring.datasource.url=jdbc:mysql://localhost:3306/tests?characterEncoding=utf8&useSSL=false
# 数据库用户名
spring.datasource.username=root
# 数据库密码
spring.datasource.password=1234
# 数据库驱动
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

## JDBC操作

创建实体类

```
@Data
public class UserDomain implements Serializable {
    private int id;
    private String userName;
    private int userAge;

    public UserDomain(int id,String userName,int userAge){
        this.id = id;
        this.userName = userName;
        this.userAge = userAge;
    }
    public UserDomain(){}
}
```

### execute方法

execute方法用来直接执行SQL操作，是最直接的操作数据库的方法。

```
package com.ual.demo.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserDomainController {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @GetMapping("/createTable")
    public String createTable(){
        String sql ="CREATE TABLE `user2` (\n" +
                    "  `id` int(11) NOT NULL AUTO_INCREMENT,\n" +
                    "  `user_name` varchar(255) NOT NULL,\n" +
                    "  `user_age` int(11) DEFAULT 0,\n" +
                    "  PRIMARY KEY (`id`)\n" +
                    ") ENGINE=InnoDB AUTO_INCREMENT=8 DEFAULT CHARSET=utf8;";
        jdbcTemplate.execute(sql);
        return "success";
    }
}
```

这里的被注入的JdbcTemplate就是Spring boot使用JDBC操作数据库的核心。

### update方法

update方法多用于增删改操作，update方法默认返回一个int值，即影响的行数。

```
  @GetMapping("insertUser")
    public String insertUser(){
        String sql = "INSERT INTO user2 (user_name,user_age) VALUES('dalaoyang',12)";
        int rows = jdbcTemplate.update(sql);
        return "success for row:"+rows;
    }
```

动态参数：

```
    @GetMapping("saveUser")
    public String saveUser(){
        String sql = "INSERT INTO user2 (user_name,user_age) VALUES(?,?)";
        int rows = jdbcTemplate.update(sql, "ual", 24);
        return "success for row:"+rows;
    }
```

### 批量处理方法batchUpdate

batchUpdate可以一次执行多个sql

```
   @GetMapping("batchSave")
    public int[] batchSave(){
        String sql = "INSERT INTO user2 (user_name,user_age) VALUES(?,?)";
        List<Object[]> paramList = new ArrayList<>();
        for (int i = 0;i<10;i++){
            Object[] arr = new Object[2];
            arr[0] = "ual"+i;
            arr[1] = 10+i;
            paramList.add(arr);
        }
        return jdbcTemplate.batchUpdate(sql, paramList);
    }
```

### query

```
    @GetMapping("queryUser")
    public List<UserDomain> getUser(){
        String sql = "SELECT * FROM USER2 WHERE USER_NAME = ?";
        return jdbcTemplate.query(sql, new Object[]{"ual"}, new BeanPropertyRowMapper<>(UserDomain.class));
    }
```

## 使用MyBatis操作
 
MyBatis是一个优秀的持久层框架，它支持定制化的SQL，存储过程以及高级映射。可以使用简单的XML或者注解来配置和映射原生信息，将接口和Java的POJOs映射成数据库中的记录。
**支持手写SQL**

### 配置依赖

> pom.xml

```xml
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
            </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
```

### 配置文件

在配置文件里需要配置数据库信息以及MyBatis配置。
MyBatis主要配置：

- logging.level.com.ual.demo.dao.mapper:日志的打印级别，这里的com.ual.demo.dao.mapper对应项目中Mapper的位置
- mybatis.mapper-location:Mapper的存放位置
- mybatis.check-config-location:Mybatis配置是否开启
- mybatis.config-location:MyBatis配置文件位置，与mybatis.check-config-location配合使用

> application.properties

```
# mysql
# 数据库地址
spring.datasource.url=jdbc:mysql://localhost:3306/tests?characterEncoding=utf8&useSSL=false
# 数据库用户名
spring.datasource.username=root
# 数据库密码
spring.datasource.password=1234
# 数据库驱动
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

# mybatis
# mybatis配置是否存在，一般命名为mybatis-config.xml
mybatis.check-config-location=true
# 配置文件位置
mybatis.config-location=classpath:mybatis/mybatis-cofig.xml
# mapper xml 文件位置
mybatis.mapper-locations=classpath*:mapper/*Mapper.xml
# 日志级别
logging.level.com.ual.demo.dao.UserDomainDao = debug
```

在src/main/resources/mybatis 下创建mybatis-config.xml,这个文件是MyBatis的全局配置文件，包含一下几种类型配置：

- properties(属性)
- setting(全局配置参数)
- typeAliases(类型别名)
- typeHandlers(类型处理器)
- objectFactory(对象工厂)
- plugins(插件)
- environments(环境集合属性对象)
- environment(环境子属性对象)
- transactionManager(事务处理)
- dataSource(数据源)
- mappers(映射器)

> mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD SQL Map Config 3.0//EN"     
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>   
    <typeAliases>       
        <typeAlias alias="Integer" type="java.lang.Integer" />      
        <typeAlias alias="Long" type="java.lang.Long" />     
        <typeAlias alias="HashMap" type="java.util.HashMap" />      
        <typeAlias alias="LinkedHashMap" type="java.util.LinkedHashMap" />       
        <typeAlias alias="ArrayList" type="java.util.ArrayList" />    
        <typeAlias alias="LinkedList" type="java.util.LinkedList" />     
        <typeAlias alias="user2" type="com.ual.demo.domain.UserDomain"/>   
    </typeAliases>
</configuration>
```

创建实体类，其中@Alias注解也可以表明使用类别名。
```
package com.ual.demo.domain;

import lombok.Data;
import org.apache.ibatis.type.Alias;

import java.io.Serializable;

@Data
@Alias("user2")
public class UserDomain implements Serializable {
    private int id;
    private String userName;
    private int userAge;

    public UserDomain(int id,String userName,int userAge){
        this.id = id;
        this.userName = userName;
        this.userAge = userAge;
    }
    public UserDomain(){}
}
```

### 基于XML使用

创建Mapper对应的接口类UserMapper，在类上添加注解@Mapper，表明这是一个Mapper。

- 创建Mapper对应接口类,包含5个增删改查数据库的方法：

```
package com.ual.demo.dao.mapper;

import com.ual.demo.domain.UserDomain;
import org.apache.ibatis.annotations.Mapper;

import java.util.List;

@Mapper
public interface UserDomainMapper {
    UserDomain findUserByUserName(String username);
    void updateUserByUserName(UserDomain user);
    void deleteUserByUserName(String username);
    void saveUser(UserDomain user);
    List<UserDomain> getUserList();
}
```

- 在src/main/resources/mapper下创建UserMapper.xml

对应写好在UserMapper接口类方法

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.ual.demo.dao.mapper.UserDomainMapper">
    <resultMap id="user2" type="com.ual.demo.domain.UserDomain"/>
    <parameterMap id="user2" type="com.ual.demo.domain.UserDomain"></parameterMap>
    <select id="findUserByUserName" parameterType="String" resultMap="user2">
        SELECT * FROM user2
        WHERE user_name=#{1}
    </select>
    <update id="updateUserByUserName" parameterType="user2">
        UPDATE user2 SET user_age = #{user_age} WHERE user_name = #{user_name}
    </update>
    <delete id="deleteUserByUserName" parameterType="String">
        DELETE FROM user2 WHERE user_name = #{1}
    </delete>
    <insert id="saveUser" parameterType="user2">
        INSERT INTO user2(user_name,user_age) VALUES (#{user_name},#{user_age})
    </insert>
    <select id="getUserList" resultMap="user2">
        SELECT * FROM user2
    </select>
</mapper>
```

这里先介绍一下Mapper的主要标签:
1. 用于定义SQL语句的

    - insert：多用于执行插入语句，标签内含有id(唯一标识符)和parameterType(传入的参数类型)
    - delete：多用于执行删除语句，标签内有2个属性id(唯一标识符)和parameterType(传入的参数类型)
    - update：多用于执行修改语句，标签内含有2个属性id和parameterType
    - select：多用于执行查询，与上面三个标签相比，多了个resultType的属性，用来接收返回类型。
    
2. 结果集

    - resultMap：用于建立SQL查询结果字段与实体属性的映射关闭信息
 
3. 动态SQL拼接
    
    - if：用于判断，在test属性内加入条件
    - choose：用于判断，与when和otherwise配合使用
    - foreach：循环语句，其中包含属性collection(集合，内容可以是list，array和map)，item(循环遍历的元素)，index(下标)，open(前缀)，close(后缀)，separator(分隔符)

4. 格式化输出
    
    - where：根据标签内的值是否存在自动拼接where语句
    - set：根据标签内的值是否存在自动拼接set语句
    - trim：多用于灵活取出多余关键字的标签，一般结合where或者set使用

5. 配置关联关系
    
    - collection:用于配置一对一关系
    - association：用于配置一对多关系
    
6. SQL标签
    
    - sql：主要用于提取sql片段，便于复用