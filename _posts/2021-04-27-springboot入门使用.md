---
layout:     post
title:      springboot入门使用
subtitle:   简单介绍springBoot的使用方法
date:       2021-04-27
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
---

# springboot创建一个web项目

- 加入web依赖
 添加springboot web依赖
 
> pom.xml
 
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

- 创建controller

新建一个HelloController，在类上加入@RestController注解，其功能相当于：@Controller和@ResponseBody两个注解功能之合。
在HelloController内创建方法hello，在方法上添加@GetMapping("/hello"), 这个注解是@RequestMapping(method = RequestMethod.GET)的缩写，将HTTP Get映射到方法上。

```
package com.ual.demo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String Hello(){
        return "hello world";
    }

}
```

启动应用，访问默认80端口，即可


# 配置文件

在idea创建springBoot项目时，IDE会在src/main/java/resources目录下创建一个application.properties文件。
这时，可以使用以下配置在application.properties来修改tomcat 默认端口

```
server.port=8081
```

# 自定义属性

- 在application.properties中添加如下自定义配置

```
user.name=ual
user.age=24
```

在类中，如果需要读取配置文件内容，只需要在属性上使用@Value("${属性名}")
例如：

```
@RestController
public class UserController {
    @Value("${user.name}")
    private String name;

    @Value("${user.age}")
    private int age;

    @GetMapping("/getuser")
    public String getUser(){
        return name+":"+age;
    }
}
```

即可获取到配置文件中所定义的对应配置。

- 另外，可以通过自定义属性前缀，创建自定义配置实体类：

例如，定义以下实体类

```
package com.ual.demo.domain;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "user")
@Data
public class User {
    private String name;
    private int age;
}
```

在实体类上加入注解@ConfigurationProperties(prefix = "user"),同时通过lombok提供get,set方法,同时需要在启动类上加入注解@EnableConfigurationProperties(User.class)，表明启动这个配置类。

```
package com.ual.demo;

import com.ual.demo.domain.User;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;

@SpringBootApplication
@EnableConfigurationProperties(User.class)
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

在Controller中利用@Autowired注解注入User类。

```
@RestController
public class UserController {
    @Value("${user.name}")
    private String name;

    @Value("${user.age}")
    private int age;

    @Autowired
    private User user;

    @GetMapping("/getuser")
    public String getUser(){
        return name+":"+age;
    }

    @GetMapping("getuserbean")
    public User getUserBean(){
        return user;
    }
}
```

# 多环境配置

在开发中，可能需要一套程序代码需要在不同的环境中发布，但是数据库配置，端口配置或者其他配置各不相同，如果每次都需要修改对应环境配置，难免造成不必要的麻烦。

通常，我们可以配置多个配置文件，在不同情况下进行替换。
为此，我们新建几个配置文件，文件名以**application-{name}.properties**的格式，其中{name}对应环境标识。
例如：
- application-dev.properties:开发环境
- application-test.properties:测试环境
- application-prod.properties:生产环境

然后可以在主配置文件(application.properties)中配置spring.profiles.active来设置当前要使用的配置文件。

> application.properties配置文件

```
server.port=8081

user.name=ual
user.age=24

spring.profiles.active=test
```

> application-test.properties

```
server.port=8080

user.name=uarealoser
user.age=25
```

# 自定义配置文件

介绍完多环境配置之后，我们也可以使用自定义配置文件，例如，新建一个`test.properties`

> test.properties

```
config.name=test
```

新建一个configBean来读取配置文件,添加@PropertySource(value = "classpath:自定义配置文件")
 
```
package com.ual.demo.domain;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.PropertySource;
import org.springframework.stereotype.Component;

@Data
@Component
@PropertySource(value = "classpath:test.properties")
@ConfigurationProperties(prefix = "config")
public class ConfigBean {
    private String name;
}
```

定义controller，注入bean并创建测试接口

```
package com.ual.demo.controller;

import com.ual.demo.domain.ConfigBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import sun.security.krb5.Config;

@RestController
public class ConfigController {
    @Autowired
    private ConfigBean config;

    @GetMapping("/getconfig")
    public ConfigBean getConfig(){
        return config;
    }
}
```

