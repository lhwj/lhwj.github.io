---
layout: post
title: "SpringBoot 响应式编程"
date: 2026-05-23 00:00:00 +0800
categories: [SpringBoot响应式编程]
tags: [SpringBoot, Reactive]
---

# SpringBoot 响应式编程详解

# 一、什么是响应式编程？

## 1\.1 定义

**响应式编程 = 非阻塞、事件驱动、异步流式编程**

传统编程：**主动拉取、同步阻塞、等待结果**

响应式编程：**被动响应、异步非阻塞、事件触发**

## 1\.2 核心解决的问题

传统 SpringMVC（Servlet\+Tomcat）最大痛点：**一个请求占一个线程，IO 阻塞时线程闲置浪费，高并发扛不住**

响应式核心优势：**少量线程、不阻塞 CPU、最大化利用机器资源，支撑十万级高并发**

## 1\.3 核心四要素（响应式规范）

- **发布者 Publisher**：产生数据（Mono/Flux）

- **订阅者 Subscriber**：消费数据（框架自动实现）

- **订阅关系 Subscription**：绑定上下游，可取消订阅

- **背压 Backpressure**：下游处理慢，可通知上游「慢点发」，防止服务雪崩

---

# 二、传统MVC VS 响应式WebFlux
|对比维度|SpringMVC（传统）|SpringWebFlux（响应式）|
|---|---|---|
|底层容器|Tomcat（Servlet 阻塞模型）|Netty（事件驱动非阻塞）|
|线程模型|一请求一线程|少量事件循环线程（CPU核数级别）|
|阻塞特性|IO 查询/调用接口会阻塞线程|全程非阻塞，线程不等待|
|返回值|普通对象、String、List|Mono、Flux|
|并发能力|一般，线程池上限瓶颈|极高，适合高吞吐、大流量|
|编程模型|同步、命令式|异步、流式、函数式|

# 三、响应式核心：Mono、Flux

## 3\.1 本质区别

**重点：Mono/Flux 不是数据，是「未来数据的订阅凭证/数据流管道」**

- **Mono<T>**：0 个 / 1 个元素（对应普通单条接口响应）

- **Flux<T>**：0 个 / N 个元素（对应流式、多条、持续推送响应）

## 3\.2 关键方法详解

### ① Mono 延迟非阻塞

```java
// 含义：封装一个hello数据，延迟1秒再推送，全程不阻塞线程
return Mono.just("hello").delayElement(Duration.ofSeconds(1));

```
执行流程：
1. 方法立刻返回 Mono 凭证，业务线程直接释放，去处理其他请求

2. 底层事件循环计时 1 秒

3. 时间到，触发事件，把 hello 推送给前端，完成 HTTP 响应

✅ 对比 `Thread.sleep()`：**sleep会阻塞线程，delayElement 非阻塞**

### ② Flux 流式分批推送

```java
// 含义：3条数据，每两条之间间隔1秒，逐条流式推送
return Flux.just("第一条","第二条","第三条")
        .delayElements(Duration.ofSeconds(1));
```

核心原理：**delayElements 是「元素间隔延迟」，不是整体延迟**

时间轴：0s推第一条、1s推第二条、2s推第三条，分批返回前端

## 3\.3 Mono、Flux 0个元素场景

核心认知：**Mono/Flux 的 0 个元素 ≠ 返回 null/空字符串**，是接口**正常200成功结束，无任何业务数据产出**，属于合法空响应，不会报错。

### 3\.3\.1 Mono 0个元素场景

**Mono\.empty\(\)** 代表 0 个数据：

- 根据条件查询数据，数据库无匹配结果

- 执行删除、修改、新增操作，无需返回任何业务数据，只需告知操作成功

- 权限校验、参数校验不通过，直接结束响应，不返回数据

示例代码：

```java
// 返回0个元素：正常响应、无数据、不报错
@GetMapping("/user/empty")
public Mono<String> emptyUser() {
    return Mono.empty();
}

```

访问效果：HTTP 200 成功，响应体空白，不同于 `Mono.just("")`（返回空字符串，属于1个空元素）。

### 3\.3\.2 Flux 0个元素场景

**Flux\.empty\(\)** 代表数据流为空，无任何元素产出，对应传统接口**返回空List**场景：

- 条件查询列表，数据库无任何匹配数据

