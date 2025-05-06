
# PmHub智能项目管理系统

## 项目介绍
基于SpringCloud + SpringCloud Alibaba + Flowable + TTL + Vue技术栈开发的智能项目管理系统。该系统支持自定义项目、上线项目发布、任务分派与执行跟踪、自定义工作流程、企业微信通知集成等功能。管理者可通过ECharts图表实时监控项目和任务进度，实现项目管理流程化和智能化。

## 项目亮点
- 配置Nacos为配置中心和服务注册中心，持久化配置到MySql数据库中，避免了由服务器宕机或重启造成的配置丢失。
- 使用Redis分布式锁，保证流程更新时的顺序执行，避免了由网络或性能原因影响执行顺序。
- 通过AOP特性结合自定义注解，实现了接口鉴权和内部认证。
- 为了保证缓存一致性，采用Cache Aside模式，更新数据后及时清除缓存。
- 使用Sentinel + Gateway实现网关限流，大幅提升单点服务可用性至99%以上，保证高并发情况下的服务可靠性。

## 项目架构

```
1    com.laigeoffer.pmhub
2    |— pmhub-ui                  // 前端框架 [1024]
3    |— pmhub-gateway             // 网关模块 [6880]
4    |— pmhub-auth                // 认证中心 [6800]
5    |— pmhub-api                 // 接口模块
6    |    |— pmhub-api-system                    // 系统接口
7    |    └— pmhub-api-workflow                  // 流程接口
8    |— pmhub-base                // 通用模块
9    |    |— pmhub-base-core                     // 核心模块组件
10   |    |— pmhub-base-datasource               // 多数据源组件
11   |    |— pmhub-base-seata                    // 分布式事务组件
12   |    |— pmhub-base-security                 // 安全模块组件
13   |    |— pmhub-base-swagger                  // 系统接口组件
14   |    └— pmhub-base-notice                   // 消息组件组件
15   |— pmhub-modules             // 业务模块
16   |    |— pmhub-system                        // 系统模块 [6801]
17   |    |— pmhub-gen                           // 代码生成 [6802]
18   |    |— pmhub-job                           // 定时任务 [6803]
19   |    |— pmhub-project                       // 项目服务 [6806]
20   |    └— pmhub-workflow                      // 流程服务 [6808]
21   |— pmhub-monitor             // 监控中心 [6888]
22   └—pom.xml                                   // 公共依赖
```

## 启动RocketMQ的dockercompose脚本

```yaml
version: '3.8'
services:
  namesrv:
    image: apache/rocketmq:5.3.2
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    networks:
      - rocketmq
    command: sh mqnamesrv
  broker:
    image: apache/rocketmq:5.1.0
    container_name: rmqbroker
    ports:
      - 10909:10909
      - 10911:10911
      - 10912:10912
    environment:
      - NAMESRV_ADDR=rmqnamesrv:9876
    depends_on:
      - namesrv
    networks:
      - rocketmq
    volumes:
      - /etc/rocketMQ/broker/broker.conf:/home/rocketmq/rocketmq-5.1.0/conf/broker.conf
      - /etc/rocketMQ/data:/home/rocketmq/store
    command: sh mqbroker
  proxy:
    image: apache/rocketmq:5.1.0
    container_name: rmqproxy
    networks:
      - rocketmq
    depends_on:
      - broker
      - namesrv
    ports:
      - 8080:8080
      - 8081:8081
    restart: on-failure
    environment:
      - NAMESRV_ADDR=rmqnamesrv:9876
    command: sh mqproxy
  dashboard:
    image: apacherocketmq/rocketmq-dashboard:latest
    container_name: rocketmq-dashboard
    environment:
      - JAVA_OPTS=-Drocketmq.namesrv.addr=rmqnamesrv:9876
    ports:
      - "8082:8080"
    networks:
      - rocketmq
    depends_on:
      - namesrv
      - broker
      - proxy
networks:
  rocketmq:
    driver: bridge
```

启动后可以在对应端口看到dashboard

http://[你的IP地址]:8082/#/

