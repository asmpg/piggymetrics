
# Piggy Metrics

**个人财务简单解决方案**

这是一个用来[验证概念的应用程序](https://piggymetrics.tk), 通过使用Spring Boot, Spring Cloud 和 Docker来编写一个[微服务架构模式](http://martinfowler.com/microservices/) 的demo.
此demo用户界面干净利落.

![](https://cloud.githubusercontent.com/assets/6069066/13864234/442d6faa-ecb9-11e5-9929-34a9539acde0.png)
![Piggy Metrics](https://cloud.githubusercontent.com/assets/6069066/13830155/572e7552-ebe4-11e5-918f-637a49dff9a2.gif)

## 功能性服务

我们将PiggyMetrics 拆解为三个核心微服务. 所有微服务都是可独立部署的程序,分别围绕指定的业务领域进行服务.

<img width="880" alt="Functional services" src="https://cloud.githubusercontent.com/assets/6069066/13900465/730f2922-ee20-11e5-8df0-e7b51c668847.png">

#### 账户服务
包含一般的用户输入逻辑和验证信息: 收入/支出 项目, 存储和账户设置

方法	| 路由	| 描述	| 用户授权	| UI访问权限
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /accounts/{account}	| 获取指定账户信息	|  | 	
GET	| /accounts/current	| 获取当前账户信息 | × | ×
GET	| /accounts/demo	| 获取demo账户信息(预置用户收入/支出信息) |   | 	×
PUT	| /accounts/current	| 保存当前账户信息	| × | ×
POST | /accounts/	| 注册一个新账户	|   | ×


#### 静态数据服务
执行主要静态参数的计算,生成账户时序图. Datapoint contains values, normalized to base currency and time period. 
这些数据主要用来追踪账户生命周期中的动态现金流

Method	| Path	| Description	| User authenticated	| Available from UI
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /statistics/{account}	| Get specified account statistics	          |  | 	
GET	| /statistics/current	| Get current account statistics	| × | × 
GET	| /statistics/demo	| Get demo account statistics	|   | × 
PUT	| /statistics/{account}	| Create or update time series datapoint for specified account	|   | 


#### 通知服务
保存用户联系信息和通知设置(例如提醒和备份频率)
定时从其他服务收集信息,并向订阅用户发送邮件

Method	| Path	| Description	| User authenticated	| Available from UI
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /notifications/settings/current	| Get current account notification settings	| × | ×	
PUT	| /notifications/settings/current	| Save current account notification settings	| × | ×

#### 备注
- 每个微服务拥有独立的数据库,所以服务之间是无法相互绕过API,直接访问彼此的持久化数据的
- 当前项目中, 我是用了MongoDB作为每一个服务的默认数据库,当然,另选一个通用数据库,也是可以的(可以按服务需求,选择更合适的数据库)
- 服务到服务之间的通信方式十分简单: 服务间仅使用同步REST API进行通信. Common practice in a real-world systems is to use combination of interaction styles. For example, perform synchronous GET request to retrieve data and use asynchronous approach via Message broker for create/update operations in order to decouple services and buffer messages. However, this brings us to the [eventual consistency](http://martinfowler.com/articles/microservice-trade-offs.html#consistency) world.

## 基础服务
现在已经有一堆的分布式系统的通用模块可以用来实现我们的核心服务.
[Spring cloud](http://projects.spring.io/spring-cloud/) 为我们提供了一系列的工具,来强化Spring Boot应用,使其实现分布式系统中的各个职能. 我这里简要地介绍一下.
<img width="880" alt="Infrastructure services" src="https://cloud.githubusercontent.com/assets/6069066/13906840/365c0d94-eefa-11e5-90ad-9d74804ca412.png">
### 配置服务
[Spring Cloud Config](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html) 承担了分布式系统中,集中式配置服务的角色, 其具有横向可扩展的特点. 其使用了可插入式的仓库层,目前支持本地存储/Git/SVN
当前项目中,我们使用`native profile`的配置方式来加载classpath中的配置文件,你可以在[Config service resources](https://github.com/sqshq/PiggyMetrics/tree/master/config/src/main/resources)中resource目录看到`share`目录. 当Notification-service 请求他的配置文件时, 
Config service 就会把`shared/notification-service.yml` 和`shared/application.yml` (此文件在所有服务间共享)返回给通知服务.

##### 客户端配置
客户端在Spring Boot application 中加入`spring-cloud-starter-config` 依赖, 剩下的Spring会自动配置.
现在,你不需要在客户端应用中嵌入其他配置信息,只需要提供一个`bootstrap.yml`文件供配置客户端使用,所有配置服务中心的配置文件会自动下载并使用,`bootstrap.yml` 内容如下: 
```yml
spring:
  application:
    name: notification-service #指定客户端应用名称,配置服务中心将使用该字段判断返回内容
  cloud:
    config:
      uri: http://config:8888 # 配置服务中心地址
      fail-fast: true # 如果无法联系配置服务中心,则当前application立即触发启动失败事件
```

##### 通过 Spring Cloud Config, 你可以动态修改application的配置. 
例如在 [EmailService bean](https://github.com/sqshq/PiggyMetrics/blob/master/notification-service/src/main/java/com/piggymetrics/notification/service/EmailServiceImpl.java) 中使用了注解`@RefreshScope`. 这就意味着,仅需如下两步,你就可以不用Rebuild 和Restart Notification service application,却可以改变发送邮件的内容和邮件标题.

第一步: Config server中修改需要的配置文件信息. 第二步: 向 Notification service发送更新配置文件的请求:
`curl -H "Authorization: Bearer #token#" -XPOST http://127.0.0.1:8000/notifications/refresh`

PS: 你也可以使用 [webhooks to automate this process](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html#_push_notifications_and_spring_cloud_bus)来实现上述功能

##### 备注
- 动态配置 有一些限制. `@RefreshScope` 在 `@Configuration` 注解类 和`@Scheduled` 注解方法中并不适用
- `fail-fast` 属性意味着客户端启动时,如果无法连接配置服务,则自动启动失败.
- 下面有一些重大的 [安全备注](https://github.com/sqshq/PiggyMetrics#security) below

### 授权服务
授权职责被完全提取为一个独立服务,用来为其他后端服务提供 [OAuth2 tokens](https://tools.ietf.org/html/rfc6749) 授权服务. 授权服务不仅为用户提供授权服务,同时也为 机器之间的通信提供安全保障

在当前工程,我使用基于[`Password credentials`](https://tools.ietf.org/html/rfc6749#section-4.3)类型的用户验证(仅在UI中使用)和基于[`Client Credentials`](https://tools.ietf.org/html/rfc6749#section-4.4)类型的微服务验证

Spring Cloud Security 提供了注解和自动配置功能,这使得我们很容易授权验证的服务和客户端实现.详情请查看[Spring Cloud Security 文档](http://cloud.spring.io/spring-cloud-security/spring-cloud-security.html) 和[授权服务端代码](https://github.com/sqshq/PiggyMetrics/tree/master/auth-service/src/main/java/com/piggymetrics/auth)

对于授权的客户端来说,一切操作跟基于session的授权验证一样. You can retrieve `Principal` object from request, check user's roles and other stuff with expression-based access control and `@PreAuthorize` annotation.

PiggyMetrics中的授权客户端分为两类: 微服务(account-service, statistics-service, notification-service) 和UI(浏览器端),同样,这两类客户端也就分属不同的 作用范围: `server`和`ui`.所以,我们就可以使用下面这种方式设置外部对于controller的访问

``` java
@PreAuthorize("#oauth2.hasScope('server')")
@RequestMapping(value = "accounts/{name}", method = RequestMethod.GET)
public List<DataPoint> getStatisticsByAccountName(@PathVariable String name) {
	return statisticsService.findByAccountName(name);
}
```

### API 网关
如你所见, 我们有三个核心微服务,以暴露API的方式,对客户端提供服务.实际项目中,随着系统复杂度增加,微服务的数量也会迅速增加. 甚至有可能一个复杂网页的渲染就需要数以百计的服务参与其中.

理论上,一个客户端可以直接跟每个微服务发起请求.但是,这样也带来了显而易见的挑战和限制,比如: 是否客户端要拥有全部endpoints地址,并为每一小块信息单独执行HTTP请求,然后再将这些信息合并成一个整体. 而且,还有可能一些后端微服务使用了web不友好的协议.

因而,我们更倾向于使用API 网关. ta是一个系统的唯一入口,用来接受请求,并将该请求路由到指定后端服务,或者调用数个后端服务[整合结果](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html)后返回给请求发起方.
 Also, it can be used for authentication, insights, stress and canary testing, service migration, static response handling, active traffic management.

Netflix开源了一款用于API 网关的服务[zuul](http://techblog.netflix.com/2013/06/announcing-zuul-edge-service-in-cloud.html), 得益于Spring Cloud ,我们现在可以简单地通过注解 `@EnableZuulProxy`来启用zuul.
当前工程中,我们使用Zuul来提供前端(网页)到微服务的路由工作. 为了方便,我们也将 前端内容集成到了API 网关中.下面是Zuul中Notification service路由的简单配置:

```yml
zuul:
  routes:
    notification-service:
        path: /notifications/**
        serviceId: notification-service
        stripPrefix: false

```

上面的代码表示,Zuul会将所有以`/notifications` 开头的请求路由到Notification service. 这里我们没有任何hardcode 来写明Notification service的地址,Zuul 使用 [Service discovery](https://github.com/sqshq/PiggyMetrics/blob/master/README.md#service-discovery) 机制来定位 Notification service 实例,以及[Circuit Breaker and Load Balancer](https://github.com/sqshq/PiggyMetrics/blob/master/README.md#http-client-load-balancer-and-circuit-breaker), 详见下文.

### 服务发现

分布式架构模式中,另一个常见的组成部分就是 服务发现. 微服务实例由于一些原因(动态扩展/失败/升级)可能会被动态分配地址,服务发现程序可以自动关联这些实例的网址.

服务发现的关键是注册.Netflix提供了一款服务发现框架[Eureka](https://github.com/Netflix/eureka).Eureka提供了很好的 client-side 发现模式 示例,因为 客户端负责决定可用微服务的位置,并负责微服务集群中服务请求的负载均衡

在Spring Boot中,Eureka server的使用仅需三步: 
1. 注入依赖: `spring-cloud-starter-eureka-server`
2. 添加注解: `@EnableEurekaServer` 
3. 创建配置: 
With Spring Boot, you can easily build Eureka Registry with `spring-cloud-starter-eureka-server` dependency, `@EnableEurekaServer` annotation and simple configuration properties.

Eureka Client的使用分为两步:
1. 添加注解: `@EnableDiscoveryClient`
2. 创建配置: `bootstrap.yml`如下:
``` yml
spring:
  application:
    name: notification-service
```

现在,当你启动微服务时, 作为Eureka Client, 该微服务会向Eureka Server注册,并提供一些元信息(host port 健康监控URL等). Eureka Server会接收同属一个service集群的不同微服务实例的心跳,如若失败,则server会在配置的指定时间后,移除该微服务实例.

Eureka 提供了一个简单的用户界面,方便你追踪运行中的service以及每个service的可用实例列表

### 负载均衡 断路器 HTTP客户端

除了上述所说,Netflix 开源工具集 还提供了其他一些很棒的 工具

#### Ribbon
Ribbon 是一个客户端侧的负载均衡器,为你提供超多对于HTTP 和 TCP客户端行为的控制. 相较于传统负载均衡器,你可以直接访问所需服务,而不用额外跳转.

Ribbon原生与Spring Cloud and Service Discovery集成,开箱即用.[Eureka Server](https://github.com/sqshq/PiggyMetrics#service-discovery)提供的动态可用服务列表可与Ribbon结合使用 


#### Hystrix
Hystrix 是一个[断路器](http://martinfowler.com/bliki/CircuitBreaker.html)实现,用来控制服务链中的网络访问的延迟和故障. 目的是阻止拥有大量微服务的分布式系统中级联失败. 从而实现 fail fast,并尽快恢复(这是拥有自我修复的容错系统的重要组成).

通过断路器控制,Hystrix可以帮助你执行一个失败回调函数,用来避免主线任务的失败

此外，Hystrix会为每个命令生成执行结果和延迟的指标，我们可以使用它来[监控系统行为](https://github.com/sqshq/PiggyMetrics#monitor-dashboard).

#### Feign
Feign是一个可声明式的Http客户端,和 Ribbon and Hystrix无缝集成. 

集成并启用上述三个工具,仅需两步:
1. 添加依赖: `spring-cloud-starter-feign`
2. 添加注解: `@EnableFeignClients`

此时,你就拥有了 一整套拥有默认配置且开箱即用的 负载均衡器, 断路器,http客户端

下面是来自Account Service的一个示例:

``` java
@FeignClient(name = "statistics-service")
public interface StatisticsServiceClient {

	@RequestMapping(method = RequestMethod.PUT, value = "/statistics/{accountName}", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
	void updateStatistics(@PathVariable("accountName") String accountName, Account account);

}
```

- 接口一个
- `@RequestMapping` 可在Spring MVC controller 和Feign methods间共享
- 上面示例中,指定了期望调用该接口的 service id - `statistics-service`, 得益于Eureka的自动发现服务 (显然,你也可以通过指定url访问其他资源)

### 监控 控制台

本项目中,每个继承了Hystrix的微服务,都通过Spring Cloud Bus (with AMQP broker) 向Turbine推送数据,监控模块仅仅是一个由[Turbine](https://github.com/Netflix/Turbine) and [Hystrix Dashboard](https://github.com/Netflix/Hystrix/tree/master/hystrix-dashboard)组成的Spring Boot Application


请查阅 [how to get it up and running](https://github.com/sqshq/PiggyMetrics#how-to-run-all-the-things),去启动项目

Let's see our system behavior under load: Account service calls Statistics service and it responses with a vary imitation delay. Response timeout threshold is set to 1 second.

<img width="880" src="https://cloud.githubusercontent.com/assets/6069066/14194375/d9a2dd80-f7be-11e5-8bcc-9a2fce753cfe.png">

<img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127349/21e90026-f628-11e5-83f1-60108cb33490.gif">	| <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127348/21e6ed40-f628-11e5-9fa4-ed527bf35129.gif"> | <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127346/21b9aaa6-f628-11e5-9bba-aaccab60fd69.gif"> | <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127350/21eafe1c-f628-11e5-8ccd-a6b6873c046a.gif">
--- |--- |--- |--- |
| `0 ms delay` | `500 ms delay` | `800 ms delay` | `1100 ms delay`
| Well behaving system. The throughput is about 22 requests/second. Small number of active threads in Statistics service. The median service time is about 50 ms. | The number of active threads is growing. We can see purple number of thread-pool rejections and therefore about 30-40% of errors, but circuit is still closed. | Half-open state: the ratio of failed commands is more than 50%, the circuit breaker kicks in. After sleep window amount of time, the next request is let through. | 100 percent of the requests fail. The circuit is now permanently open. Retry after sleep time won't close circuit again, because the single request is too slow.

### 日志分析

中心化的日志记录有助于在分布式环境中快速定位问题所在. Elasticsearch, Logstash and Kibana stack 可以帮助你检索并分析日志,利用率和网络活动数据
欢迎使用我的Docker[配置文件](http://github.com/sqshq/ELK-docker).

### 分布式跟踪

问题分析是分布式系统的一个难点,例如: 你很难实现跨微服务的问题追踪. 分布式系统中,请求的跟踪和监视特别困难,尤其是当你没有对微服务实例插入监视的时候. 即使有日志,也很难判断请求与那个活动相关.

[Spring Cloud Sleuth](https://cloud.spring.io/spring-cloud-sleuth/)的出现,一举解决了这个难题. ta创造性地给logging加入了两类id: traceId和spanId. spanId代表了一个单一的工作单元,例如发送一个http请求,而traceId则是包含了一系列的spanId,这些spanId组成了一个树形结构.
举个例子: 在一个分布式大数据存储中,一个trace可以是一个PUT请求,traceId和spanId让我们更容易知道当前请求处理到哪里了,从而使log阅读更简单.

The logs are as follows, notice the `[appname,traceId,spanId,exportable]` entries from the Slf4J MDC:

```text
2018-07-26 23:13:49.381  WARN [gateway,3216d0de1384bb4f,3216d0de1384bb4f,false] 2999 --- [nio-4000-exec-1] o.s.c.n.z.f.r.s.AbstractRibbonCommand    : The Hystrix timeout of 20000ms for the command account-service is set lower than the combination of the Ribbon read and connect timeout, 80000ms.
2018-07-26 23:13:49.562  INFO [account-service,3216d0de1384bb4f,404ff09c5cf91d2e,false] 3079 --- [nio-6000-exec-1] c.p.account.service.AccountServiceImpl   : new account has been created: test
```

- *`appname`*: The name of the application that logged the span from the property `spring.application.name`
- *`traceId`*: This is an ID that is assigned to a single request, job, or action
- *`spanId`*: The ID of a specific operation that took place
- *`exportable`*: Whether the log should be exported to [Zipkin](https://zipkin.io/)

## 安全

高级安全配置超出了概念验证项目的范围,如希望模拟更真实环境,请考虑使用https+JCE keystore 来加密微服务密码和配置服务的属性内容
(see [documentation](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html#_security) for details).

## 基建自动化

部署微服务及其依赖 相较于传统单应用部署 会更加复杂. 所以 自动化部署就尤为重要.通过自动交付,我们可以得到以下好处:

- 随时发布
- 任意构建版本都可以发布
- 一次构建,随处发布

下面是本项目使用的持续交付工作流:

<img width="880" src="https://cloud.githubusercontent.com/assets/6069066/14159789/0dd7a7ce-f6e9-11e5-9fbb-a7fe0f4431e3.png">

In this [configuration](https://github.com/sqshq/PiggyMetrics/blob/master/.travis.yml), Travis CI builds tagged images for each successful git push. So, there are always `latest` image for each microservice on [Docker Hub](https://hub.docker.com/r/sqshq/) and older images, tagged with git commit hash. It's easy to deploy any of them and quickly rollback, if needed.

## How to run all the things?

Keep in mind, that you are going to start 8 Spring Boot applications, 4 MongoDB instances and RabbitMq. Make sure you have `4 Gb` RAM available on your machine. You can always run just vital services though: Gateway, Registry, Config, Auth Service and Account Service.

#### Before you start
- Install Docker and Docker Compose.
- Export environment variables: `CONFIG_SERVICE_PASSWORD`, `NOTIFICATION_SERVICE_PASSWORD`, `STATISTICS_SERVICE_PASSWORD`, `ACCOUNT_SERVICE_PASSWORD`, `MONGODB_PASSWORD` (make sure they were exported: `printenv`)
- Make sure to build the project: `mvn package [-DskipTests]`

#### Production mode
In this mode, all latest images will be pulled from Docker Hub.
Just copy `docker-compose.yml` and hit `docker-compose up`

#### Development mode
If you'd like to build images yourself (with some changes in the code, for example), you have to clone all repository and build artifacts with maven. Then, run `docker-compose -f docker-compose.yml -f docker-compose.dev.yml up`

`docker-compose.dev.yml` inherits `docker-compose.yml` with additional possibility to build images locally and expose all containers ports for convenient development.

#### Important endpoints
- http://localhost:80 - Gateway
- http://localhost:8761 - Eureka Dashboard
- http://localhost:9000/hystrix - Hystrix Dashboard (Turbine stream link: `http://turbine-stream-service:8080/turbine/turbine.stream`)
- http://localhost:15672 - RabbitMq management (default login/password: guest/guest)

#### Notes
All Spring Boot applications require already running [Config Server](https://github.com/sqshq/PiggyMetrics#config-service) for startup. But we can start all containers simultaneously because of `depends_on` docker-compose option.

Also, Service Discovery mechanism needs some time after all applications startup. Any service is not available for discovery by clients until the instance, the Eureka server and the client all have the same metadata in their local cache, so it could take 3 heartbeats. Default heartbeat period is 30 seconds.

## Contributions are welcome!

PiggyMetrics is open source, and would greatly appreciate your help. Feel free to suggest and implement improvements.