- 流式数据生产无结果，直接结束数据流

```java
// 返回0个元素：空数据流，等同于空集合
@GetMapping("/list/empty")
public Flux<String> emptyList() {
    return Flux.empty();
}

```

### 3\.3\.3 数据流
**响应式数据流：有数据就执行链式，没数据直接断路跳过，绝对不会出现 null 空指针。**

Mono.empty () = 数据流彻底结束，直接跳过后面所有操作符
- 不会进 map
- 不会拿到 null
- 不会执行任何代码
- 绝对不会空指针

## 3\.4 Flux 必须流式分批返回？

**Flux 不绑定流式分批，和SSE完全无关！Flux 只是多元素数据流容器，是否分批推送，完全由SSE协议决定**。

### 3\.4\.1 场景一：Flux 普通一次性返回（90%业务场景）

不添加 SSE 响应头，框架会自动**收集全部元素，一次性返回JSON数组**，效果和 MVC 返回 List 完全一致。

```java
// 普通接口：一次性返回全部数据，无流式、无分批
@GetMapping("/flux/list")
public Flux<String> fluxNormalList() {
    return Flux.just("张三","李四","王五");
}

```

前端接收结果：`[张三,李四,王五]`，标准JSON数组，一次性加载完成。

### 3\.4\.2 场景二：Flux 流式分批返回（实时场景）

只有手动声明 `produces = MediaType.TEXT_EVENT_STREAM_VALUE`（SSE协议），才会开启分批流式推送。

```java
// 加SSE协议头，才会1秒推送一条，实现流式响应
@GetMapping(value = "/flux/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> fluxStreamList() {
    return Flux.just("张三","李四","王五")
            .delayElements(Duration.ofSeconds(1));
}

```

## 3\.5 Mino、Flux总结

- ❌ 错误：Mono/Flux 是直接返回给前端的数据

- ✅ 正确：Mono/Flux 是**数据生产规则**，WebFlux框架异步组装HTTP响应

- ❌ 错误：Flux 必然分批流式返回

- ✅ 正确：Flux 默认一次性返回数组，**只有搭配SSE才是流式推送**

- ❌ 错误：empty空元素是报错/空字符串

- ✅ 正确：empty 是合法空响应，0个元素、正常结束、无异常

---

# 四、Flux 和 SSE 的关系

**Flux 和 SSE 是两个完全不同的东西，各司其职、搭配使用**

## 4\.1 两者定位

- **Flux**：后端响应式数据流（Java 代码层面），负责**生产多条流式数据**

- **SSE（Server\-Sent Events）**：HTTP 流式传输协议（网络层面），负责**把多条数据分段传给前端**

## 4\.2 搭配使用原理

在接口上加 `produces = MediaType.TEXT_EVENT_STREAM_VALUE`

1. 告诉浏览器：当前是 SSE 流式长连接，不是一次性 HTTP 响应

2. Flux 持续产生数据

3. WebFlux 框架通过 SSE 协议，逐条推送给前端

4. 前端通过原生`EventSource` 逐条接收渲染

## 4\.3 通俗比喻

- Flux = 水龙头（持续出水、出数据）

- SSE = 水管（负责传输水流/数据流）

- 前端 EventSource = 水桶（接收数据）

---

# 五、WebFlux 两种开发模式

## 5\.1 注解式开发

和 SpringMVC 写法几乎一致，上手零成本，适合 CRUD 业务

```java
@RestController
@RequestMapping("/reactive")
public class ReactiveController {

    // 单次响应（非阻塞延迟）
    @GetMapping("/msg")
    public Mono<String> getMsg() {
        return Mono.just("响应式单条数据")
                .delayElement(Duration.ofSeconds(1));
    }

    // SSE 流式响应
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> stream() {
        return Flux.just("第一条", "第二条", "第三条")
                .delayElements(Duration.ofSeconds(1));
    }
}

```

## 5\.2 函数式路由开发（网关、中间件首选）

抛弃 @Controller、@GetMapping，纯代码定义路由和逻辑，路由集中管理，适合网关、动态路由、请求拦截场景

```java
@Configuration
public class RouteConfig {

    @Bean
    public RouterFunction<ServerResponse> routerFunction() {
        return RouterFunctions.route()
                .GET("/func/hello", req -> ServerResponse.ok().body(Mono.just("函数式路由响应"), String.class))
                .build();
    }
}

```

