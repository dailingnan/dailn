---
title: Spring Cloud 负载均衡
date: 2019-01-07 21:19:15
categories: "Spring Cloud教程"
tags: [微服务,Spring Cloud]
---
<Excerpt in index | 首页摘要> 
# Spring Cloud Ribbon实现负载均衡



## 负载均衡

负载均衡在系统架构中是一个非常重要的角色，在前面大型网站架构学习总结中，可以看到，高可用，伸缩性，性能几个架构要素中，负载均衡都有着很重要的地位，是系统压力缓解，系统扩容的重要手段之一。
<!-- more -->
<The rest of contents | 余下全文>
### 服务端负载

一般来说，我们讲的负载均衡都是讲服务端负载均衡（不论硬负载还是软负载），比较常见的通过Nginx反向代理来实现负载均衡，例如下面图中所示

![1546179423387](https://note.youdao.com/yws/api/personal/file/172094B19A6B4745BFE6C4A19631F388?method=download&shareKey=e7d1548b174fd31e4faa03d3a57c6dee)

### 客户端负载均衡

这次我们所用到的Ribbon其实就是一种客户端负载均衡，与服务端负载均衡不同的是，客户端负载均衡不是通过一个统一的均衡器（Nginx）去均衡的，而是每一个客户端都维护着各自的负载均衡实现，例如下图所示

![1546179924437](https://note.youdao.com/yws/api/personal/file/0D621A1B9C644A6B9526EC004E443DAA?method=download&shareKey=c3d0120af54ce25309ee252bed77bd06)

### 优势与不足对比

* 客户端负载均衡：优势主要体现在**稳定性高**，各个客户端的负载均衡互不影响，例如上面客户端负载均衡中客户端2的Ribbon出问题了，肯定不会影响客户端1的调用的。劣势就是相对的，**升级维护成本高**，每次要升级的时候，每个客户端的Ribbon都需要升级。
* 服务端负载均衡：优势主要体现在**统一维护成本低**，例如上面的服务端负载均衡，只需要升级Nginx就可以了。劣势同样是相对的，一旦故障，影响大。上面服务端负载均衡中，只要Nginx出问题了，整个系统就不能正常访问了。



## Spring Cloud整合Ribbon实现负载均衡

### 新建服务名为consumer2的Eureka Client（两个实例）

#### 实例1

```properties
server.port=1115
spring.application.name=consumer2
eureka.instance.hostname=localhost1
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```

#### 实例2

```properties
server.port=1114
spring.application.name=consumer2
eureka.instance.hostname=localhost1
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```

#### 提供的服务接口

```java
@RestController
public class HelloController {

    private  final Logger logger  =  Logger.getLogger(getClass());

    @Autowired
    private DiscoveryClient client;

    @RequestMapping(value="/hello",method = RequestMethod.GET)
    public String index(){
        ServiceInstance instance = client.getLocalServiceInstance();
        logger.info("/hello host:"+instance.getHost()+"server_id"+instance.getServiceId()+"端口："+instance.getPort());
        return "hello";
    }


}
```

consumer2的两个实例已经准备好了，暴露出来的接口，访问会打印出各自对应的端口信息，接下来我们就通过Ribbon来测试负载均衡



### 新建名为consumer3的Ribbon Erueka Client工程

```properties
server.port=1116
spring.application.name=consumer3
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```

### Erueka Client导入Ribbon依赖

```properties
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

### 引入Ribbon

```java
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaRibbonApplication {

   public static void main(String[] args) {
      SpringApplication.run(EurekaRibbonApplication.class, args);
   }


   @Bean
   @LoadBalanced
   RestTemplate restTemplate(){
      return new RestTemplate();
   }
}
```

这里可以看到，就是在之前用到的**RestTemplate**对象加上**@LoadBalanced**注解就达到将Ribbon引入的目的了

### 测试负载均衡接口

```properties
@RestController
public class HelloController {

    private  final Logger logger  =  Logger.getLogger(getClass());

    @Autowired
    private DiscoveryClient client;

    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/hello")
    public String index(){
    //在这里调用consumer2提供的接口
        return restTemplate.getForEntity("http://consumer2/hello",String.class).getBody();
    }


}
```

1. 接下来我们分别启动consumer2的两个实例（对应实际中的两台服务器），以及一个包含Ribbon的comsumer3实例。启动完成后服务列表如下

![1546181883854](https://note.youdao.com/yws/api/personal/file/83021B247CD840CF845578707889C1EF?method=download&shareKey=85a84d74e8495724c34f61f86d9cdd61)

2. 访问http://localhost:1116/hello，我们通过浏览器访问5次

   consumer2 （1115）实例控制台输出信息

```properties
2018-12-30 23:00:17.557  INFO 11468 --- [nio-1115-exec-3] c.d.eureka_client1.HelloController       : /hello host:localhost1server_idconsumer2端口：1115
2018-12-30 23:00:22.898  INFO 11468 --- [nio-1115-exec-5] c.d.eureka_client1.HelloController       : /hello host:localhost1server_idconsumer2端口：1115
2018-12-30 23:00:29.584  INFO 11468 --- [nio-1115-exec-7] c.d.eureka_client1.HelloController       : /hello host:localhost1server_idconsumer2端口：1115

```

​     consumer2 （1114）实例控制台输出信息

```properties
2018-12-30 23:00:25.757  INFO 12652 --- [nio-1114-exec-3] c.d.eureka_client1.HelloController       : /hello host:localhost1server_idconsumer2端口：1114
2018-12-30 23:00:30.086  INFO 12652 --- [nio-1114-exec-4] c.d.eureka_client1.HelloController       : /hello host:localhost1server_idconsumer2端口：1114
```

这里可以看到，consumer2 （1115）实例被访问了3次， consumer2 （1114）被访问了2次，由于我们没有配置访问策略，所以默认用的轮询策略，也就证明Ribbon起到负载均衡的作用了。



## 负载均衡策略

这里对上面说的策略补充一下，Ribbon中主要有以下几种策略

* 随机规则：RandomRule 随机访问一个实例
* 最可用规则：BestAvailableRule 根据性能，响应速度，空闲程度等计算
* 轮询规则(Ribbon默认)：RoundRobinRule  多个实例依次轮询访问
* 重试实现：RetryRule 对内部定义的策略反复尝试



# 总结

Eureka 整合Ribbon后，通过RestTemplate以服务名访问的方式调用就能实现负载均衡，我们不需要关注各应用的ip、端口，这些信息Ribbon都能从Eureka Server的服务列表获取到，在此基础上，实现Ribbon还是挺方便的。