---
title: Spring Cloud 服务端高可用
date: 2019-01-07 21:19:15
categories: "Spring Cloud教程"
tags: [微服务,Spring Cloud]
---
<Excerpt in index | 首页摘要> 

# Spring Cloud 服务端高可用

## 背景

在上一篇文章中，我们学习了基本的服务注册和发现，在微服务架构这样的分布式环境中，，我们要充分考虑发生故障的情况，我们知道Eureka服务端主要是维护客户端实例，所以高可用尤为重要，不可能说一个服务端挂了，导致所有的客户端都不可用，接下来我们就学习下如何让服务端实现高可用。
<!-- more -->
<The rest of contents | 余下全文>


## 高可用注册中心

Eureka Server的设计一开始就考虑了高可用的问题，在Eureka的服务治理中，所有服务实例既是服务消费者，也是服务提供者，注册中心也同样如此。之前我们在搭建Eureka Server的时候，有在配置文件中增加

```properties
#不向注册中心注册自己
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

这里是因为自己本身就是服务提供者，就没必要在这个基础上注册自身了，但是还是可以向**其他注册中心**以服务的方式注册自己的，这就是Eureka Server实现高可用的方式。

> Eureka Server的高可用实际上就是将自己作为服务向其他注册中心注册自己，这样就可以形成一组互相注册的服务注册中心，以实现服务清单的互相同步，达到高可用的效果。



## 实现高可用

### 两个Eureka Server的配置

这里我们在之前的Eureka Server基础上增加一个注册中心，并且相互注册，下面试两个注册中心的配置

#### 第一个Eureka Server配置

```properties
server.port=1111
#设置主机名
eureka.instance.hostname=localhost
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
#向Eureka Server 2注册
eureka.client.serviceUrl.defaultZone=http://localhost1:1113/eureka/
```

#### 第二个Eureka Server配置

```properties
server.port=1113
#设置主机名
eureka.instance.hostname=localhost1
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
#向Eureka Server 1注册
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```



这里补充下，因为Eureka 是使用主机名注册的，所以我们指定主机名，然后在HOST文件中加上对应关系

```properties
127.0.0.1 localhost
127.0.0.1 localhost1
```

也可以通过指定ip的方式注册

```eureka.instance.perferIpAddress=true```

#### 测试高可用

##### 查看各自的注册中心界面

接下来我们分别启动两个服务端，查看其注册中心界面

访问http://localhost:1111/

![1546144811852](https://note.youdao.com/yws/api/personal/file/77CC3862BDC947FA81ABBE109C0449C7?method=download&shareKey=bb9059b8243983535ee5e54f2fa4f4c5)

访问http://localhost:1113/

![1546144831986](https://note.youdao.com/yws/api/personal/file/83704A6AFE9C44F79E1F72A879D3D451?method=download&shareKey=e754bb77b79d15429a24f64a89f29b86)

这里可以看到在**DS Replicas**中，有另外一个注册中心的地址，**DS Replicas**是副本的意思，出现这个就代表成功把自己当做服务注册到另外一个注册中心上面去了。

##### 通过客户端来测试高可用

我们通过各自的注册中心能够发现副本就代表注册成功了，那么接下来我们就可以通过Eureka Client来测试一波，流程如下

1. 开启两个Eureka Server，并且相互注册。
2. 开启一个Eureka Client，并且向两个Eureka  Server分别注册
3. 访问通过**RestTemplate**以服务名的方式能够成功访问客户端自己提供的服务。
4. 停止任意一个Eureka Server，再继续访问，看能否成功访问

######  Eureka Client配置

```properties
spring.application.name=consumer
server.port=1112
#向注册中心注册自己，多个用逗号隔开
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/,http://localhost1:1113/eureka/
```

###### 测试Java类

```java
/**
 * @author dailn
 * @Classname TestController
 * @Desc
 * @create 2018-12-30 11:34
 **/
@RestController
public class TestController {

    Logger logger = LoggerFactory.getLogger(TestController.class);

    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/test")
    public void test(){
      restTemplate.getForEntity("http://CONSUMER/helloWord",String.class).getBody();
    }

    @RequestMapping("/helloWord")
    public String helloWord(){
        logger.info("helloWord");
        return "hello word!";
    }

}
```

接下来我们再次查看两个注册中心界面

![1546145797999](https://note.youdao.com/yws/api/personal/file/B3981CFF56C94FD5850D8F36D8A82970?method=download&shareKey=2546a74307b9056e098ab6ac8f038cf2)

![1546145817649](https://note.youdao.com/yws/api/personal/file/7703AA45F7774EDC9AB33F9A4FF7937B?method=download&shareKey=5523338b982f231c66f9051318ea9f5e)

都能看到Eureka Client名为consumer的实例已经注册上来

然后访问测试接口http://localhost:1112/test，观察控制台打印信息

```java
2018-12-30 12:55:24.680  INFO 11084 --- [nio-1112-exec-7] c.d.eureka_client.TestController         : helloWord
```

接下来我们停掉一个服务在观察控制台，为了准确性，我们多访问几次

```java
2018-12-30 13:00:58.104  INFO 11084 --- [nio-1112-exec-1] c.d.eureka_client.TestController         : helloWord
2018-12-30 13:01:03.231  INFO 11084 --- [nio-1112-exec-7] c.d.eureka_client.TestController         : helloWord
2018-12-30 13:01:04.705  INFO 11084 --- [nio-1112-exec-2] c.d.eureka_client.TestController         : helloWord
```

恩，可以看到还是能正常访问，我们切换，停掉另外一个Eureka Server也是同样结果



## 总结

Eureka Server通过注册中心相互注册、同步实例信息、构成集群来实现高可用的，同理，可以推测出Eureka client实现高可用的方式，无非就是改变端口，同一个服务多次注册到注册中心构成集群。客户端集群这里就不在介绍了，等接下来介绍Ribbon的时候在来实现。