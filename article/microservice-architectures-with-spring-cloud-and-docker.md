# 使用 Spring Cloud 和 Docker 实现微服务架构 #

## 简介 ##

本文通过以 Spring Boot, Spring Cloud 和 Docker 构建一个简易的应用程序为例, 为理解常见的微服务架构模式提供一个起点.

全部代码都可以在 [Github](https://github.com/sqshq/PiggyMetrics) 上找到, 所有的镜像也都可以在 Docker Hub 上下载. 只需一个命令即可启动整个系统.

我选择了一个旧的, 曾经是一个庞然大物的项目作为这个系统的基础. 该应用程序的主要功能是处理个人财务, 组织收入和支出, 管理储蓄, 分析统计, 并进行简单的预测.

## 功能服务 ##

这个庞大的旧系统被拆分为三个核心的微服务, 所有的微服务都是围绕某些业务功能进行组织, 可以独立部署的应用程序.

![功能服务](https://dzone.com/storage/temp/1858094-730f2922-ee20-11e5-8df0-e7b51c668847-2.png)

### 帐号服务 ###

使用 Spring Cloud 和 Docker 实现微服务架构含一般用户输入逻辑和验证: 收入/费用项目, 储蓄和账户设置.

| METHOD | PATH                | DESCRIPTION                                 | USER AUTHENTICATED | AVALABLE FROM UI |
| :--    | :--                 | :--                                         | :--                | :--              |
| GET    | /accounts/{account} | 获取账户信息                                |                    |                  |
| GET    | /accounts/current   | 获取当前账户信息                            | ×                 | ×               |
| GET    | /accounts/demo      | 获取模拟账户信息(测试用收入/收费项目信息等) |                    | ×               |
| PUT    | /accounts/current   | 修改当前账户信息                            | ×                 | ×               |
| POST   | /accounts           | 创建账户                                    |                    | ×               |

### 统计服务 ###

对每个账户在统计维度上执行统计并生成时间序列的值, 每个值包含基本货币和时间点. 这些数据可以用于追踪账户在整个生命周期中的现金流动态.

| METHOD | PATH                  | DESCRIPTION            | USER AUTHENTICATED | AVALABLE FROM UI |
| :--    | :--                   | :--                    | :--                | :--              |
| GET    | /statistics/{account} | 获取账户的统计数据     |                    |                  |
| GET    | /statistics/current   | 获取当前账户的统计数据 | ×                 | ×               |
| GET    | /statistics/demo      | 获取模拟账户的统计数据 |                    | ×               |
| PUT    | /statistics/{account} | 创建或更新账户的统计值 |                    |                  |

### 通知服务 ###

存储用户的联系信息和通知设置(如提醒和备份频率), 并从其他服务收集所需的信息, 然后向订阅的客户发送电子邮件.

| METHOD | PATH                | DESCRIPTION                                 | USER AUTHENTICATED | AVALABLE FROM UI |
| :--    | :--                 | :--                                         | :--                | :--              |
| GET | /notifications/settings/current | 获取当前账户的通知设置 | × | × |
| PUT | /notifications/settings/current | 修改当前账户的通知设置 | × | × |

### 备注 ###

- 每个微服务都有自己的数据库, 服务不能绕过其他服务提供的 API 而直接读取它的数据库
- 在这个项目中, 我使用 MongoDB 作为每个服务的主数据库, 对每个服务选择最适合的数据库类型是有意义的
- 服务与服务之间的通信非常简单: 微服务之间只使用 REST API 进行通信. 在正在的产品中会使用多种交互方式, 例如通过同步的 GET 请求来获取数据, 并通过消息队列来异步的执行创建/更新操作, 以便分离服务和消息. 但是这样会带来[最终一致性](https://martinfowler.com/articles/microservice-trade-offs.html#consistency)的问题.

## 基础设施服务 ##

在微服务架构中有很多常见的模式来帮助我们的核心服务正常运行, Spring Cloud 提供了许多强大的工具来为 Spring Boot应用程序实现这些模式. 接下来我会简要介绍一下这些基础设施服务.

![基础设施服务](https://dzone.com/storage/temp/1858172-365c0d94-eefa-11e5-90ad-9d74804ca412-2.png)

### 配置服务 ###

[Spring Cloud Config](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html) 是分布式系统中可水平伸缩的集中式配置服务, 它即支持本地存储, 也支持通过 SVN 和 Git 管理的配置数据.

在这个项目中我使用本地配置文件的方式, Spring Cloud Config 会从本地类路径中加载配置文件, 配置文件都存放在 resources/shared 目录中. 当通知服务从配置服务中获取它的配置时, 配置服务将 shared/notification-service.yml 和 shared/application.yml 中的配置内容返回, 这些配置将在所有的客户端之间共享.

#### 客户端使用方式 ####

只需要将 spring-cloud-starter-config 依赖添加到 Spring Boot 的应用程序中即可, 自动配置将会完成其余的工作.

不需要在你的应用程序中嵌入任何的属性, 只需要提供具有应用程序名称和配置服务地址的 bootstrap.yml 配置文件即可, 文件内容如下:

```
spring:
  application:
    name: notification-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
```

#### 使用 Spring Cloud Config 来动态修改应用程序配置 ####

例如, 在通知服务中的 [EmailService Bean](https://github.com/sqshq/PiggyMetrics/blob/master/notification-service/src/main/java/com/piggymetrics/notification/service/EmailServiceImpl.java) 使用了 @RefreshScope 注解, 这代表我们可以更改电子邮件文本和主题行, 而不需要重新构建和启动通知服务的应用程序.

首先, 在配置服务器中更改所需的属性, 然后执行对通知服务的刷新请求: curl -H "Authorization: Bearer #token#" -XPOST http://127.0.0.1:8000/notifications/refresh 即可.

我们也可以使用 webhooks 的方式来自动化这个过程.

#### 备注 ####

- 动态刷新也存在一些限制: @RefreshScope 不能对含有 @Configuration 注解的类生效, 也不能对含有 @Scheduled 注解的方法生效
- fail-fast 属性代表如果 Spring Boot 应用无法连接到配置服务的话它将启动失败

### 身份验证服务 ###

将授权职责提取到单独的服务中, 并由该服务为后端资源授予 OAuth2 令牌, 身份验证服务用于用户授权以及在外围进行安全的机器到机器通信.

在这个项目中我使用[密码凭证](https://tools.ietf.org/html/rfc6749#section-4.3)作为用户授权的授权类型, 使用[客户端凭证](https://tools.ietf.org/html/rfc6749#section-4.4)作为微服务授权的授权类型.

Spring Cloud Security 提供了便利的注解和自动配置, 使得授权和验证可以轻松的在服务器和客户端实现. 可以在[文档](http://cloud.spring.io/spring-cloud-security/spring-cloud-security.html)中了解更多的信息, 或者查看[身份验证服务](https://github.com/sqshq/PiggyMetrics/tree/master/auth-service/src/main/java/com/piggymetrics/auth)的代码中的详细配置.

从客户端的角度看, 使用方式和传统的基于会话的授权完全一样. 你可以从请求中检索 Principal 对象, 并使用基于表达式的访问控制和 @PreAuthorize 注解来检查用户角色和其他内容.

在 PiggyMetrics 项目中的每个客户端都有一个权限范围: server 用于后端服务, ui 用于浏览器. 我们可以使用以下方式来防止 Controller 被没有权限的角色访问:

```
@PreAuthorize("#oauth2.hasScope('server')")
@RequestMapping(value = "accounts/{name}", method = RequestMethod.GET)
public List<DataPoint> getStatisticsByAccountName(@PathVariable String name) {
    return statisticsService.findByAccountName(name);
}
```

### API网关 ###

在这个项目中我们有三个核心服务会将 API 公开给客户端, 在真正的产品中这个数字以及整个系统的复杂性会非常快速的增长. 实际上, 当渲染一个复杂的网页时可能需要调用[数百种](http://highscalability.com/amazon-architecture)服务.

理论上, 客户端可以直接向每个微服务发出请求, 但是这种方式存在很显然的挑战和复杂性: 客户端需要知道所有的服务端点地址, 分别对每个服务执行请求并将结果进行合并, 这对 web 来说不是一个友好的方式, 但是对后端来说是有用的.

更好的方式是使用 API网关, 它是系统中的一个入口点, 用于将请求路由到合适的后端服务或调用多个后端服务并[汇总结果](https://medium.com/netflix-techblog/optimizing-the-netflix-api-5c9ac715cf19)来响应请求. 此外 API网关还可用于身份验证, 探查, 压力测试和金丝雀测试, 服务迁移, 静态内容响应处理以及主动流量管理.

Nerflix开源了这样的一个 [API网关服务](https://medium.com/netflix-techblog/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee), 在 Spring Cloud 中我们可以简单的使用 @EnableZuulProxy 注解来启用它. 在这个项目中我使用 Zuul 来存储静态内容并将请求路由到相应的微服务, 通知服务的一个简单的路由配置如下:

```
zuul:
  routes:
    notification-service:
        path: /notifications/**
        serviceId: notification-service
        stripPrefix: false
```

上面的配置表示所有以 /notifications 开始的请求都被路由到通知服务进行处理, 在这个方案中没有对地址进行硬编码. Zuul 使用服务发现机制来定位所有的通知服务实例, 断路器和负载均衡也是使用的这种机制.

### 服务发现 ###

另一个众所周知的架构模式是服务发现, 它允许自动检测服务实例的网络地址, 以应对由于自动缩放, 故障或升级等原因造成的服务地址变更.

服务发现的关键部分是服务注册, 我使用 Netflix Eureka 来实现这个功能. Eureka 是客户端发现模式的很好的例子, 由客户端通过注册服务器来发现可用的服务实例的地址以及进行负载均衡.

借助 Spring Boot, 我们可以简单的通过增加 spring-cloud-starter-eureka-server 依赖项, 并通过 @EnableEurekaServer 注解和简单的配置属性来构建 Eureka 注册服务器.

客户端通过使用 @EnableDiscoveryClient 注解和 bootstrap.yml 配置文件来启用服务发现, 一个简单的配置如下:

```
spring:
  application:
    name: notification-service
```

当应用程序启动时, 它将向 Eureka 服务器注册自己的服务并提供服务元数据, 包括主机和端口, 健康检查 URL 以及主页等. Eureka 服务器接收来自各个服务实例的心跳信息, 如果超过配置的时间间隔而没有接收到心跳就将该实例从服务列表中删除.

此外, Eureka 提供了一个简单的界面, 在该界面我们可以跟踪运行的服务实例和数量, 界面的地址为: http://localhost:8761

![Eureka 面板](https://dzone.com/storage/temp/1870222-e1431aecb5c24f6cb130e5e04b498e9e.png)

### 负载均衡, 断路器和 HTTP客户端 ###

Netflix OSS 还提供了其他的功能强大的工具.

#### Ribbon ####

Ribbon 是一个客户端负载均衡器, 它对 HTTP 和 TCP 客户端有很大的控制能力. 与传统的负载均衡器相比, 在 Ribbon 中每个调用都不用额外的通信, 可以直接与所需的服务进行连接.

Ribbon 是开箱即用的, 它与 Spring Cloud 的服务发现集成在一起, 由 Eureka 客户端提供可用的服务器实例列表, 而 Ribbon 在这个列表中进行负载均衡处理.

#### Hystrix ####

Hystrix 是[断路器模式](https://martinfowler.com/bliki/CircuitBreaker.html)的一个实现, 它通过网络访问的延迟和失败来控制对依赖的访问, 其主要目的是防止在微服务的分布式环境中的雪崩现象. 这能够增强系统的容错能力, 有助于系统从错误中尽快恢复.

除了断路器控制之外, Hystrix 中还可以添加一个回退方法, 以便在访问失败的情况下获取一个默认的返回结果.

而且, Hystrix 为每个访问生成执行结果和延迟的度量数据, 我们可以用它来[监控](https://github.com/sqshq/PiggyMetrics#monitor-dashboard)系统的运行状态.

#### Feign ####

Feign 是一个声明式的 HTTP 客户端, 可以和 Hystrix 及 Ribbon 无缝集成. 实际上, 通过添加一个简单的 spring-cloud-starter-feign 依赖和 @EnableFeignClients 注解, 我们就可以拥有一整套的负载均衡器, 断路器和 HTTP 客户端, 并且这些都有合适的随时可用的默认配置.

以下是账户服务中使用 Feign 的一个例子:

```
@FeignClient(name = "statistics-service")
public interface StatisticsServiceClient {
    @RequestMapping(method = RequestMethod.PUT, value = "/statistics/{accountName}", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    void updateStatistics(@PathVariable("accountName") String accountName, Account account);
}
```

- 我们需要做的只是声明一个接口
- 可以在 Feign 中使用 Spring MVC 方式的 @RequestMapping
- 指定需要访问的服务的名称, 或者明确的指定固定的 URL

## 监视仪表板 ##

在这个项目中, 搭载 Hystrix 的每个微服务都通过 Spring Cloud Bus(使用AMQP代理) 向 Terbine 推送指标数据, 监控项目只是一个整合了 [Terbine](https://github.com/Netflix/Turbine) 和 [Hystrix 仪表板](https://github.com/Netflix/Hystrix/tree/master/hystrix-dashboard)的 Spring Boot 应用.

下图描述了在负载下的系统行为:

![image](https://cloud.githubusercontent.com/assets/6069066/14194375/d9a2dd80-f7be-11e5-8bcc-9a2fce753cfe.png)

| ![image](https://cloud.githubusercontent.com/assets/6069066/14127349/21e90026-f628-11e5-83f1-60108cb33490.gif) | ![image](https://cloud.githubusercontent.com/assets/6069066/14127348/21e6ed40-f628-11e5-9fa4-ed527bf35129.gif) | ![image](https://cloud.githubusercontent.com/assets/6069066/14127346/21b9aaa6-f628-11e5-9bba-aaccab60fd69.gif) | ![image](https://cloud.githubusercontent.com/assets/6069066/14127350/21eafe1c-f628-11e5-8ccd-a6b6873c046a.gif) |
| --- |--- |--- |--- |
| 0ms 延迟 | 500毫秒延迟 | 800毫秒延迟 | 1100毫秒延迟 |
| 系统运行良好, 吞吐量为 22请求/秒, 统计服务中的活动线程较少, 服务响应时间中位置为 50毫秒 | 活动线程数量增加, 有 30-40% 的错误率, 但是断路器依然是关闭的 | 失败的比例大于 50%, 断路器处于半开状态, 在一定的睡眠窗口时间之后将放开一个请求 | 100%的错误率, 断路器处于永久打开状态, 即使超过睡眠时间之后也不会关闭断路器, 因为请求响应太慢 |

## 日志分析 ##

当尝试定位分布式环境中的问题时, 集中式日志是非常有用的. Elasticsearch, Logstash 和 Kibana 组成的技术栈可以让你轻松的搜索和分析日志, 利用率和网络活动数据. 在我其他的[项目](https://github.com/sqshq/ELK-docker)中有可用的 Docker 配置.

## 安全 ##

更高级的安全配置超出了这个简易项目的介绍范围, 如果需要模拟更真实的系统, 可以考虑使用 HTTPS 和 JCE 密钥库来加密微服务密码和配置服务器的属性内容, 参阅[文档](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html#_security)以了解更详细的信息.

## 自动化的基础设施 ##

部署一个微服务和它的相关依赖的系统, 相比部署一个单体应用要复杂得多, 所以拥有能够自动化的基础设施是非常重要的. 如果采用持续交付方式, 我们可以获得以下好处:

- 随时发布的能力
- 任何构建都能够部署
- 一次构建, 按需部署

在这个项目中实现的一个简单的持续交付工作流如下:

![自动化基础设施](https://cloud.githubusercontent.com/assets/6069066/14159789/0dd7a7ce-f6e9-11e5-9fbb-a7fe0f4431e3.png)

在配置中, [Travis CI](https://github.com/sqshq/PiggyMetrics/blob/master/.travis.yml) 为每个成功的 Git push 构建一个标记的镜像, 因此 [Docker Hub](https://hub.docker.com/r/sqshq/) 上的每个微服务总是有最新的镜像, 而旧的镜像使用 Git commit hash 作为标记. 如果需要的话, 可以非常容易的对它们进行部署或回滚.

## 如何运行 ##

可以非常简单的运行起整个系统. 我们将启动 8 个 Spring Boot 应用程序, 4 个 MongoDB 实例和 RabbitMq. 请确保机器上有 4GB 的内存, 通过网关, 服务注册, 配置, 认证服务和账户服务, 我们保持服务的正常运行.

### 准备工作 ###

- 安装 Docker 和 Docker Compose
- 配置以下环境变量: CONFIG\_SERVICE\_PASSWORD, NOTIFICATION\_SERVICE\_PASSWORD, STATISTICS\_SERVICE\_PASSWORD, ACCOUNT\_SERVICE\_PASSWORD, MONGODB\_PASSWORD

### 生产环境模式 ###

在这个模式中, 会从 Docker Hub 上拉取最新的镜像. 只需要拷贝 docker-compose.yml 文件然后使用以下命令运行:

```
docker-compose up -d
```

### 开发环境模式 ###

如果你需要构建自己的镜像, 你需要克隆所有的仓库并使用 Maven 构建项目, 然后运行以下命令:

```
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
```

docker-compose.dev.yml 继承了 docker-compose.yml, 可以在本地构建镜像并暴露所有的容器端口以方便开发.

### 重要的端点 ###

- localhost:80 - 网关
- localhost:8761 - Eureka 面板
- localhost:9000 - Hystrix 面板
- localhost:8989 - Turbine 流
- localhost:15672 - RabbitMq 管理端点

### 备注 ###

所有的 Spring Boot 应用都需要在配置服务运行后才能启动. 我们可以同时启动所有的容器, 因为 Spring Boot 的 fail-fast 属性和 restart: always 的 docker-compose 选项. 这代表所有的从属容器将尝试重新启动, 直到配置服务器启动完成.

此外, 在所有的应用启动之后, 服务发现机制还需要一点时间, 直到所有的 Eureka 服务器和客户端都在本地缓存了相同的元数据, 这大概需要 3 个心跳周期, 每个心跳周期的默认时间是 30 秒.

# 资料 #

- 原文 [Microservice Architectures With Spring Cloud and Docker](https://dzone.com/articles/microservice-architecture-with-spring-cloud-and-do)