![image.png](https://s2.loli.net/2025/04/30/MuVv4XlJrwmGLqe.png)

# 具体功能点

## Gateway全局过滤器统计接口调用耗时

首先我们需要知道网关是什么

网关就是一个个微服务的门卫，所有的请求都交给网关，网关再交给对应服务器。

所以你可以看见，网关天生就是一个反向代理服务。

而在网关中，有下面这三大件

> Route（路由）：这是网关的基本构建块。它由一个 ID，一个目标 URI，一组断言和一组过滤器定义。如果断言为真，则路由匹配。
>
> Predicate（断言）：输入类型是一个 ServerWebExchange。我们可以使用它来匹配来自 HTTP 请求的任何内容，例如 headers 或参数。
>
> Filter（过滤器）：Gateway中的Filter 分为两种类型，分别是 Gateway Filter 和 Global Filter。过滤器 Filter 将会对请求和响应进行修改处理。

所以应该这样理解：

Filter是用来过滤请求的

Predicate是匹配条件

Route 通过Predicate的匹配条件匹配到对应的目标服务器，然后将过滤器中的请求发过去。

### 路由 Route

网关配置如下：

```yaml
spring:
  redis:
  cloud:
    gateway:
      discovery:
        locator:
          lowerCaseServiceId: true
          enabled: true
      routes:
        # 认证中心
        - id: pmhub-auth
          uri: lb://pmhub-auth
          predicates:
            - Path=/auth/**
          filters:
            # 验证码处理
            - CacheRequestFilter
           # - ValidateCodeFilter
            - StripPrefix=1
        # 代码生成
        - id: pmhub-gen
          uri: lb://pmhub-gen
          predicates:
            - Path=/gen/**
          filters:
            - StripPrefix=0
```

比如认证中心的id，就是在nacos中注册的服务名，这样所有`/auth/**`的请求都会被转发到这个服务上。不过id是可以缺省的，uri才是真正决定是哪个服务的路径，不可缺省

有三种方式配置uri

1. webSocket

   ```yaml
   routes:
       # 认证中心
       - id: pmhub-auth
         uri: ws://localhost:8080/
         predicates:
         - Path=/auth/**
   ```

   

2. Http

   ```yaml
   routes:
       # 认证中心
       - id: pmhub-auth
         uri: http://localhost:8080/
         predicates:
         - Path=/auth/**
   ```

3. Nacos

   ```yaml
   routes:
       # 认证中心
       - id: pmhub-auth
         uri: lb://pmhub-auth
         predicates:
         - Path=/auth/**
   ```

   

### 断言 Predicate

断言实现了一组规则，用来给请求找到对应的路径。

spring cloud gateway 在创建route对象时，会使用RoutePredicateFactory生成Predicate对象，这个对象可以赋值给route

多个predicate可以通过逻辑与结合起来使用

常见的断言有下面几种，请看连接 [Spring Cloud Gateway之Predicate断言详解](https://blog.csdn.net/qq_36551991/article/details/135285220)

### 过滤器 Filter

类似spring MVC的拦截器 Interceptor，其中 pre 和 post 分别会在执行前和执行后被调用，分别用来修改请求和返回信息。

过滤器可以用来做**接口时长统计、限流、黑白名单**等。

过滤器一般分三种，全局、单一内置、自定义

#### 全局过滤器

作用于所有路由，无需单独配置，可以实现很多统一化处理的需求，比如权限认证、Ip限制访问等。AuthFilter 就是一个全局过滤器，实现GlobalFilter, Ordered两个接口即可

#### 单一内置过滤器

可以在配置中增加Filters参数，来筛选请求，比如我只接收请求头中带有“X-Request-pmhub”或者“X-Request-pmhub2”的请求，可以这样做

1. 指定请求头内容

```yaml
routes:
    # 认证中心
    - id: pmhub-auth
    uri: ws://localhost:8080/
    predicates:
        - AddRequestHeader=X-Request-pmhub,pmhubValue1
        - AddRequestHeader=X-Request-pmhub,pmhubValue2
```

2. 指定相应头
```yaml
routes:
    # 认证中心
    - id: pmhub-auth
    uri: ws://localhost:8080/
    predicates:
        - AddResponseHeader=X-Response-pmhub, BlueResponse # 新增请求参数X-Response-pmhub并设值为BlueResponse
```

​	过滤相应信息，可以对下游系统或者web做相应的逻辑处理

3. 指定前缀和路径

```yaml
routes:
    # 认证中心
    - id: pmhub-auth
    uri: ws://localhost:8080/
    predicates:
        - PrefixPath=/pmhub # http://localhost:6880/pmhub/gateway/filter
        - RedirectTo=302, https://laigeoffer.cn/ # 访问http://localhost:6880/pmhub/gateway/filter跳转到https://laigeoffer.cn/
```

​	这里还可以进行重定向

#### 自定义过滤器

统计接口耗时情况，讲讲设计思路？

利用自定义过滤器实现。

先创建一个全局Filter，在里面的filter方法中统计

1.  记录接口开始的访问时间
2.  在请求处理完成后执行异步任务记录日志

```java
//先记录下访问接口的开始时间
exchange.getAttributes().put(BEGIN_VISIT_TIME, System.currentTimeMillis());

// Mono.fromRunnable 是非阻塞的，适合在 then 中处理后续的日志逻辑。
return chain.filter(exchange).then(Mono.fromRunnable(() -> {
    try {
        // 记录接口访问日志
        Long beginVisitTime = exchange.getAttribute(BEGIN_VISIT_TIME);
        if (beginVisitTime != null) {
            URI uri = exchange.getRequest().getURI();
            Map<String, Object> logData = new HashMap<>();
            logData.put("host", uri.getHost());
            logData.put("port", uri.getPort());
            logData.put("path", uri.getPath());
            logData.put("query", uri.getRawQuery());
            logData.put("duration", (System.currentTimeMillis() - beginVisitTime) + "ms");

            log.info("访问接口信息: {}", logData);
            log.info("我是美丽分割线: ###################################################");
        }
```