## 5\.3 选型规则

- **普通业务 CRUD**：用 注解式（简单、好维护、团队通用）

- **网关、路由转发、流式服务、动态路由**：用 函数式路由（灵活、统一管控）

---

# 六、响应式 HTTP 工具：WebClient（非阻塞）

## 6\.1 和传统工具对比

- **RestTemplate**：MVC 专用，**同步阻塞**，调用接口线程卡死等待

- **WebClient**：WebFlux 专用，**异步非阻塞**，全程不占用线程

## 6\.2 WebClient 代码（非阻塞）

```java
@RestController
public class RemoteController {

    // 全局单例 WebClient
    private final WebClient webClient = WebClient.create();

    @GetMapping("/remote")
    public Mono<String> remoteCall() {
        // 非阻塞调用第三方接口，线程不等待，直接释放去处理其他请求
        return webClient.get()
                .uri("https://httpbin.org/get")
                .retrieve()
                .bodyToMono(String.class);
    }
}

```

## 6\.3 RestTemplate 代码（阻塞，传统MVC专用）

**核心特点**：同步阻塞调用，发起接口请求后，当前业务线程会卡死等待响应，全程占用线程资源，直至拿到结果才会释放，无法处理其他请求。

使用前无需额外引入依赖，Spring Web 包已内置，仅适配 SpringMVC 阻塞项目，**不建议在 WebFlux 响应式项目中使用**，会破坏非阻塞线程模型。

```java
@RestController
public class RestTemplateController {

    // 初始化RestTemplate实例
    private final RestTemplate restTemplate = new RestTemplate();

    // 同步阻塞调用第三方接口
    @GetMapping("/sync/remote")
    public String syncRemoteCall() {
        // 重点：线程在此处阻塞等待接口响应，期间无法处理任何请求
        String result = restTemplate.getForObject("https://httpbin.org/get", String.class);
        return result;
    }
}

```

## 6\.4 两者核心执行差异

**RestTemplate（阻塞）**：请求发起 → 线程卡死等待 → 拿到结果 → 方法结束 → 线程释放

**WebClient（非阻塞）**：请求发起 → 立刻返回Mono凭证、释放线程 → 异步等待响应结果 → 结果就绪后回调返回

---

# 七、响应式编程核心优势与适用场景

## 7\.1 核心优势

1. **高并发高性能**：少量线程支撑海量请求，CPU 利用率拉满

2. **非阻塞不浪费资源**：IO 等待时线程释放，处理其他请求

3. **天然支持流式推送**：轻松实现实时消息、日志推送、大屏刷新

4. **背压机制**：防止下游过载，服务更稳定

## 7\.2 适用场景

- 高并发接口、秒杀、大促流量场景

- 实时推送服务（消息、日志、监控、大屏）

- 微服务网关（SpringCloudGateway 底层基于 WebFlux）

- 长连接、流式数据处理场景

## 7\.3 不适用场景

- 简单内部 CRUD 业务（响应式有学习成本，没必要）

- 团队无响应式经验，维护成本高

---

# 八、响应式编程 进阶核心知识点

## 8\.1 响应式核心线程模型

传统MVC采用“一请求一线程（Thread\-Per\-Request）”模型：每接收一个请求就单独分配一条工作线程，请求触发IO阻塞（查库、调接口、等待延时）时，线程会被死死占用、闲置等待，无法处理其他请求，线程池打满后，新请求会排队、阻塞，严重时服务雪崩宕机。

WebFlux 基于 Netty 底层的 **EventLoop 事件循环模型**，是完全不同于MVC的线程调度模型，也是实现高并发、非阻塞的核心：

**EventLoop 执行所有「就绪的业务回调代码」，不等待IO、不做耗时操作、只负责调度，绝不阻塞执行**。

**完整分工机制（最核心真相）**

- **EventLoop线程**：只干「轻量、快速、非阻塞」的活
  执行：map、filter、数据转换、成功回调、异常兜底、路由分发
  不做：等待数据库、等待接口、sleep、任何阻塞等待

- **操作系统/内核线程池**：专门干「耗时IO阻塞活」
  数据库查询、网络请求、磁盘读写全部交给系统异步线程处理

**完整执行流程**

1\. 用户请求进来 → EventLoop 接收请求，派发任务

