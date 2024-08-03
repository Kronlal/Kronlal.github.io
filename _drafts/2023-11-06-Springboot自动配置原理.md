---
layout: post
title: Springboot自动配置原理
subtitle: Springboot笔记
date: 2023-11-06 19:50:00 +0800
categories: Springboot
author: 月梦
cover: 'https://z1.ax1x.com/2023/11/06/pilURCq.png'
cover_author: '加米谷大数据张老师'
cover_author_link: 'https://blog.csdn.net/shuimuzh123?type=blog'
tags: 
- Java  
- Spring  
- Springboot
---

Springboot只需要导入starter，就可以愉快地写代码了，其余的配置都不需要我们来考虑，显得十分便捷，那么Springboot这种自动配置机制的原理是怎样的呢？  

## Springboot开发流程
以web应用程序开发为例：  
1. 导入`starter-web`，即导入了web开发场景  
2. 编写主程序，并且主程序类被注解`@SpringBootApplication`标识  
3. 编写业务代码，全程无需关心各种业务整合（Springboot代替我们完成了）  

## 导入 starter-web
导入web开发的场景启动器`starter-web`  
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
这个场景启动器对应的文件为`spring-boot-starter-web-3.1.5.pom`，它导入了相关场景的所有依赖。例如`spring-boot-starter-json`,`spring-boot-starter-tomcat`,`springmvc`等。  
值得注意的是，每个场景启动器都引入了一个核心场景启动器，即`spring-boot-starter`。  

### 核心场景启动器 spring-boot-starter
核心场景启动器对应的文件为`spring-boot-starter-3.1.5.pom`，可以看到核心场景启动器也引入了若干依赖，其中比较重要的一个是自动配置包，即`spring-boot-autoconfigure`。  
[![pilc8AK.md.png](https://z1.ax1x.com/2023/11/07/pilc8AK.md.png#pic_center "spring-boot-autoconfigure")](https://imgse.com/i/pilc8AK)  
`spring-boot-autoconfigure`包下包含了`springboot`官方所有场景的配置类，只要这个包下的类可以生效，那么`Springboot`官方写好的整合功能就生效了。  

**但是，问题在于，`Springboot`默认只扫描主程序所在的包及其下面的子包，并不能扫描到`spring-boot-autoconfigure`包下的配置类，Springboot是如何让它们生效的呢？**  

## 主程序
一个简单的主程序示例如下：  
```java
// springboot必需的注解
// 表示这是一个springboot应用
@SpringBootApplication
public class MainApp {
    public static void main(String[] args) {
        SpringApplication.run(MainApp.class,args);
    }
}
```
主程序上带有注解`@SpringBootApplication`,这个注解由三个注解组成，分别是`@SpringBootConfiguration`(标识这是一个配置类),`@EnableAutoConfiguration`和`@ComponentScan`。  

### @EnableAutoConfiguration 注解
`@EnableAutoConfiguration`很关键，它是Springboot开启自动配置的核心注解。  
`@EnableAutoConfiguration`上有一个`@Import({AutoConfigurationImportSelector.class})`，其作用是给容器中导入组件，这里是起到批量导入的功能，允许类`AutoConfigurationImportSelector`指定需要导入哪些组件。  
这里使得`Springboot`启动会默认加载100多个配置类，这些类来自于`spring-boot-autoconfigure`包下`META-INF\spring\org.springframework.boot.autoconfigure.AutoConfiguration.imports`文件指定的所有类，名称均为xxxAutoConfiguration形式。  

**虽然`Springboot`默认只扫描主程序所在的包及其子包，但是却通过注解把自动配置类都导入了进来。**  

**虽然这些类全部都被导入了，但是这些类不一定都生效。**  

### 自动配置类生效
以`Kafka`的自动配置类为例：  
```java
@AutoConfiguration
@ConditionalOnClass({KafkaTemplate.class})
@EnableConfigurationProperties({KafkaProperties.class})
@Import({KafkaAnnotationDrivenConfiguration.class, KafkaStreamsAnnotationDrivenConfiguration.class})
public class KafkaAutoConfiguration {
    private final KafkaProperties properties;

    KafkaAutoConfiguration(KafkaProperties properties) {
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean({KafkaConnectionDetails.class})
    PropertiesKafkaConnectionDetails kafkaConnectionDetails(KafkaProperties properties) {
        return new PropertiesKafkaConnectionDetails(properties);
    }
    ······
```
注意到，这个自动配置类之上，有一个注解`@ConditionalOnClass({KafkaTemplate.class})`，这意味着只有当类路径下存在`KafkaTemplate`这个类，也就是说我导入了这个包，整个配置才会生效。  

**这就是按需生效，不是导入的类都能生效，而是通过条件注解来控制哪些类生效。**  

在自动配置类中，会使用`@Bean`注解给容器中放一堆组件，这样`Springboot`就完成了自动配置。  

## 通过配置文件配置
在写好了一个web程序后，为什么通过配置文件(`.properties`)就可以配置应用程序的信息，例如端口号等？  
这是因为每个自动配置类之上都有一个注解形似：  
`@EnableConfigurationProperties({KafkaProperties.class})`，它用于把配置文件中指定前缀的属性值封装到`xxxProperties`属性类中。  
在自动配置类生效时，会自动加载配置文件中的属性，这样只需要程序重启即可更新配置。  

