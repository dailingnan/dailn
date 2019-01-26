---
title: Spring Cloud hystrix服务短路和服务降级
date: 2019-01-12 21:13:18
categories: "Spring Cloud教程"
tags: [微服务,Spring Cloud]
---

<Excerpt in index | 首页摘要> 

# Spring Cloud Hystrix实现服务短路和服务降级



## 背景

​	在微服务架构中，我们将系统拆分成了一个个的**服务单元**，各单元应用间通过服务注册与订阅的方式互相依赖。由于每个单元都在不同的进程中运行，依赖通过远程调用的方式执行，这样就有可能因为**网络原因**或是依赖**服务自身问题**出现调用故障或延迟，而这些问题会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会出现因等待出现故障的依赖方响应而形成**任务积压**，线程资源无法释放，最终导致自身服务的**瘫痪**，进一步甚至出现故障的蔓延最终导致整个系统的瘫痪。如果这样的架构存在如此严重的隐患，那么相较传统架构就更加的不稳定。为了解决这样的问题，因此产生了**断路器**等一系列的服务保护机制。

<!-- more -->
<The rest of contents | 余下全文>

## 断路器和服务降级

### 断路器（服务熔断）

​	很多朋友一开始可能会把断路器和服务降级的概念搞混（表示一开始我也傻傻分不清），这两个其实不是同一个东西来的，断路器本身是一种开关装置，用于在电路上保护线路过载，当线路中有电器发生短路时，断路器能够及时切断故障电路、防止发生过载、发热甚至起火等严重后果。在微服务架构中，断路器的作用也是类似的，当某个服务单元发生故障（类似用电器发生短路）之后，通过断路器的故障监控，向调用方返回一个错误响应，而不是长时间的等待。这样就不会使得线程因服务调用故障服务被长时间占用不释放，避免了故障在分布式系统中的蔓延。

### 服务降级

​	当某个服务出现故障了，在还没修复之前，请求进来肯定都是出错的，从系统整理负荷考虑，此时我们可以提供一种应急方案，主逻辑出错，此时应急方案（次逻辑）就被当成主逻辑调用，保证系统的正常运行。等服务主逻辑被修复了再切换回主逻辑，切换这个过程，可以人工干预，也可以是通过自动策略实现。降级本身也是一种策略，不仅仅应用在微服务中，在分布式系统架构中，会针对很多场景提供降级方案。这样能更好的保证系统的高可用性。

### Spring Cloud断路器和服务降级

​	在Spring  Cloud中，断路器和服务降级是相辅相成的，通过Hystrix实现，当服务熔断后，可直接调用回调方法实现服务降级。同时Hystrix会从系统度量指标metrics中获取服务的健康状态，根据请求总数（QPS），错误百分比等信息，来调整断路器超时时间，某段时间内（休眠窗）使得原先需要2s（Hystrix默认超时时间）才能够触发熔断的逻辑不再有超时时间限制，直接调用降级方法（次逻辑）。相当于在一段时间内，当次逻辑当成主逻辑执行，不再执行主逻辑。一段时间后断路器将进入半开状态，释放一次请求到原来的主逻辑上，如果此次请求正常返回，那么断路器将继续闭合，主逻辑恢复，如果这次请求依然有问题，断路器继续进入打开状态，休眠时间窗重新计时。



## Hystrix 示例

​	我们首先启动3个工程，服务注册中心、consumer服务、consumer2、这几个工程都是基于前面章节的示例，不记得朋友可以去前面章节查看。

​	其中consumer是用来远程掉用consumer2提供的服务，我们给他加上Hystrix功能

#### 添加Hystrix依赖

```properties
       <dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
			<version>1.4.6.RELEASE</version>
		</dependency>
```

博主这里用的spring boot1.5.19，不加版本号，默认去找的Hystrix`1.4.7RELEASE`版本，pom会报错提示找不到对应版本，跑到maven仓库一看，最高才`1.4.6.RELEASE`，肯定找不到呀，所以这里指定下版本就好了。

#### 启动类添加@EnableCircuitBreaker注解

