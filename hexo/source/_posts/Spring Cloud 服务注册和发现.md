---
title: Spring Cloud 服务注册和发现
date: 2019-01-07 21:19:15
categories: "Spring Cloud教程"
tags: [微服务,Spring Cloud]
---
<Excerpt in index | 首页摘要> 


# Spring Cloud 服务注册和发现

## 搭建服务注册中心
<!-- more -->
<The rest of contents | 余下全文>

### 导入maven依赖

```properties
<parent>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-parent</artifactId>
   <version>1.5.19.BUILD-SNAPSHOT</version>
   <relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
		<java.version>1.8</java.version>
		<spring-cloud.version>Edgware.BUILD-SNAPSHOT</spring-cloud.version>
	</properties><dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>

<properties>
		<java.version>1.8</java.version>
		<spring-cloud.version>Edgware.BUILD-SNAPSHOT</spring-cloud.version>
	</properties><dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>


<dependencyManagement>
   <dependencies>
      <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-dependencies</artifactId>
         <version>${spring-cloud.version}</version>
         <type>pom</type>
         <scope>import</scope>
      </dependency>
   </dependencies>
</dependencyManagement>
```

可通过**Spring Initializr**选择**Eureka Server**模块生成，要注意下SpringBoot的版本，不同的版本对应不同的Spring Cloud版本，我这里用的是1.5.19。



### 修改配置文件

```properties
server.port=1111
eureka.instance.hostname=localhost
#关闭向注册中心注册
eureka.client.register-with-eureka=false
#关闭从注册中心获取实例
eureka.client.fetch-registry=false
```

由于本身就是服务端，所以不需要向注册中心注册自己本身，服务端的只要职责是维护服务实例，所以也不需要获取服务实例



### 修改启动类

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {

   public static void main(String[] args) {
      SpringApplication.run(EurekaApplication.class, args);
   }

}
```

导入依赖后，只需要在启动上面加上**@EnableEurekaServer**注解就可以了



### 启动

直接启动应用，然后访问http://localhost:1111/就能看到注册中心的界面



![1546138557820](https://note.youdao.com/yws/api/personal/file/4EE2AE653A204BB2B2783BEACB5A9FC1?method=download&shareKey=f31e9676db8abd1d5125205013abf48e)

此时可以看到实例列表是没有实例的，接下来我们注册一个客户端上去



## 搭建Eureka客户端

### 导入maven依赖

```
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```



### 修改配置文件

```
spring.application.name=consumer
server.port=1112
#指定注册中心的地址
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```

这里只需要指定注册中心的地址就可以了



### 修改启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {

   public static void main(String[] args) {
      SpringApplication.run(EurekaClientApplication.class, args);
   }
}
```

这里只需要加上**@EnableDiscoveryClient**注解就可以了



### 启动

启动Eureka客户端后，我们再次查看注册中心就能发现我们的客户端实例注册上去了

![1546140044542](https://note.youdao.com/yws/api/personal/file/B63449D6D39E4469A01BDCE88F0148F7?method=download&shareKey=e74937594dd8cb1ea71962253a45296f)



## Eureka Server增加安全用户认证

在使用Eureka Server的时候，我们直接访问注册中心的页面就能看到所有的实例信息，在生产环境这样肯定是不安全的，不仅如此，我们把Eureka Client注册上去也不需要认证，用户只要知道地址就能达到把自己的服务伪装成同名服务注册上去，这也是很不安全的，Erueka Server对此提供了安全用户认证。

 ### 配置访问用户名和密码

```properties
#添加HTTP basic基本验证
security:
  basic:
    enabled: true
  user:
    name: dailn
    password: dailn!123
```

首先我们需要开启安全校验，然后配置用户名和密码

#### 访问注册中心管理页面

![1546155275439](https://note.youdao.com/yws/api/personal/file/9822EBC60FC94E7185258F79D734E298?method=download&shareKey=01a71f78fc63ba692c76bdd0a0186503)

可以看到，这里就需要校验用户和密码了，只有校验通过才能访问注册中心管理页面

![1546155354052](https://note.youdao.com/yws/api/personal/file/73F1E72382944D98810A2298F2DC37FC?method=download&shareKey=087e4f6a8366a6db77aab45c89b1a9c1)

#### 客户端再次注册

```
com.netflix.discovery.shared.transport.TransportException: Cannot execute request on any known server
```

增加了安全认证后，客户端就注册不上去了，此时我们需要在客户端注册的时候也加上安全认证

```properties
eureka.client.security.basic.user=dailn
eureka.client.security.basic.password=dailn!123
eureka.client.serviceUrl.defaultZone=http://${eureka.client.security.basic.user}:${eureka.client.security.basic.password}@localhost:1111/eureka/

```

重新启动就能注册成功，在注册中心管理界面也能看到实例了

![1546156122709](https://note.youdao.com/yws/api/personal/file/1D969E4A75BC4A0F92CEFFA3A41C810E?method=download&shareKey=62d1b8caf282b53132809708ebd18315)



## 总结



这一篇文章我们简单的学习下服务的注册和发现，只需要导入相关的依赖，修改配置文件和启动类就可以了，最后为了安全起见还给Erureka Server增加了安全认证，下一篇文章在介绍服务端如何实现高可用。