2\. 遇到数据库/网络IO：**立刻把IO丢给操作系统异步线程**，EventLoop 马上释放，去接下一个请求

3\. IO在后台慢慢执行，不占用EventLoop

4\. IO完成、数据就绪 → **系统回调 EventLoop**

5\. EventLoop 再次接手，执行**业务链式代码**（map、filter、数据封装、返回响应）

**耗时等待交给系统，真正的业务逻辑仍然是 EventLoop 执行！**

**MVC VS WebFlux 终极对比**

- **MVC**：线程一边执行业务，一边卡死等IO，全程占用线程

- **WebFlux**：线程执行业务 → 遇IO立刻撒手 → IO好了再回来执行业务，全程不闲置

### 8\.1\.1 EventLoop 精准定义

EventLoop 本质是**无限循环的轻量工作线程**，核心工作：监听网络事件、分发任务、执行所有就绪的业务回调代码。全程只处理**快速、非阻塞**逻辑，绝不原地等待IO。

**核心真相**：EventLoop 会执行业务代码（map、filter、数据处理、回调逻辑），**但绝不等待耗时IO**。

耗时IO（查库、网络请求、延时）全部交给操作系统异步线程后台处理，EventLoop 立刻释放去处理新请求，IO完成后系统回调线程，继续执行业务收尾逻辑。

### 8\.1\.2 EventLoop 三大核心规则

- **固定少量线程**：默认线程数 = CPU核心数 \* 2，仅用极少系统资源，就能支撑十万级并发请求

- **线程永不阻塞、永不闲置**：只执行就绪的轻量业务代码、回调逻辑，遇到IO立即撒手释放线程，全程高速循环工作，无空闲资源浪费

- **IO事件异步托管**：所有阻塞型IO操作全部交由系统异步处理，不占用核心业务线程，实现真正的非阻塞高并发

### 8\.1\.3 MVC 与 WebFlux 线程模型终极对比

- **MVC（一请求一线程）**：请求线程全程绑定业务，遇IO阻塞卡死闲置，线程池容量决定并发上限，高并发极易排队、雪崩

- **WebFlux（事件循环模型）**：线程遇IO立刻释放，不等待、不闲置，少量线程无限复用，极致压榨CPU性能

### 8\.1\.4 通俗大白话比喻

MVC：一个工人服务一个客户，客户等待业务耗时操作时，工人全程卡死等待，无法接待新人。

WebFlux EventLoop：少量核心工人，快速处理所有即时业务，耗时等待的活全部交给后台助手，工人干完立刻接下一个新请求，全程不摸鱼、不卡顿。

## 8\.2 背压 Backpressure

**背压 = 下游处理慢，上游自动降速，防止服务被冲垮**

### 传统MVC问题

上游疯狂发请求，下游处理不过来，请求堆积、OOM、雪崩，**无任何自我保护**。

### WebFlux背压机制
响应式背压是原生自动机制 ，订阅者（下游）会告诉发布者（上游）：**我现在只能处理2条，你别发10条**。

上游自动限流、减速、等待，**天然限流、防雪崩**。

这是响应式编程碾压传统同步编程的核心特性之一。

## 8\.3 常用核心操作符

### Mono常用

- `Mono.just(T)`：创建单个元素数据流

- `Mono.empty()`：创建0元素空数据流

- `.delayElement()`：非阻塞延迟

- `.map()`：转换数据（A转B）

- `.flatMap()`：异步转换（嵌套Mono/Flux必须用）

- `.onErrorResume()`：异常兜底降级

### Flux常用

- `Flux.just()`：创建多元素数据流

- `Flux.empty()`：空数据流

- `.delayElements()`：元素间隔延迟（流式推送核心）

- `.filter()`：过滤元素

- `.take(n)`：只取前n条数据

## 8\.4 响应式异常处理

响应式中**绝对不能用try\-catch捕获异步异常**，必须用响应式兜底操作符。
```java
@GetMapping("/error")
public Mono<String> errorTest() {
    return Mono.just("正常数据")
            .map(s -> {
                // 模拟业务异常
                int i = 1 / 0;
                return s;
            })
            // 异常降级兜底
            .onErrorResume(e -> Mono.just("服务异常，已降级返回默认数据"));
}

```

核心：**异步逻辑的异常，只能在数据流链路上捕获，外部try\-catch无效**。