```java
@EnableCircuitBreaker
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {

	public static void main(String[] args) {
//

		SpringApplication.run(EurekaClientApplication.class, args);
	}


	@Bean
	@LoadBalanced
	RestTemplate restTemplate(){
		return new RestTemplate();
	}

}
```

#### consumer的HelloService

```java
@Service
public class HelloServer {

    @Autowired
    RestTemplate restTemplate;

    
    @HystrixCommand(fallbackMethod = "helloFallback")
    public String  hello(){
        return restTemplate.getForEntity("http://consumer2/hello",String.class).getBody();
    }

    public String helloFallback(){
        return "fallback  error";
    }
   
}

```

HelloService这里主要是用来调用consumer2的hello服务，Hystrix是用于客户端的，所以我们这里开启Hystrix,开启只需要在调用方法上面加上`@HystrixCommand`注解就可以了，这里我们在指定下`fallbackMethod`(用于降级的回调方法)为`helloFallback`。

####  consumer2的HelloController

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

HelloController就是consumer2提供的hello服务，这里为了简单测试，就没加服务层了。

#### 测试

我们启动注册中心、consumer、consumer2

服务列表

![](https://note.youdao.com/yws/api/personal/file/C0DC27D38F294EF987EAC3207FADEB19?method=download&shareKey=ccff041e21e6c08237aae031a9e7aa69)

此时能够看到consumer和consumer2实例都正常启动了

访问http://localhost:1112/hello

![](https://note.youdao.com/yws/api/personal/file/AEC7056D8CCB4DD385C99CA6F98F18D5?method=download&shareKey=9979eb6b8f2320a828a9b4bce88565ac)

此时能够看到服务能够正常调用返回hello，接下来我们把consumer2停掉

#### 不启动consumer2服务测试

这里我们测试下不启动consumer2用来模拟服务出现故障的情况

服务列表

![](https://note.youdao.com/yws/api/personal/file/9733E45ADCC7467AB7860A34052FC561?method=download&shareKey=b33fbdd3f0deb1ad133e5d51f3e4edc8)

访问http://localhost:1112/hello

![](https://note.youdao.com/yws/api/personal/file/3519EE73659A4214BCB89499BE607844?method=download&shareKey=787f4034707de377d1406d02f82524bc)

此时能够看到，consumer2出现故障时，能够触发服务熔断并且调用我们指定的回调方法，达到降级的目的。我们再次启动consumer2，此时又能正常的调用，输出hello了。

#### 给consumer添加休眠，模拟网络阻塞

我们之前有提到，断路器的默认短路时间为2秒，这里我们给consumer2休眠3秒，看看能不能触发服务熔断

改造consumer2的HelloController类

```java
@RestController
public class HelloController {

    private  final Logger logger  =  Logger.getLogger(getClass());

    @Autowired
    private DiscoveryClient client;

    @RequestMapping(value="/hello",method = RequestMethod.GET)
    public String index(){
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        ServiceInstance instance = client.getLocalServiceInstance();
        logger.info("/hello host:"+instance.getHost()+"server_id"+instance.getServiceId()+"端口："+instance.getPort());
        return "hello";
    }

}
```

服务列表

![](https://note.youdao.com/yws/api/personal/file/C5DBC0CFE58D42E4AFBDE9196A955711?method=download&shareKey=6bfedc0d36f732ba7542031e9b63c5a5)

这里两个实例都正常启动了，上面的红色警告是没有关闭Eureka的保护机制提醒的，不影响调用。

访问http://localhost:1112/hello

![](https://note.youdao.com/yws/api/personal/file/3519EE73659A4214BCB89499BE607844?method=download&shareKey=787f4034707de377d1406d02f82524bc)

此时可以看到，即时服务正常启动了，但是超过Hystrix的短路超时时间，也会触发熔断。Hystix的超时时间我们可以通过以下配置指定

```properties
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=60000
```



## 总结

通过这篇文章，我们了解了服务熔断，服务降级。以及怎么在Spring Cloud中通过Hystix实现断路器和服务降级。

其实Hystix除了实现断路器和服务降级，还可以用来缓存请求（不能跨请求缓存），请求合并等。。，以后有机会可以单独介绍。