## 8\.5 线程切换规则（不能写阻塞代码）

WebFlux 全程基于少量 EventLoop 线程：

❌ **严禁在WebFlux中使用**：

- `Thread.sleep()`

- `RestTemplate` 阻塞调用

- 耗时同步循环、阻塞IO

后果：**直接卡死EventLoop线程，整个服务并发暴跌**

✅ **必须替换为**：

- 非阻塞延迟：`delayElement`

- 非阻塞调用：`WebClient`

- 响应式数据库：Spring Data ReactiveMongo/ReactiveRedis

## 8\.6 MVC 和 WebFlux 不要混用

如果项目同时引入 `web` \+ `webflux` 依赖：

- SpringBoot**自动降级为Tomcat阻塞模型**

- 所有非阻塞特性全部失效

- Mono/Flux 会变成伪异步，完全丧失高并发能力

规范：**普通业务用MVC，高并发/网关项目纯WebFlux，绝不混用**。

## 8\.7 响应式两种数据流终结形态

1. **普通HTTP响应（默认）**：WebFlux收集完所有数据 → 一次性返回JSON，Flux等价List，Mono等价普通对象

2. **SSE流式响应（加协议头）**：数据来一条推一条，长连接持续推送，适合实时大屏、日志、消息通知

## 8\.8 真实企业使用现状

- **普通业务CRUD**：依然用MVC（简单、好维护、团队上手快）

- **网关层、流量入口、高并发服务、实时服务**：清一色WebFlux
  Spring Cloud Gateway（底层纯WebFlux）

- 实时推送服务、IoT设备服务、日志采集服务

- 秒杀、大促流量承接服务

---

# 十、真实业务层响应式开发

**真正的响应式开发核心在 Service 层**，Controller 只做请求接收和响应返回，所有复杂逻辑、查询、转换、兜底全部在 Service 完成。

核心原则：**响应式代码全程链式贯穿，Mono/Flux 从 Service 到 Controller 透传，绝对不允许阻塞、不允许拆包获取数据**。

## 10\.1 业务场景说明

模拟用户业务：根据ID查询用户、查询用户列表、查询不存在数据（0元素场景）、业务数据转换、异常降级。

## 10\.2 第一步：定义实体类

```java
/**
 * 业务实体类
 */
public class User {
    private Long id;
    private String username;
    private Integer age;

    // 构造器、getter、setter
    public User() {}
    public User(Long id, String username, Integer age) {
        this.id = id;
        this.username = username;
        this.age = age;
    }

    // 省略get/set
    public Long getId() { return id; }
    public String getUsername() { return username; }
    public Integer getAge() { return age; }
}

```

## 10\.3 第二步：模拟响应式数据库层（Dao模拟）

模拟 Reactive 数据库查询，**全程非阻塞**，无任何同步代码，贴合真实响应式数据库（ReactiveMongo/MyBatis\-Plus Reactive）。

```java
import org.springframework.stereotype.Repository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.Arrays;
import java.util.List;

/**
 * 响应式数据层（模拟数据库）
 * 所有返回值必须是 Mono/Flux
 */
@Repository
public class UserDao {

    // 模拟数据库存量数据
    private final List<User> userDb = Arrays.asList(
            new User(1L, "张三", 22),
            new User(2L, "李四", 25),
            new User(3L, "王五", 28)
    );

    /**
     * 根据ID查询单个用户
     * 返回Mono：查到返回用户，查不到返回Mono.empty() 0个元素
     */
    public Mono<User> getUserById(Long id) {
        return Flux.fromIterable(userDb)
                .filter(user -> user.getId().equals(id))
                .next(); // next()：取第一条，转成Mono（0/1个元素）
    }

    /**
     * 查询所有用户列表
     * 返回Flux：多条数据，空数据返回Flux.empty()
     */
    public Flux<User> listUser() {
        return Flux.fromIterable(userDb);
    }
}

```

## 10\.4 第三步：核心 Service 业务层

包含：**空数据处理、业务字段转换、异常兜底、非阻塞延迟、业务过滤**，完整复刻真实企业业务逻辑。

```java
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Service
public class UserService {

    // 注入响应式数据层
    private final UserDao userDao;
    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }

    /**
     * 业务1：根据ID查询用户，做业务处理
     * 1. 查不到数据返回空（0元素）
     * 2. 对用户名做业务加工
     * 3. 异常自动降级
     */
    public Mono<String> getUserInfo(Long id) {
        return userDao.getUserById(id)
                // 业务数据转换：数据库实体 → 业务展示数据
                .map(user -> "用户名：" + user.getUsername() + "，年龄：" + user.getAge())
                // 空数据兜底：查不到数据，返回自定义提示（替代原生empty空白响应）
                .defaultIfEmpty("暂无该用户数据")
                // 全局业务异常降级
                .onErrorResume(e -> Mono.just("查询用户失败，服务已降级"));
    }

    /**
     * 业务2：查询用户列表，流式业务处理
     * 1. 过滤成年用户
     * 2. 非阻塞模拟业务耗时
     * 3. 空列表兜底
     */
    public Flux<String> listAdultUser() {
        return userDao.listUser()
                // 业务过滤：只保留18岁以上用户
                .filter(user -> user.getAge() >= 18)
                // 业务数据加工
                .map(user -> "合法成年用户：" + user.getUsername())
                // 模拟业务耗时（非阻塞延迟，不卡死线程）
                .delayElements(java.time.Duration.ofMillis(500))
                // 空列表兜底
                .switchIfEmpty(Flux.just("暂无成年用户数据"))
                // 异常兜底
                .onErrorContinue((e, obj) -> System.out.println("单条数据处理异常，跳过当前数据"));
    }

    /**
     * 业务3：纯空结果场景（0个元素）
     * 适用于：删除/修改操作，无需返回数据
     */
    public Mono<Void> deleteUser(Long id) {
        // 模拟删除业务，无返回数据，返回0个元素
        return Mono.empty();
    }
}

```

## 10\.5 第四步：轻量化 Controller

Controller **不写任何业务逻辑**，只接收参数、调用Service、返回响应。

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/user")
public class UserController {

    private final UserService userService;
    public UserController(UserService userService) {
        this.userService = userService;
    }

    // 查询单个用户业务
    @GetMapping("/{id}")
    public Mono<String> getUser(@PathVariable Long id) {
        return userService.getUserInfo(id);
    }

    // 查询流式用户列表业务
    @GetMapping("/adult/list")
    public Flux<String> getAdultList() {
        return userService.listAdultUser();
    }

    // 无返回值删除业务（0元素场景）
    @GetMapping("/delete/{id}")
    public Mono<Void> delete(@PathVariable Long id) {
        return userService.deleteUser(id);
    }
}

```

## 10\.7 核心重点：defaultIfEmpty 与 switchIfEmpty 完整区别


### 10\.7\.1 核心区分

- **defaultIfEmpty\(\)**：专门给 **Mono（单条数据）** 兜底，只能传「固定静态默认值」

- **switchIfEmpty\(\)**：**Mono/Flux 通用**，主打 **Flux空集合/空数据流**，可以切换「新的异步数据流」

### 10\.7\.2 详细特性对比

#### defaultIfEmpty\(\)

- 适用场景：单个数据查询为空（根据ID查用户、查详情）

- 传入参数：普通静态值（字符串、对象、数字）

- 不支持：异步操作、二次查库、新数据流

- 本质：空数据流时，替换一个固定结果


```java
// Mono空数据兜底，固定提示文本
.defaultIfEmpty("暂无该用户数据")

```

#### switchIfEmpty\(\)

- 适用场景：集合列表为空、任意数据流为空（主打Flux）

- 传入参数：**Mono/Flux 新数据流**

- 支持：异步二次查询、兜底接口调用、复杂链式逻辑

- 本质：空数据流时，**切换一整条新的响应式链路**

```java
// Flux空列表兜底，返回新的Flux数据流
.switchIfEmpty(Flux.just("暂无成年用户数据"))

```

### 10\.7\.3 关键能力差异

如果空数据时需要 **二次查库、调接口、异步处理**：

- ❌ 不能用 defaultIfEmpty（只能写固定值，不支持异步）

- ✅ 必须用 switchIfEmpty（支持嵌套响应式数据流）

### 10\.7\.4 总结

1. **单个数据空、只需要固定提示 → defaultIfEmpty**

2. **列表集合空、需要异步兜底 → switchIfEmpty**

3. **简单静态兜底用 defaultIfEmpty，复杂动态兜底用 switchIfEmpty**


## 10\.8 传统MVC VS 响应式WebFlux 业务本质区别

**传统MVC业务（同步命令式）**：主动查询数据 → 阻塞等待结果 → 拿到数据同步处理 → 直接返回结果，全程占用工作线程，IO耗时全程浪费资源

**WebFlux响应式业务（异步流式）**：定义数据处理规则（链式链路） → 直接返回数据流凭证 → 框架异步执行、IO后台托管 → 数据就绪后回调执行业务、组装响应，全程非阻塞、线程高复用

1. **全程透传**：Dao→Service→Controller，全程 Mono/Flux 透传，不拆包、不阻塞、不获取结果

2. **业务全链式**：所有过滤、转换、兜底、异常处理，全部在数据流链式完成

3. **禁止同步操作**：Service层绝对不能出现 for循环、sleep、同步IO、RestTemplate

4. **空数据必兜底**：所有 empty 场景，必须配置 defaultIfEmpty / switchIfEmpty 友好返回

5. **异常必降级**：所有业务链路必须加响应式异常兜底，防止服务雪崩


# 十一、全文总结

1. **响应式编程核心**：基于事件驱动、异步非阻塞，依靠少量EventLoop线程实现超高并发，自带背压机制保护服务稳定

2. **Mono核心定位**：适配0/1个元素，用于单条查询、详情接口、增删改无返回值业务，empty为合法200空响应

3. **Flux核心定位**：适配0/N个元素，默认一次性返回JSON集合（等价List），仅搭配SSE协议实现流式分批推送

4. **空兜底方法区别**：defaultIfEmpty适配Mono静态固定兜底，switchIfEmpty适配Flux/复杂异步数据流兜底

5. **Flux与SSE关系**：Flux是后端数据流生产工具，SSE是HTTP流式传输协议，二者各司其职、搭配实现实时推送

6. **HTTP调用工具区别**：RestTemplate为MVC阻塞调用，WebClient为WebFlux非阻塞调用，响应式项目严禁混用RestTemplate

7. **线程模型核心差异**：MVC一请求一线程、IO阻塞闲置；WebFlux少量事件循环线程、遇IO即释放、全程高速复用

8. **WebFlux核心禁忌**：禁止MVC与WebFlux依赖混用、禁止所有阻塞代码、必须全程响应式透传

9. **背压机制作用**：下游主动限速，上游动态限流，杜绝请求堆积、OOM、服务雪崩

10. **开发模式选型**：普通CRUD用注解式开发，网关、动态路由、中间件用函数式路由

11. **异常处理规则**：响应式异步逻辑无法用try\-catch捕获，必须使用onErrorResume等链式操作符兜底降级

12. **企业落地规范**：普通内部业务优先MVC（低成本好维护），高并发、大流量、实时推送、网关场景优先WebFlux

1. **响应式编程核心**：非阻塞、事件驱动、异步回调、背压保护，少量线程扛十万并发

2. **Mono**：0/1个元素，对应单条数据、空查询、无返回操作

3. **Flux**：0/N个元素，默认一次性返回集合，仅搭配SSE实现流式分批推送

4. **empty空元素**：合法200空响应，不等于null、空字符串，无报错

5. **Flux与SSE关系**：Flux生产数据流，SSE提供流式传输管道，二者各司其职

6. **WebClient/RestTemplate**：WebClient非阻塞适配WebFlux，RestTemplate阻塞适配MVC

7. **线程模型差异**：MVC一请求一线程，WebFlux少量事件循环线程永不阻塞

8. **核心禁忌**：WebFlux禁止阻塞代码、禁止混用MVC依赖、必须用响应式组件

9. **背压机制**：下游限速、上游限流，天然防堆积、防雪崩

10. **开发模式**：普通业务用注解式，网关/动态路由用函数式路由

1. **响应式编程核心**：非阻塞、事件驱动、异步流式，用少量线程扛高并发

2. **Mono**：0/1 个结果，对应普通单次 HTTP 响应

3. **Flux**：0/N 个结果，对应流式分批推送响应

4. **Flux\+SSE**：Flux 生产数据流，SSE 负责流式传输，前端 EventSource 接收

5. **WebFlux vs MVC**：MVC 阻塞简单，WebFlux 非阻塞高性能

6. **WebClient**：响应式非阻塞 HTTP 调用工具，替代阻塞的 RestTemplate