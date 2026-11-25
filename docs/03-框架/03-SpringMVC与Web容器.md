# Spring MVC 与 Web 容器

> **定位**：讲清一次 HTTP 请求在 Spring MVC 里怎么被接收、路由、绑定、渲染与拦截，以及 Tomcat、会话、跨域、异步、接口安全这些 Web 层必考点。
> **重要度**：★★★★★（Web 层是面试与线上问题的高发区：参数绑定、拦截器、跨域、容器调优必问）
> **建议复习时长**：5~7 小时（建议结合抓包与容器调优实验）
> **关联文档**：[Spring 核心 IoC 与 AOP](./01-Spring核心IoC与AOP.md)、[Spring Boot 与自动装配](./02-SpringBoot与自动装配.md)、[Nginx 与网关限流](../04-中间件/05-Nginx与网关限流.md)、[计算机网络](../06-计算机基础/01-计算机网络.md)、[Redis 数据结构与核心原理](../02-数据库/04-Redis数据结构与核心原理.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| DispatcherServlet 九大组件与请求流程 | ★★★★★ | 源码追问 |
| @RequestMapping 匹配规则与 RESTful 设计 | ★★★★★ | 八股原理 / 场景题 |
| 参数绑定与消息转换 | ★★★★★ | 配置实践 / 故障排查 |
| 数据校验与全局异常处理 | ★★★★☆ | 手写代码 / 配置实践 |
| 拦截器 vs 过滤器 vs AOP | ★★★★★ | 八股原理 / 场景题 |
| 跨域 CORS | ★★★★☆ | 故障排查 / 配置实践 |
| 文件上传下载与静态资源映射 | ★★★☆☆ | 配置实践 |
| 会话管理、JWT 与 SSO | ★★★★★ | 八股原理 / 场景题 |
| Servlet 容器（Tomcat 架构与调优） | ★★★★☆ | 配置实践 / 故障排查 |
| 异步请求、SSE 与 WebSocket | ★★★★☆ | 手写代码 / 场景题 |
| 接口安全（幂等、限流、防重放、越权） | ★★★★★ | 场景题 / 故障排查 |

### 先记住 Web 请求的主线

```text
浏览器发请求 → Tomcat 接收 → Filter 检查/包装 → DispatcherServlet 找 Controller
→ Spring 取参数并转换 → Controller 调业务 → Spring 把结果转换成响应 → 浏览器收到响应
```

后面的章节是在这条主线上补充不同问题：路由如何匹配、参数如何读取、错误如何返回、登录状态如何携带、请求太多或连接太慢时怎么办。

## 1. DispatcherServlet 与请求处理流程
### 1.1 核心原理
#### 九大组件：把名字翻译成“它在请求中做什么”

Spring MVC 的九大组件是按职责分工的清单，不是每个请求都会把九个组件逐个用一遍。日常 REST API 最常碰到的是 HandlerMapping、HandlerAdapter、异常解析器和消息转换器；视图、主题、Flash 属性更多见于传统页面应用。

| 组件 | 用人话说 | 什么时候会用到 |
| --- | --- | --- |
| `HandlerMapping` | 查“这个 URL 和 HTTP 方法应该交给哪个 Controller 方法？” | 几乎每个 MVC 请求 |
| `HandlerAdapter` | Controller 方法有不同类型和调用方式，它负责用合适方式调用找到的处理器 | 几乎每个 MVC 请求 |
| `HandlerExceptionResolver` | Controller 或参数处理出错时，尝试把异常变成错误响应/错误页面 | 请求过程中发生异常 |
| `ViewResolver` | 把 `"orders"` 这样的视图名找到对应模板页面 | 返回模板视图的传统 MVC |
| `RequestToViewNameTranslator` | Controller 没明确给视图名时，按规则猜一个默认视图名 | 少见的隐式视图返回 |
| `LocaleResolver` | 判断当前用户使用什么语言/地区 | 国际化页面或本地化内容 |
| `ThemeResolver` | 选择页面主题/外观 | 使用 Spring MVC 主题功能的传统页面 |
| `MultipartResolver` | 把上传请求解析成文件和表单字段 | `multipart/form-data` 上传 |
| `FlashMapManager` | 临时保存一次重定向要带给下一次请求的数据 | 页面提交后重定向（PRG 模式） |

**一次请求的完整流程**：① 请求进入 DispatcherServlet#doDispatch；② HandlerMapping 找到 `HandlerExecutionChain`（Handler + 拦截器链）；③ 拦截器依次执行 `preHandle`，返回 false 就中断；④ HandlerAdapter 调用 Handler，在调用 Controller 前由 `HandlerMethodArgumentResolver` 解析参数（@RequestParam/@PathVariable/@RequestBody 等各有实现）；⑤ 执行 Controller 方法；⑥ 返回值由 `HandlerMethodReturnValueHandler` 处理（@ResponseBody 走 `HttpMessageConverter`，返回视图名走 ViewResolver）；⑦ 正常处理时执行 `postHandle`；⑧ 渲染视图或写回响应体；⑨ `afterCompletion`。MVC 处理过程中的异常由 `HandlerExceptionResolver` 尝试处理，`@ExceptionHandler`/`@ControllerAdvice` 就是通过 `ExceptionHandlerExceptionResolver` 实现的。

先别背组件名，把一次“查看 CRM 客户详情”的请求想成前台接单：

```text
浏览器说：GET /api/customers/42
  1. DispatcherServlet 接过 HTTP 请求（MVC 的总前台）
  2. HandlerMapping 查到 CustomerController.get(42)（把请求派给谁）
  3. HandlerAdapter 按 MVC 规则准备并调用该方法（负责怎么调用）
  4. 参数解析器从 URL 中取出 42，并转换成 Long
  5. Controller 调业务服务，拿到 CustomerVO
  6. 返回值处理器选 JSON 转换器，把 CustomerVO 写成响应 body
  7. Filter/Interceptor 等完成收尾，响应回到浏览器
```

如果地址存在但没有匹配到方法，通常是 404；如果方法执行时抛异常，则进入异常处理流程。模板页面和 JSON 是两种不同的“回包方式”：模板模式先找模板再生成 HTML；REST 模式通常直接把 Java 对象转换成 JSON。

**把流程拆成可观察的阶段**：浏览器先完成 DNS、TCP/TLS 等连接工作，再发送 HTTP 请求；Servlet 容器把字节解析为 `HttpServletRequest`/`HttpServletResponse`，经过 Filter 链后调用 DispatcherServlet。MVC 先用 HandlerMapping 找到匹配的 Controller 方法及拦截器链；找不到时会产生 404（某些配置下静态资源 Handler 仍可能接手）。接下来 HandlerAdapter 在调用方法前逐个解析参数、执行数据绑定和校验。Controller 返回后，返回值处理器决定是写响应体、返回 `ResponseEntity`、渲染模板，还是启动异步处理。最终容器把响应状态、响应头和 body 写回网络。

**成功与失败路径并不完全对称**：`preHandle` 任一返回 `false` 会停止后续拦截器和 Controller；只有之前已成功通过 `preHandle` 的拦截器会收到反向 `afterCompletion`。Controller 正常返回后才调用 `postHandle`；若 Controller 或参数解析抛异常，通常跳过 `postHandle`，交给异常解析器生成错误响应，最后执行已通过的拦截器的 `afterCompletion`。异步请求首次线程退出时会调用 `AsyncHandlerInterceptor#afterConcurrentHandlingStarted`；等异步结果就绪后再次 dispatch，完成响应时再走完成回调。具体细节可能随 Spring 版本与异步入口略有不同。

**返回值不是“统一转 JSON”**：`@ResponseBody`/`@RestController` 选择消息转换器并直接写 body；`ResponseEntity<T>` 还允许方法同时指定 HTTP 状态和响应头；普通 `@Controller` 返回视图名时由 ViewResolver 解析，再由 View 渲染；返回 `void` 也可能代表已自行写出响应或无需内容。选错返回类型可能导致把字符串当视图名解析、响应重复写入或状态码与业务含义不一致。

**观察请求的最小方法**：给请求分配 traceId，在 Filter 入口/出口记录 method、path、status、耗时（不要默认记录密码、完整 Token 或大 body）；在 MVC 拦截器记录选中的 Handler；结合访问日志和线程 dump 分辨“未到应用”“未匹配路由”“业务执行慢”“下游等待”四类问题。线上日志应对 query、header、body 做敏感字段脱敏。
### 1.2 面试问答
**Q1：DispatcherServlet 的工作流程？** A：收请求 → HandlerMapping 找 Handler 与拦截器 → 拦截器先执行 preHandle → HandlerAdapter 调用 Handler（期间解析参数并执行 Controller）→ 返回值处理（@ResponseBody 用 HttpMessageConverter 写 JSON，否则 ViewResolver 渲染）→ 正常处理后执行 postHandle → 完成响应 → afterCompletion。MVC 处理过程中的异常由 HandlerExceptionResolver 尝试处理。
**Q2：为什么需要 HandlerAdapter 这一层？** A：因为处理器类型不唯一（@Controller 方法、HttpRequestHandler、旧的 Controller 接口、函数式端点），DispatcherServlet 无法知道怎么调用它们，HandlerAdapter 把"调用方式"抽象出来，实现开闭原则——新增处理器类型只需新增适配器。
**Q3：参数是怎么绑定到方法上的？** A：由 `HandlerMethodArgumentResolver` 组合链完成：`@RequestParam`→RequestParamMethodArgumentResolver、`@PathVariable`→PathVariableMethodArgumentResolver、`@RequestBody`→RequestResponseBodyMethodProcessor、`@ModelAttribute`→ModelAttributeMethodProcessor、无注解的 POJO 也走 ModelAttribute。返回值侧对称地由 `HandlerMethodReturnValueHandler` 处理。
**Q4：Controller 是单例吗？线程安全吗？** A：是单例（Spring 默认），因此**不能有可变成员变量**。请求相关数据必须放在方法参数或局部变量里；需要请求级状态时用 `RequestContextHolder` 或 request 作用域 Bean。
### 1.3 高频追问与陷阱
- 追问：`doDispatch` 中 `mappedHandler.applyPreHandle` 返回 false 会直接结束并反向执行已通过的 afterCompletion。
- 陷阱：Filter 在 DispatcherServlet 之前、Interceptor 在其之后，所以 Filter 里拿不到 `HandlerMethod`，但能包装 request/response 改 body。
### 1.4 代码/配置示例
```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public UserVO get(@PathVariable Long id, @RequestParam(required = false) String fields) {
        return userService.get(id, fields);   // 无状态，方法参数即请求上下文
    }
}
```
### 1.5 一句话速记
HandlerMapping 找路、HandlerAdapter 调用、ArgumentResolver 绑参、ReturnValueHandler 出参、ExceptionResolver 兜异常，拦截器穿插其中。

## 2. @RequestMapping 匹配规则与 RESTful 设计
### 2.1 核心原理

可以把 `@RequestMapping` 理解为 Controller 在路由表里登记的“接单条件”。它不只看 URL，还可以同时看 HTTP 方法、query 参数、请求头和请求/响应内容类型。只有请求满足这些条件，方法才匹配。

| 维度 | AntPathMatcher | PathPatternParser |
| --- | --- | --- |
| 默认启用 | Spring 5.3 之前 / Boot 2.5 及之前 | **Spring 6 / Boot 3 默认**（Boot 2.6 起改默认） |
| 匹配时机 | 每次请求解析一次 | 启动时预解析为模式树，性能更好 |
| `**` 语义 | 可出现在中间 | 只允许出现在末尾（更严格） |
| 后缀匹配 | 支持 `/user.json` | 默认关闭 |
| 尾斜杠匹配 | `/user/` 与 `/user` 等价 | Spring 6 中 `useTrailingSlashMatch` 废弃且默认 false |
| 配置项 | — | `spring.mvc.pathmatch.matching-strategy` |

**RESTful 设计要点**：用名词复数表示资源（`/orders`、`/orders/{id}`），用 HTTP 方法表达动作（GET 查、POST 增、PUT 全量改、PATCH 局部改、DELETE 删）；查询条件用 query 参数（分页 `?page=1&size=20`）；状态码语义正确（200 成功、201 创建、204 删除成功无返回、400 参数错、401 未认证、403 无权限、404 不存在、409 冲突、422 语义错误、429 限流、500 服务端错误、503 不可用）；统一响应体（`code`/`message`/`data`），并对**业务错误码与 HTTP 状态码做映射约定**。

RESTful 不是什么神秘框架，而是一种设计 HTTP API 的习惯：把订单当作“资源”，URL 说你要操作哪个资源，HTTP 方法说你想做什么。例如 `GET /orders/42` 是读取订单 42，`DELETE /orders/42` 是删除/取消它；分页、筛选这类条件放在 `?page=1`。不用把动作都写成 `/getOrder`、`/deleteOrder`。但取消订单这类有业务规则的动作，也可以清晰地写成 `POST /orders/42/cancel`；重点是团队语义一致，不是为了 RESTful 把复杂业务硬塞进 PUT/DELETE。

**匹配可以理解为“先找候选，再选最具体的”**：一个 HandlerMapping 会结合路径、HTTP method、请求参数、请求头、`consumes`（请求的 Content-Type）和 `produces`（客户端可接受的响应类型）判断候选方法，再按匹配条件的具体程度选出最合适者。`@RequestMapping` 不仅仅按 URL 匹配。路径存在但 method 不对通常是 405；请求体媒体类型不支持通常是 415；服务端能返回的类型与 `Accept` 不兼容通常是 406。缺少必要参数、类型转换失败则在调用 Controller 前就可能变成 400。实际状态映射受异常处理器及框架配置影响，应结合应用约定检查。

例如下面两个方法虽然都处理 `/orders`，但接单条件不同：

```java
@GetMapping("/orders")
List<OrderVO> list(@RequestParam int page) { ... }  // GET + 查询参数，读取列表

@PostMapping(path = "/orders", consumes = "application/json")
OrderVO create(@RequestBody CreateOrderRequest request) { ... } // POST + JSON，创建订单
```

请求 `GET /orders?page=1` 会匹配第一个；请求 `POST /orders` 且 `Content-Type: application/json` 会匹配第二个。若发成 `GET /orders`，必填的 `page` 缺失可能是 400；如果用 `DELETE /orders` 而没有对应路由，通常是 405；请求体格式不符合 `consumes`，通常是 415。这样记状态码比死背数字容易：**400 是请求内容有问题，404 是路由/资源没找到，405 是方法不对，415 是请求格式不支持。**

**HTTP 方法语义与幂等不是实现保证**：GET 应安全（不改变业务状态）；PUT、DELETE 在 HTTP 语义上定义为幂等，但应用仍要正确实现；POST 通常非幂等。PATCH 表示局部修改，但具体 patch 格式（JSON Merge Patch、JSON Patch 或自定义 DTO）要明确约定。不能因为方法名是 PUT 就认为业务天然不会重复扣款；写操作仍要有业务幂等机制。
### 2.2 面试问答
**Q1：Boot 3 的路径匹配变化会踩什么坑？** A：默认 PathPatternParser：`**` 只能放末尾、后缀匹配关闭、**尾斜杠默认不再等价**。老前端请求 `/api/user/` 会 404（Spring 6 中 `useTrailingSlashMatch` 已废弃，需自己加一条映射或规范 URL）。降级方式：`spring.mvc.pathmatch.matching-strategy=ant_path_matcher`（不推荐长期使用）。
**Q2：RESTful 里 PUT 和 PATCH 怎么选？** A：PUT 表示按接口约定提交资源的完整新状态，PATCH 表示只提交需要改动的部分；“未传字段怎么处理”要由接口约定，不能一概说成 null。HTTP 语义上两者都应幂等，但业务实现仍要保证重复提交不会产生额外副作用。实践中也常用 POST + 明确动作端点（`/orders/{id}/cancel`）表达业务动作。
**Q3：`@RequestBody` 和 `@ResponseBody` 与 RESTful 的关系？** A：二者让 Controller 直接读写消息体，配合 `@RestController`（= @Controller + @ResponseBody）就是 RESTful 接口的默认写法；JSON 序列化由 `MappingJackson2HttpMessageConverter` 完成。
**Q4：接口返回 HTTP 200 但业务失败，算不算 RESTful？** A：严格来说不算（应使用语义化状态码 + ProblemDetail 之类的错误体）；但国内多数规范用"统一响应体 + code 字段"承载业务错误，HTTP 状态码只表达传输与框架级错误。面试时讲清两种约定的取舍：语义化状态码对网关/监控友好，统一 code 对前端处理更简单。
### 2.3 高频追问与陷阱
- 陷阱：`@RequestMapping` 类级与方法级路径拼接时会自动补 `/`，但类级路径不以 `/` 开头也能工作（会被补齐），不要依赖这个宽松行为。
- 追问：`@GetMapping` 等是组合注解，本质是 `@RequestMapping(method = GET)`。
### 2.4 代码/配置示例
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    @PostMapping                       // 201 Created
    @ResponseStatus(HttpStatus.CREATED)
    public OrderVO create(@RequestBody @Valid OrderCreateCmd cmd) { return service.create(cmd); }

    @GetMapping("/{id}")               // 200
    public OrderVO get(@PathVariable Long id) { return service.get(id); }

    @DeleteMapping("/{id}")            // 204 No Content
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void cancel(@PathVariable Long id) { service.cancel(id); }
}
```
### 2.5 一句话速记
Boot 3 默认 PathPatternParser（`**` 只末尾、无后缀与尾斜杠宽容）；RESTful 用名词 + HTTP 方法 + 语义化状态码。

## 3. 参数绑定、消息转换与编码
### 3.1 核心原理

先看请求数据放在哪里，再选绑定注解。HTTP 请求可以粗略拆成“地址路径、地址参数、请求头、请求体”几部分；Spring MVC 不会把所有数据都混在一起猜，而是让不同注解从不同位置取值：

| 注解 | 取值来源 | 适用场景 | 备注 |
| --- | --- | --- | --- |
| @RequestParam | query/form 参数 | 查询条件、分页参数 | 可用 defaultValue/required；同名可省略注解 |
| @PathVariable | URI 模板变量 | 资源 ID | RESTful 路径变量 |
| @RequestBody | 请求体（JSON/XML） | 复杂对象提交 | 只能有一个；靠 HttpMessageConverter 转换 |
| @RequestHeader / @CookieValue | 请求头 / Cookie | 取 Token、TraceId | — |
| @ModelAttribute | 参数自动装配成对象 | 表单提交（key=value） | 也会自动加入 Model |
| @SessionAttribute / @RequestAttribute | session/request 域 | 跨请求或内部转发传值 | — |
| 无注解 POJO | 同 @ModelAttribute | 简单表单绑定 | 复杂嵌套需注意绑定规则 |

**HttpMessageConverter**：按 `Content-Type` 与目标类型选择实现，常用有 `MappingJackson2HttpMessageConverter`（JSON）、`StringHttpMessageConverter`、`ByteArrayHttpMessageConverter`、`FormHttpMessageConverter`（表单/文件）、`MappingJackson2XmlHttpMessageConverter`（需额外依赖）。Spring 6 支持 `ResponseEntity`/`ProblemDetail` 输出 RFC 7807 结构。

把它想成不同格式的“翻译员”：请求写着 `Content-Type: application/json`，JSON Converter 把 JSON 文本翻译成 Java DTO；返回一个 Java 对象时，Jackson Converter 再把对象翻译成 JSON。`Content-Type` 说的是“我发来的 body 是什么格式”，`Accept` 说的是“我希望收到什么格式”，两者不要混淆。

例如：

```text
POST /api/orders?dryRun=true
Content-Type: application/json
X-Trace-Id: abc-123

{"productCode":"CRM-01","quantity":2}
```

在这个请求中，`@PathVariable` 对应 URL 路径中的变量（如 `/orders/{id}`）；`@RequestParam` 对应 `?dryRun=true`；`@RequestHeader` 对应 `X-Trace-Id`；`@RequestBody` 对应最下面的 JSON。表单请求则通常由 `@ModelAttribute` 把多个 `key=value` 字段填进一个对象。先确认数据在请求哪一部分，再选注解，就不会把这些注解记混。

**编码与乱码三处源头**：① 请求体编码——`server.servlet.encoding.charset=UTF-8`、`force=true`（Boot 已默认 UTF-8，Spring 6 中 `force` 相关配置的默认值以官方文档为准，遇乱码显式设置最稳妥）；② URL 编码——Tomcat 8 起 `URIEncoding` 默认 UTF-8，老版本/老容器需显式设置；③ 响应与页面编码——`Content-Type: application/json;charset=UTF-8`、HTML 的 `<meta charset>`；此外数据库连接需 `useUnicode=true&characterEncoding=utf8`（MySQL 8 推荐 `utf8mb4`）。

**一次参数绑定的内部步骤**：DispatcherServlet 选择的 HandlerAdapter 会按方法参数顺序询问一组 `HandlerMethodArgumentResolver`：“我能否处理这个参数？”匹配的解析器读取相应数据源，然后通过 `ConversionService`/`WebDataBinder` 做类型转换和绑定。query 中的 `"42"` 因而可转成 `Long`；格式错误会在 Controller 执行前失败。表单或 `@ModelAttribute` 对象绑定后再校验；`@RequestBody` 则先由 Converter 按 Content-Type 反序列化，再对对象校验。JSON 的属性名、构造器、日期格式等由 Jackson 配置决定，和表单 setter 绑定不是同一套机制。

**绑定边界与安全**：不要把数据库实体直接作为输入 DTO，尤其不要允许客户端通过自动绑定修改 `role`、`ownerId`、`price` 等服务端字段（mass assignment）。用专用命令对象只暴露允许输入的属性；复杂绑定可用 `@InitBinder` 禁止字段或注册转换器。`@RequestBody` 通常消费请求流，不能假设同一 body 可被多个参数解析器重复读取。缺少 `Content-Type`、发送了不匹配的媒体类型，或响应的 `Accept` 不匹配，都可能在业务方法执行前导致 415/406。

**响应写回的选择过程**：方法返回后，ReturnValueHandler 识别返回类型，随后选一个 `HttpMessageConverter`，由它把 Java 对象序列化为 JSON、文本或字节流并设置响应头。协商会结合 `produces`、客户端的 `Accept` 和可用 Converter；例如浏览器可接受 JSON，而 Controller 返回对象时通常使用 Jackson Converter。`ResponseEntity<T>` 可以显式设置状态与 headers，再由 Converter 写 body；`ResponseBodyAdvice` 可在写出前统一包装或脱敏。响应一旦提交到网络（例如流式下载或已 flush），就不能再改状态码或重写已发送的 body。

**最小 DTO 示例**（省略 import；需要 Jakarta Validation 与 Bean Validation Provider，例如 Hibernate Validator）：
```java
public record CreateOrderRequest(
        @NotBlank String productCode,
        @Min(1) @Max(100) int quantity) {}

@RestController
@RequestMapping("/api/orders")
class OrderController {
    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    ResponseEntity<OrderView> create(@Valid @RequestBody CreateOrderRequest request) {
        OrderView created = orderService.create(request);
        return ResponseEntity.created(URI.create("/api/orders/" + created.id())).body(created);
    }
}
```
这里 `consumes` 明确请求体格式，`@RequestBody` 负责 JSON 到 DTO 的转换，`@Valid` 在方法调用前校验字段；转换错误与校验错误都应转成不泄露内部细节的 4xx 响应。
### 3.2 面试问答
**Q1：@RequestParam、@PathVariable、@RequestBody 的区别？** A：@RequestParam 取 query/form 参数（String→目标类型转换）；@PathVariable 取 URI 模板变量；@RequestBody 读整个请求体并用 MessageConverter 反序列化（通常 JSON）。要点：GET 请求一般用前两者，POST JSON 用 @RequestBody；@RequestBody 与 @RequestParam 可同时使用（JSON body + query 参数）。
**Q2：@RequestBody 能接收 Map 吗？接收 List 吗？** A：能接 `Map<String,Object>`、`List<T>`（Jackson 只要泛型信息可推断），但**不推荐**：类型不安全、无法校验、接口文档不可读。规范做法是定义 DTO 并用 `@Valid` 校验。
**Q3：前端传 JSON 中文乱码怎么排查？** A：按三层查：① 请求头 `Content-Type` 是否带 `charset=UTF-8`；② 服务端是否设置了 `server.servlet.encoding.force=true`；③ 若用 MessageConverter 自定义了编码，检查 `supportedMediaTypes` 与 `StringHttpMessageConverter` 默认字符集（历史上存在 ISO-8859-1 默认值的坑）。响应乱码则看 `produces` 与 `Content-Type` 是否一致。
**Q4：为什么表单能绑定到对象，而 JSON 不行？** A：绑定机制不同。表单（`application/x-www-form-urlencoded`）由 `ServletRequestDataBinder` 按参数名逐字段 setter 绑定（支持 `user.name` 这种嵌套路径）；JSON 由 Jackson 反序列化（要求 setter/字段/构造器匹配，或用 `@JsonCreator`）。所以表单能绑定的对象用 JSON 传可能失败，反之亦然。
### 3.3 高频追问与陷阱
- 陷阱：`@RequestBody` 只能有一个，且与 `@ModelAttribute` 同时使用会冲突（一个读流、一个读参数）。
- 陷阱：GET 请求带 `@RequestBody` 在部分客户端/网关下 body 会被丢弃，规范上不应这样用。
### 3.4 代码/配置示例
```java
@PostMapping("/search")
public PageResult<OrderVO> search(@RequestBody @Valid OrderQuery query,
                                  @RequestHeader(value = "X-Trace-Id", required = false) String traceId) {
    return service.search(query);
}
```
（乱码相关配置：`server.servlet.encoding.charset=UTF-8`、`server.servlet.encoding.force=true`）
### 3.5 一句话速记
query/form 用 @RequestParam/@ModelAttribute，URI 用 @PathVariable，JSON 用 @RequestBody；乱码先查 Content-Type charset 与 encoding.force。

## 4. 数据校验与全局异常处理
### 4.1 核心原理
把接口处理想成两道检查：

```text
第一道：请求能不能读懂？（JSON 格式、数字能否转换）
第二道：读懂的数据合不合理？（数量必须大于 0、名称不能为空）
两道都通过后，才进入 Controller/业务处理；任一道失败，就返回清楚的错误信息。
```

Bean Validation 注解负责第二道检查。注解要放在请求 DTO 的字段上，并在 Controller 参数上用 `@Valid` 或 `@Validated` 触发。例如“字段不能为空”与“字符串不能只有空格”不是一回事，所以分别有 `@NotNull` 和 `@NotBlank`；集合要至少有一个元素可用 `@NotEmpty`，长度范围可用 `@Size`。对象里还包含子对象时，在子对象字段上再加 `@Valid`，才会继续检查里面的字段。

**Bean Validation 常用注解**：`@NotNull`、`@NotBlank`（字符串）、`@NotEmpty`（集合/字符串）、`@Size`、`@Min/@Max`、`@DecimalMin/@DecimalMax`、`@Pattern`、`@Email`、`@Past/@Future`、`@Positive`、`@Digits`；嵌套对象需在字段上加 `@Valid` 才会级联校验；集合元素校验需 `List<@Valid Item>` 或字段参数级 `@Valid`。Boot 3 使用 `jakarta.validation.*` 命名空间，Boot 2 通常使用 `javax.validation.*`，两者不能混用。

| 维度 | @Valid | @Validated |
| --- | --- | --- |
| 来源 | Bean Validation 标准（Boot 3 常用 `jakarta.validation`；Boot 2 常用 `javax.validation`） | Spring（`org.springframework.validation.annotation`） |
| 分组校验 | 不支持 | **支持 groups** |
| 位置 | 方法参数、字段、方法（级联） | 类、方法参数、字段 |
| 方法级校验 | 用于触发级联校验；具体执行机制随 Spring 版本变化 | 支持分组；也常用于 Spring AOP 方法校验 |
| 常用场景 | 嵌套/级联校验 | 分组 + 方法参数校验 |

**全局异常处理**：`@RestControllerAdvice` + `@ExceptionHandler` 拦截指定异常并返回统一响应体；继承 `ResponseEntityExceptionHandler` 可接管 Spring MVC 内置异常（参数绑定失败、405、415、404 等，Spring 6 中方法签名以官方文档为准）；Spring 6 支持 `ProblemDetail`（RFC 7807）。

**方法参数校验的版本差异**：Spring Framework 6.1 起，Spring MVC 对 Controller 方法参数约束提供内建方法校验（例如 `@RequestParam @Min(1) int page`）；这与 `@RequestBody @Valid` 的对象属性校验是两条路径，内建方法校验可能抛 `HandlerMethodValidationException`。Controller 若使用类级 `@Validated`，通常会改走 Spring AOP 方法校验，可能影响 MVC 的异常处理行为。旧版 Spring 常见做法是在 Controller 类上加 `@Validated` 并通过 AOP 校验查询参数。要根据 Spring Framework/Boot 版本选择一种机制并覆盖对应异常类型，不要把不同版本的错误类型写成唯一结论。

**区分异常发生在哪一层**：JSON 语法错误通常是 `HttpMessageNotReadableException`；query/path 类型转换失败常见为 `MethodArgumentTypeMismatchException`；Bean Validation 对 `@RequestBody @Valid` 的对象校验失败常见为 `MethodArgumentNotValidException`；方法参数约束校验在旧版/AOP 路径可能产生 `ConstraintViolationException`，Spring MVC 6.1+ 内建校验常见 `HandlerMethodValidationException`。这些失败可能发生在 Controller 方法体之前。业务异常则由应用主动抛出。认证过滤器中发生的错误在 MVC 异常解析器之外，应由 Filter/Spring Security 认证入口写响应，不能指望 `@ControllerAdvice` 捕获。

**异常映射的设计**：把异常转成稳定的公开错误模型（状态码、机器可读错误码、面向用户的消息、可选 traceId），不要把 Java 类名、SQL、堆栈、内部主机名返回给客户端。日志记录一次即可，区分预期的 4xx（通常不需要 error 堆栈）与意外的 5xx（需带 traceId 和异常原因）。若采用 RFC 7807，可通过 Spring 6 `ProblemDetail` 表达 `type/title/status/detail/instance`；响应格式应与客户端及网关约定一致。

例如“创建订单数量传成 `"many"`”会先在请求转换/绑定时失败，Controller 可能还没运行；数量传成 `0` 则能读成数字，但业务校验不通过；订单库存不足则通常是业务层抛出的业务异常。全局异常处理器的工作是把这些不同来源的错误翻译成一致、不会泄露内部细节的 HTTP 响应，而不是让错误消失或一律返回 200。
### 4.2 面试问答
**Q1：@Valid 和 @Validated 的区别？** A：@Valid 是 Jakarta Bean Validation 标准注解，常用于触发对象及嵌套对象校验；@Validated 是 Spring 扩展，支持分组，也常用于 Spring AOP 方法校验。Controller 查询参数校验的机制有版本差异：Spring 6.1+ 有 MVC 内建方法校验；旧版常通过类级 `@Validated` AOP 实现。DTO 可用 `@Valid` 或 `@Validated(组.class)`，不要在不核对版本的情况下机械地给所有 Controller 加类级 `@Validated`。
**Q2：分组校验怎么用？** A：定义标记接口 `interface Create {}`、`interface Update {}`，在字段上写 `@NotNull(groups = Update.class)`，Controller 参数写 `@Validated(Update.class)`；默认组为 `Default`，想在校验指定分组时同时校验默认组需继承 `Default`（Spring 提供 `@GroupSequence` 做顺序校验）。
**Q3：嵌套校验为什么没生效？** A：因为少了 `@Valid`。Bean Validation 不会自动深入嵌套对象，必须在嵌套字段上显式加 `@Valid`（集合元素校验同理）。这是"校验注解写了但没拦下来"的第一大原因。
**Q4：全局异常处理器怎么写才不漏？** A：① `@RestControllerAdvice` 一个类统一处理，按异常类型精确到粗粒度分层（业务异常、参数异常、认证授权异常、兜底 Exception）；② 继承 `ResponseEntityExceptionHandler` 覆盖 Spring MVC 内置异常，避免框架异常绕过你的统一格式；③ 兜底处理要打日志（含 traceId）但**不要对外暴露堆栈**；④ 注意 `@ExceptionHandler` 的匹配优先级是最具体类型优先。
### 4.3 高频追问与陷阱
- 陷阱：`@ExceptionHandler` 无法捕获 Filter 层异常（Filter 在 MVC 之外），也不能捕获异步线程内抛出的异常（需异步异常处理器）。
- 陷阱：把兜底 `Exception` 处理器的 HTTP 状态码统一返回 200，会让网关/监控无法识别真实错误率。
### 4.4 代码/配置示例
```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    @ExceptionHandler(BizException.class)
    public ResponseEntity<ApiResult<Void>> biz(BizException e) {
        return ResponseEntity.status(e.getStatus()).body(ApiResult.fail(e.getCode(), e.getMessage()));
    }
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException e, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ":" + f.getDefaultMessage()).collect(Collectors.joining("; "));
        return ResponseEntity.status(status).headers(headers)
                .body(ApiResult.fail("PARAM_INVALID", msg));
    }
}
```
`ResponseEntityExceptionHandler` 已集中声明 Spring MVC 内置异常的处理映射；继承它时，应覆盖相应 `handle...` 方法，而不要再为相同异常另加一个 `@ExceptionHandler`，避免映射冲突。上例的 `HttpStatusCode` 是 Spring 6 API；Spring 5 项目使用的签名不同，应按所用版本调整。`ApiResult` 是应用自定义响应类型。
### 4.5 一句话速记
@Valid 管嵌套级联、@Validated 管分组与方法级校验；统一异常用 @RestControllerAdvice + ResponseEntityExceptionHandler。

## 5. 拦截器 vs 过滤器 vs AOP
### 5.1 核心原理

记住它们分别站在三道不同的“门”上：

```text
HTTP 请求
  → Filter：Servlet 门口，DispatcherServlet 还没接手
  → DispatcherServlet / HandlerInterceptor：MVC 路由门口，已经知道要调用哪个 Controller
  → AOP：Spring Bean 方法周围，不一定是 HTTP 请求
```

例如 traceId 可以在 Filter 一进应用时生成，因为这时还没找到 Controller；如果要根据 `@AdminOnly` 这类 Controller 方法注解做额外检查，Interceptor 已经能看到目标 Handler；事务则应该放在 Service 层代理（AOP）处理，因为定时任务或消息消费者调用 Service 时也需要事务，不只 HTTP 请求需要。

| 维度 | Filter | HandlerInterceptor | AOP 切面 |
| --- | --- | --- | --- |
| 规范/层级 | Servlet 规范，容器级 | Spring MVC 级 | Spring 容器级 |
| 执行位置 | DispatcherServlet 之前/之后 | DispatcherServlet 内、Handler 前后 | 被代理的 Bean 方法周围 |
| 能否拿到 HandlerMethod | 否 | 是（可读注解做鉴权） | 否（只有方法签名） |
| 能否拿到 Bean | 能（若由容器注册） | 能 | 能 |
| 能否改请求/响应体 | 能（用 Wrapper 包装） | 一般不能 | 不能 |
| 触发范围 | 所有请求（含静态资源） | 匹配的 Handler 映射 | 匹配的方法调用（含非 Web 调用） |
| 典型用途 | 编码、CORS、TraceId、请求体日志、XSS 过滤 | 登录校验、权限、限流、参数预处理 | 事务、缓存、日志耗时、分布式锁 |

**HandlerInterceptor 三个方法时机**：`preHandle`（Handler 执行前，返回 false 直接中断链）、`postHandle`（Handler 执行后、视图渲染前；@ResponseBody 场景响应体已写入，改 Model 无效）、`afterCompletion`（视图渲染完成后执行，**异常也会执行**，适合清理 ThreadLocal）。

**怎么选择扩展点**：Filter 是 Servlet 容器的链，适合在 MVC 之前统一处理 URL、请求/响应包装、字符编码、CORS 等；它不知道最终匹配到哪个 Controller。Interceptor 属于 Spring MVC，能看到 Handler（通常是 `HandlerMethod`），适合按 Controller 注解做审计或请求级检查；它看不到未进入 DispatcherServlet 的请求，也不是完整的安全边界。AOP 围绕 Spring Bean 方法执行，即使不是 HTTP 请求也会触发，适合事务、服务层审计、缓存等。响应体已经由 Converter 写出的场景，不要在 `postHandle` 修改；统一改写序列化结果优先考虑 `ResponseBodyAdvice`。

**安全边界和异步注意事项**：鉴权授权优先使用 Spring Security 的 Filter 链，而不是只依赖 MVC 拦截器——静态资源、错误派发、其他 Servlet 等可能不经过同一 MVC 映射。不要只凭 `HandlerMethod` 上是否存在某个注解就做完整权限判断，仍需校验当前主体、资源归属和权限。清理 MDC/ThreadLocal 必须覆盖正常、异常及异步 dispatch；线程池会复用线程，遗漏清理会把前一个请求的上下文带给后一个请求。
### 5.2 面试问答
**Q1：拦截器和过滤器有什么区别？** A：Filter 属于 Servlet 规范、在 DispatcherServlet 之前，能操作原始请求/响应流；Interceptor 属于 Spring MVC、在 DispatcherServlet 内，能拿到 HandlerMethod 与注解，方便做基于注解的鉴权/限流。选择：需要改 body/编码/粗粒度过滤用 Filter；需要基于 Controller 方法元数据做控制用 Interceptor；与 Web 无关的横切（事务、缓存）用 AOP。
**Q2：afterCompletion 在异常时执行吗？** A：执行。`preHandle` 返回 true 的拦截器，其 afterCompletion 一定会在请求结束时被调用（含异常与异步完成后），因此清理 MDC/ThreadLocal 放这里最合适。注意若 `preHandle` 返回 false，则只有已成功执行的拦截器（不含它自己）会执行 afterCompletion。
**Q3：拦截器里能修改返回值吗？** A：不能直接修改 @ResponseBody 的结果。需要改响应内容要么用 Filter + ResponseWrapper，要么用 AOP 环绕 Controller，要么用 `ResponseBodyAdvice`（Spring MVC 提供的响应体统一处理扩展点，适合统一包装/脱敏）。
**Q4：静态资源请求会走拦截器吗？** A：取决于映射。默认静态资源由 `ResourceHttpRequestHandler` 处理，也会经过拦截器链（拦截器按路径匹配），所以登录拦截器要把静态资源与健康检查路径排除（`excludePathPatterns`），否则页面样式/探针会被拦。
### 5.3 高频追问与陷阱
- 陷阱：拦截器里 `@Autowired` 注入的 Bean 在注册方式不对时可能为 null（`new` 出来的拦截器不归容器管，需用 `WebMvcConfigurer#addInterceptors` 注册 Bean）。
- 追问：`ResponseBodyAdvice` 是实现"统一响应体/字段脱敏"的正统扩展点，比 Filter 包装流更安全。
### 5.4 代码/配置示例
```java
@Component
public class AuthInterceptor implements HandlerInterceptor {
    @Override public boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler) {
        if (handler instanceof HandlerMethod hm && hm.hasMethodAnnotation(PublicApi.class)) return true;
        if (!tokenService.valid(req.getHeader("Authorization"))) { resp.setStatus(401); return false; }
        return true;
    }
    @Override public void afterCompletion(HttpServletRequest r, HttpServletResponse s, Object h, Exception e) {
        MDC.clear();          // 清理，避免线程池复用串号
    }
}
@Configuration
public class WebConfig implements WebMvcConfigurer {
    private final AuthInterceptor authInterceptor;
    public WebConfig(AuthInterceptor authInterceptor) { this.authInterceptor = authInterceptor; }

    @Override public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**").excludePathPatterns("/api/login", "/actuator/**");
    }
}
```
**对象必须由容器管理**：上例通过构造器注入 Spring 管理的 `AuthInterceptor`，而不是 `new AuthInterceptor()`；否则其依赖（如 `tokenService`）不会被注入。真实登录授权还应由 Spring Security 或明确的安全过滤链完成，示例仅展示拦截器注册方式。
### 5.5 一句话速记
Filter 在容器层（能改流）、Interceptor 在 MVC 层（能拿 HandlerMethod）、AOP 在 Bean 层（与 Web 无关的横切）。

## 6. 跨域 CORS
### 6.1 核心原理
例如 CRM 页面运行在 `https://crm.example.com`，API 是 `https://api.example.com`。虽然它们可能属于同一家公司，但浏览器看到 host 不同，就按跨源处理。浏览器先发请求或预检，再根据 API 返回的 CORS 响应头决定是否允许网页 JavaScript 读取结果。

**同源策略**：协议 + 域名 + 端口三者一致才同源；`http://a.com` 与 `https://a.com`、`http://a.com:8080` 都不同源。同源策略限制的是**读取响应**（跨域请求本身可能已发出），所以会出现"服务端已处理、前端拿不到结果"的现象。

**简单请求**：方法为 GET/POST/HEAD 且 Content-Type 仅限 `application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`，且无自定义头 → 浏览器直接发送。**预检请求**：不满足简单请求条件时，浏览器先发 `OPTIONS`，带 `Access-Control-Request-Method`/`Access-Control-Request-Headers`，服务端需返回 `Access-Control-Allow-Origin`、`Allow-Methods`、`Allow-Headers`、`Max-Age`，通过后才发真实请求。

| 方案 | 写法 | 适用场景 |
| --- | --- | --- |
| 方法/类级注解 | `@CrossOrigin(origins = "https://a.com", allowCredentials = "true")` | 少量接口 |
| 全局配置 | `WebMvcConfigurer#addCorsMappings` 或 `CorsFilter` Bean | 统一跨域策略（推荐） |
| 网关/Nginx | 反向代理 + `add_header Access-Control-*` | 生产常用，业务零改动 |

**带 Cookie 的跨域**：`Access-Control-Allow-Origin` **不能是 `*`**，必须回显具体 Origin，同时 `Access-Control-Allow-Credentials: true`，前端 `xhr.withCredentials = true`；另外 Cookie 需要 `SameSite=None; Secure`（HTTPS）才能跨站发送（Chrome 80+ 默认 SameSite=Lax）。

**浏览器如何决定是否放行**：服务端返回 CORS 头并不会阻止请求到达服务端；浏览器根据响应头决定是否把响应暴露给网页脚本。预检只是浏览器在真正请求前询问“此来源能否用这个方法和这些头”，服务端对 OPTIONS 成功应答后浏览器才发真实请求。简单请求则会先发真实请求，即使响应没有允许头，浏览器仍可能阻止页面读取响应。因此 CORS 不是身份认证、CSRF 防护或防火墙，不能把“跨域不允许”当作接口访问控制。

最容易记的区别是：**CORS 是浏览器的读响应规则，不是服务器拒绝陌生客户端的安全门。** curl、Postman、服务间 HTTP 调用通常不受浏览器同源策略限制；因此“Postman 成功、浏览器报跨域”并不矛盾。带 Cookie 时浏览器会携带用户会话，所以服务端必须精确允许可信 Origin，不能一边允许凭证、一边用 `*` 放开所有网站。

**配置策略**：允许来源应采用精确白名单并规范化 Origin（含 scheme、host、port），不要用任意 `Origin` 原样回显；携带凭证时不能使用通配来源。若由 Gateway/Nginx 统一添加 CORS 头，下游应用不应再重复添加。若启用 Spring Security，要让 CORS 处理在认证拒绝之前正确生效；否则 OPTIONS 被 401 拦截，即使 MVC CORS 配置正确，浏览器仍会报预检失败。
### 6.2 面试问答
**Q1：跨域是谁限制的？** A：浏览器。服务端之间的调用（RPC、Feign、curl）没有同源策略。所以"Postman 能通、浏览器报 CORS"是正常现象。
**Q2：为什么预检请求会失败？响应头都配了啊？** A：常见原因：① OPTIONS 请求被鉴权拦截器拦下（返回 401/302），预检要求 2xx；② 网关或 Nginx 没有把 `Access-Control-*` 头透传/重复添加导致浏览器判定无效；③ `Allow-Origin` 用了 `*` 但又带 credentials；④ 自定义头没在 `Allow-Headers` 里列出；⑤ 重定向（302）导致预检失败。解法：放行 OPTIONS、统一在一层（网关或应用）加 CORS 头，避免双层重复。
**Q3：`Access-Control-Max-Age` 有什么用？** A：缓存预检结果，减少 OPTIONS 请求次数（单位秒，浏览器对上限有各自限制，通常几分钟到几小时）。高并发下它能显著降低无谓的 OPTIONS 流量。
**Q4：生产上 CORS 一般放哪里配？** A：推荐网关或 Nginx 统一处理（业务无感、口径一致）；如果业务直接对外暴露，则在应用内用全局 `CorsFilter`/`WebMvcConfigurer`。**不要**在不同层重复添加 CORS 头，重复会导致浏览器报"multiple values"错误。
### 6.3 高频追问与陷阱
- 陷阱：跨域问题被误判为"接口没收到请求"——加日志看 OPTIONS 是否到达，能快速区分是浏览器拦截还是服务端拦截。
- 追问：`SameSite=Lax` 下，跨站 POST 不会带 Cookie，这是很多"登录后跨域 401"的根因。
### 6.4 代码/配置示例
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOriginPatterns("https://*.a.com")   // 允许携带凭证时不能用 "*"
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*").allowCredentials(true).maxAge(3600);
    }
}
```
### 6.5 一句话速记
同源策略限制的是浏览器读响应；预检 OPTIONS 必须返回 2xx；带 Cookie 时 Allow-Origin 不能为 `*`。

## 7. 文件上传下载与静态资源映射
### 7.1 核心原理
上传可以看成“浏览器把文件装进一个 multipart 请求，Spring 将其中的文件部分交给 `MultipartFile`”。应用随后要决定文件能不能收、存到哪里、用什么名字保存；`MultipartFile` 只是这次请求里的临时文件入口，不等于已经持久保存。

**上传**：Boot 自动配置 `MultipartResolver`（实际是 `StandardServletMultipartResolver` + Servlet 3.0 `Part` API），配置项 `spring.servlet.multipart.max-file-size`（单文件）与 `max-request-size`（整请求）默认各为 1MB/10MB，生产必须按业务调大。参数用 `MultipartFile` 接收，注意 `transferTo` 与临时文件清理、文件名安全（防目录穿越：`../`）、类型白名单、大小与数量限制、存储到对象存储而非本地磁盘（多实例部署）。

**下载**：用 `ResponseEntity<Resource>` 或 `StreamingResponseBody`，设置 `Content-Disposition: attachment; filename*=UTF-8''...`（中文名需 URL 编码），大文件要支持 Range（`ResourceHttpRequestHandler` 与 `ResourceRegion` 支持断点续传），并注意流式写出避免全量读入内存。

**静态资源映射**：Boot 默认映射 `classpath:/static/`、`/public/`、`/resources/`、`/META-INF/resources/` 到 `/**`；可用 `spring.web.resources.static-locations`（Boot 2.4+ 的键名，2.4 之前是 `spring.resources.static-locations`）与 `WebMvcConfigurer#addResourceHandlers` 自定义，并可配置 `spring.web.resources.cache.cachecontrol.max-age` 做缓存与版本化。

**上传的可信边界**：`MultipartFile#getOriginalFilename()` 和浏览器提供的 Content-Type 都是不可信输入。扩展名白名单只能作为第一层筛选，不能证明真实文件类型；对高风险格式应检测文件签名、限制解压后大小/文件数量、防止压缩炸弹，必要时病毒扫描。服务端生成存储名，文件放在 Web 根目录之外或对象存储，下载时再做鉴权；上传成功前不要把临时路径当成永久存储。上传请求大小还要同时考虑代理、网关、Servlet 容器和应用配置的限制。

**下载与资源映射的取舍**：`ResponseEntity<Resource>` 适合可控大小的文件；`StreamingResponseBody` 可边读边写，减少把整个文件装入堆的风险，但必须设置合理超时、处理客户端中断并控制并发。静态资源宜使用不可变版本 URL 配合较长缓存；可变资源需短缓存或校验器（ETag/Last-Modified）。不要把用户上传目录直接映射成可执行内容或公开静态目录。

可以把上传和下载分成两条相反的流水线：

```text
上传：浏览器文件 → 应用检查大小/类型/权限 → 生成安全存储名 → 存对象存储 → 数据库保存文件元信息
下载：客户端请求文件 ID → 应用校验是否有权访问 → 从存储读取 → 设置下载响应头 → 流式返回
```

不要让客户端决定磁盘路径，也不要因为知道文件 URL 就默认允许下载；“文件存在”和“当前用户有权看它”是两件事。
### 7.2 面试问答
**Q1：上传大文件报错怎么排查？** A：按顺序查：① `spring.servlet.multipart.max-file-size` 与 `max-request-size`（超限抛 `MaxUploadSizeExceededException`，可被 @ControllerAdvice 捕获返回友好提示）；② 反向代理限制（Nginx `client_max_body_size`、网关 body 大小限制）；③ 磁盘空间与临时目录（`java.io.tmpdir`）；④ 后端到对象存储的超时与重试。
**Q2：为什么生产不建议把上传文件存本地磁盘？** A：多实例部署时文件不共享（请求落到另一台就 404）、容器重建即丢失、扩容/迁移困难、备份复杂。推荐对象存储（OSS/S3/MinIO）+ CDN，业务库只存 URL；小规模单机可用 NAS 挂载，但要评估单点与容量。
**Q3：文件下载中文名乱码怎么处理？** A：用 RFC 5987 写法：`Content-Disposition: attachment; filename="fallback.txt"; filename*=UTF-8''%E4%B8%AD%E6%96%87.txt`，即对 UTF-8 文件名做百分号编码；只写 `filename=` 时浏览器按 Latin-1 解析会乱码。
**Q4：静态资源和接口冲突（比如都映射到 `/**`）会怎样？** A：优先级由 HandlerMapping 顺序决定——`RequestMappingHandlerMapping` 优先于 `SimpleUrlHandlerMapping`（静态资源用它），所以同路径时接口生效、静态资源被遮蔽；反之若接口路径写成 `/**` 通配会抢走所有请求。规范做法是接口统一加前缀（如 `/api`）。
### 7.3 高频追问与陷阱
- 陷阱：`MultipartFile.transferTo(File)` 在同一路径二次调用会抛异常（已移动的临时文件不可复用），需要 `Files.copy` 或先判断。
- 追问：`spring.web.resources.static-locations` 是 Boot 2.4+ 的键名，从 2.3 升级时旧键名会失效（启用 properties-migrator 可发现）。
### 7.4 代码/配置示例
```java
@PostMapping("/upload")
public String upload(@RequestPart("file") MultipartFile file) throws IOException {
    String ext = StringUtils.getFilenameExtension(file.getOriginalFilename());
    if (!Set.of("jpg", "png", "pdf").contains(ext == null ? "" : ext.toLowerCase()))
        throw new BizException("文件类型不允许");
    return ossClient.put(UUID.randomUUID() + "." + ext, file.getInputStream(), file.getSize());
}
```
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 100MB
  web:
    resources:
      static-locations: classpath:/static/,file:/data/static/
```
### 7.5 一句话速记
上传限大小 + 白名单 + 对象存储；下载用 Content-Disposition 的 filename\* 编码；静态资源别和接口抢 `/**`。

## 8. 会话管理、JWT 与 SSO
### 8.1 核心原理

先把四个词分开，后面就不容易混：

- **Cookie**：浏览器保存并在符合规则时自动带回服务器的一种“小信封”。它是凭证的携带/保存方式，本身不等于登录方案。
- **Session**：服务端保存的一份登录会话记录。浏览器通常只保存一个随机的 `sessionId`（常见 Cookie 名是 `JSESSIONID`）；服务端收到 ID 后，再查到“这是哪个用户、会话何时过期”。
- **Token**：客户端拿着、请求时交给服务端验证的凭证。在面试里说“Token 方案”时，常常特指**不透明随机 Token**：它本身不包含用户信息，服务端拿它去 Redis/数据库查记录。
- **JWT**：一种有固定结构、由服务端签名的 Token 格式。JWT 里可以带用户 ID、过期时间等声明，服务端验证签名后可读取声明，通常不必每次查会话表。**JWT 是 Token 的一种，不是和 Token 完全平级的另一个词。**

本章后面比较“Session、Token、JWT”时，`Token` 特指不透明随机 Token。三种方案的核心差别不是名字，而是：**每次请求时，服务端要不要去共享存储查询登录状态？**

补充一个常见术语：OAuth2 里的 **Access Token** 是“访问凭证”的统称，它可以是不透明随机串，也可以采用 JWT 格式；看到“Token”这个词时，不能直接假设它一定是 JWT。

| 维度 | Session | 不透明随机 Token | JWT |
| --- | --- | --- | --- |
| 登录后客户端拿到什么 | 随机 `sessionId` | 随机 Token 字符串 | 服务端签名后的 JWT 字符串 |
| 登录状态主要放哪里 | 服务端 Session（单机内存或 Redis） | 服务端映射表（通常 Redis：token → 用户/会话） | 通常放在 Token 声明里，由签名证明未被篡改 |
| 每次请求如何识别用户 | 用 `sessionId` 查 Session | 用 Token 查服务端记录 | 验签并检查声明，通常本地即可完成 |
| 服务端是否需要共享状态 | 多实例时需要共享 Session | 多实例时需要共享 Token 记录 | 不查状态时通常不需要共享会话表，但验签公钥/密钥必须正确分发 |
| 立即注销/踢人 | 删除或失效 Session，比较直接 | 删除 Token 对应记录，比较直接 | 纯 JWT 不易立即撤销；可用短过期、黑名单或服务端会话状态 |
| 客户端保存的大小 | 通常只有一个短 ID | 通常是一个短随机串 | 通常较长，包含声明和签名 |
| 常见使用方式 | 浏览器网站 | App、API 客户端、服务间凭证 | 多服务本地验签、短期访问凭证 |

#### 方案一：Session——“服务端有一本登录名册”

登录成功后，服务端创建会话记录，例如 `sessionId=随机值 → userId=1001、tenantId=crm-a、过期时间=...`，然后把 `sessionId` 发给浏览器。浏览器之后访问接口时自动带上 `JSESSIONID` Cookie；服务端按 ID 查记录，再知道请求者是谁。

```text
登录：账号密码 → 服务端验证 → 创建 Session → 返回 sessionId
请求：浏览器自动带 sessionId → 服务端查 Session → 得到用户身份
登出：服务端删除/失效 Session → 旧 sessionId 查不到有效会话
```

所以 Session 的重点是：**状态在服务端，浏览器拿的是索引号**。优点是注销、踢人、查看当前登录设备都比较直接；缺点是多台应用服务器必须能查到同一份 Session。单机放内存时，请求如果被负载均衡到另一台机器，就可能查不到；常见解决方式是粘滞会话（有扩缩容和发布风险）或 Spring Session + Redis 共享会话。

#### 方案二：不透明 Token——“把 sessionId 换成 API 凭证”

服务端登录成功后生成一个不可预测的随机字符串，比如 `tkn_...`，把它和用户/有效期存到 Redis。客户端保存这个字符串，并在请求时主动带上：

```http
GET /api/orders
Authorization: Bearer tkn_random_value
```

服务端收到 Token 后去 Redis 查记录，查到才知道对应用户；登出时删除 Redis 记录，旧 Token 随即失效。它和 Session 的关键点很像：**服务端都保存登录状态，每次请求都要查服务端记录**。差异主要是使用接口：Session 常结合浏览器 Cookie 和 `HttpSession`；不透明 Token 常作为显式 API 凭证放在 `Authorization` 请求头，适合 App 或 API 客户端。

Token 字符串本身没有可读的用户信息，所以也叫“不透明”。它泄露后别人可以冒充持有人，必须使用足够随机的值、HTTPS、短有效期，并做好撤销和存储保护。多个应用实例需要访问同一份 Token 记录（例如 Redis），代价与共享 Session 相似。

#### 方案三：JWT——“凭证里带声明，靠签名验真”

JWT 通常由认证服务签发，内部包含声明，例如用户 ID、签发方、接收方和过期时间。客户端把 JWT 带给业务服务；业务服务用可信密钥验证签名和声明，确认凭证由可信方签发、未被修改且仍有效，然后读取用户身份。

```text
登录：认证服务验证账号 → 生成并签名 JWT → 返回客户端
请求：客户端带 JWT → 服务验签 + 检查 iss/aud/exp → 读取 sub 等声明
登出：客户端删除 JWT；但如果 JWT 未过期，服务端通常仍会接受它
```

JWT 的优点是：在不查共享会话存储的前提下，多个服务可以各自验签，减少每次请求的 Redis 查询；代价是**签发后不容易立即收回**。如果用户被封禁、权限刚被撤销或设备丢失，已签发的 JWT 在过期前可能仍有效。常见折中是短效 Access Token + 可撤销 Refresh Token；需要立即封禁时使用 `jti` 黑名单或服务端会话状态，但这样会重新引入每次查状态的成本。

#### 三种方案用同一个 CRM 例子比较

假设员工登录 CRM 后查看客户资料：

```text
Session：
请求带 JSESSIONID → CRM 查共享 Session → 确认员工和租户 → 查询客户

不透明 Token：
请求带 Authorization: Bearer <随机串> → CRM 查 Redis Token 记录
→ 确认员工和租户 → 查询客户

JWT：
请求带 Authorization: Bearer <JWT> → CRM 验签并检查过期/受众
→ 从声明取得员工身份 → 查询客户
```

Session 和不透明 Token 都是“服务端名册/登记表”模式：可以立即撤销，但依赖共享存储；JWT 是“签名凭证”模式：可以少查一次状态，但撤销和权限实时更新更麻烦。无论用哪种方式，**登录只证明身份，不自动证明用户有权看任意客户**；客户数据仍要按当前用户、角色和租户做授权过滤。

#### Cookie、请求头与 CSRF/XSS：这是凭证放在哪里的问题

Session ID、随机 Token、JWT 都是凭证；它们可以通过不同方式传送。浏览器自动附带 Cookie 的特点让会话使用方便，但恶意网站也可能诱导浏览器带 Cookie 发请求，所以要考虑 CSRF，并配置 `SameSite`、CSRF Token 或 Origin 校验。若把 Token 存在网页 JavaScript 可读取的位置，XSS 脚本可能窃取它；`HttpOnly` Cookie 可阻止 JavaScript 读取，但 Cookie 自动发送的 CSRF 风险仍要处理。

因此，“用 JWT 就安全”或“放 Authorization 头就绝对安全”都不准确。安全要同时考虑凭证的存放位置、是否自动发送、XSS/CSRF、HTTPS、过期时间、权限范围和撤销方案。

**JWT 结构**：`Header.Payload.Signature`，Base64URL 编码；Header 声明算法（HS256 对称 / RS256 非对称），Payload 放标准声明（iss、sub、exp、iat、jti）与自定义声明，Signature 用密钥对前两段签名。**注意 payload 只是编码不是加密**，不能放敏感信息。

**续期方案**：① 双 Token——短效 access token + 长效 refresh token（refresh 只用于换新，可撤销）；② 滑动过期——Redis 记录 token→过期时间，每次请求刷新 TTL（本质变成有状态）；③ 网关续签——统一在网关校验并下发新 token。

**分布式 Session 三种方案**：① 会话粘滞（Nginx ip_hash，扩容/迁移时丢会话，不推荐）；② 会话复制（Tomcat 集群广播，网络开销大，仅小集群）；③ **集中存储**（Spring Session + Redis，主流方案；Boot 3 需对应 Spring Session 版本，`spring-session-data-redis`）。**SSO/OAuth2 简述**：SSO 让一次登录访问多个系统（CAS 是典型协议，核心是中心认证 + 票据校验）；OAuth2 是**授权**框架（授权码、隐式、密码、客户端凭证四种模式，OAuth 2.1 已移除隐式与密码模式），OIDC 在其上补了身份认证（id_token）。企业内做登录+授权通常用 "OAuth2/OIDC + 网关统一鉴权"。

把 SSO 流程想成“多个系统共用一个公司的门卫”：用户第一次访问 CRM 时跳到统一认证中心登录；认证中心确认身份后给浏览器一个票据；用户再访问工单系统时，工单系统也跳到同一认证中心，认证中心看到已有登录状态，就不用用户重新输密码，而是给工单系统签发/验证它所需的票据。OAuth2 主要回答“某个客户端能获得什么访问权限”；OIDC 在 OAuth2 上增加了标准化的“用户是谁”的身份信息。OAuth2/OIDC 不是把 JWT 变成自动安全，也不意味着所有应用都应该自己实现授权服务器，生产中通常使用成熟身份平台或 Spring Security 相关组件。

**JWT 校验不是只看签名**：服务端还要检查允许的算法、签名密钥、`iss`（签发方）、`aud`（接收方）、`exp`/`nbf`（有效时间）及必要的权限声明；不能信任令牌 Header 自报的算法。JWT 的 Base64URL 只是编码，不提供保密性。令牌放 `Authorization: Bearer` 与放 Cookie 的 CSRF 风险不同：浏览器自动附带 Cookie 可能触发 CSRF；放在可被脚本读取的存储中则更易受 XSS 窃取。应根据客户端形态选择存放方式并配合 HTTPS、短有效期、最小权限和安全的刷新流程。

**Session 的生命周期**：浏览器持有 sessionId，服务端按此 ID 查找会话属性；注销通常使服务端记录失效。集中式 Session 让多个应用实例访问共享存储，但引入 Redis 可用性、序列化兼容性、TTL 续期和网络开销问题。Spring Boot 不同版本对 `spring.session.store-type` 等属性的自动检测策略有变化；Boot 3 常见方式是引入兼容的 Spring Session Redis starter 后由自动配置启用，不要未经版本核对照抄旧版配置键。

选择时可以按需求判断：浏览器传统网站、需要服务端管理登录状态时，Session + 共享存储是直观方案；App/API 想显式携带凭证、又需要随时撤销时，不透明 Token + Redis 也很合适；多个服务希望在不查询统一会话存储的情况下验证身份时，可以考虑短期 JWT，但必须接受撤销不即时、声明可能短暂过时的代价。三者都需要保护凭证并做资源授权；JWT 不是加密的用户资料袋，payload 通常可被解码读取，不要放密码、密钥或敏感个人信息。
### 8.2 面试问答
**Q1：Session、不透明 Token 和 JWT 的区别？** A：Session 用随机 sessionId 找服务端会话；不透明 Token 用随机凭证查服务端 Token 记录；JWT 把声明放在签名后的凭证里，服务端可本地验签。Session 与不透明 Token 都容易立即撤销，但多实例时需要共享状态；JWT 可减少状态查询，但撤销、权限实时更新和登出更复杂。
**Q2：JWT 登出怎么做？** A：JWT 本身无状态，登出只能：① 客户端删除 token（防不住已泄露的 token）；② 服务端维护黑名单（Redis 存 jti 到过期，代价是有状态）；③ 用短 access + 可撤销 refresh 的组合，把"撤销"放在 refresh 层。
**Q3：Cookie 的安全属性有哪些？** A：`HttpOnly`（脚本不可读，降低 XSS 直接窃取 Cookie 的风险）、`Secure`（仅 HTTPS 发送）、`SameSite`（控制跨站发送，降低 CSRF 风险）、`Domain/Path`（作用域）、`Max-Age/Expires`（有效期）。浏览器应用可以考虑 HttpOnly Cookie，但仍要设计 CSRF 防护；若用 JavaScript 可读存储，则要特别重视 XSS。
**Q4：CSRF 和 XSS 与这套机制的关系？** A：XSS 是攻击者让页面运行恶意脚本，可能读取脚本可访问的 Token 并代用户操作；防御包括输出转义、CSP、减少可读凭证。CSRF 是利用浏览器自动携带 Cookie 发请求；防御包括 SameSite、CSRF Token 和 Origin 校验。只有在凭证由客户端显式放入 `Authorization` 请求头、而不是浏览器自动携带时，通常才较不易受到传统 Cookie 型 CSRF；XSS 风险仍然存在。
**Q5：Spring Session + Redis 的原理？** A：用 `SessionRepositoryFilter` 包装 request/response，把 `HttpSession` 实现替换为 Redis 支持的实现（key 含 sessionId，写入时序列化），从而多实例共享；配合 `spring.session.redis.namespace` 隔离环境。注意序列化方式（JDK 序列化 vs JSON）会影响兼容性与安全。
### 8.3 高频追问与陷阱
- 陷阱：Session 粘滞方案在扩容/发布时导致用户被踢出，是"发布后大批 401"的常见原因。
- 陷阱：JWT 用对称密钥在多服务场景下需要共享密钥，泄露风险高；推荐 RS256 由认证中心签名、业务服务只持公钥验签。
### 8.4 代码/配置示例
```java
// JJWT API 在不同大版本间有变化；此处展示签发时应包含的声明，而非可跨版本直接编译的完整工具类。
public String issue(Long userId) {
    return Jwts.builder()
        .setSubject(String.valueOf(userId))
        .setId(UUID.randomUUID().toString())            // jti，用于黑名单
        .setIssuedAt(new Date())
        .setExpiration(Date.from(Instant.now().plus(30, ChronoUnit.MINUTES)))
        .signWith(SignatureAlgorithm.RS256, privateKey)  // 非对称签名
        .compact();
}
```
```yaml
spring:
  session:
    store-type: redis
    redis:
      namespace: myapp:session
  data:
    redis:
      host: 127.0.0.1
```
### 8.5 一句话速记
Session ID 和不透明 Token 都查服务端状态、便于撤销；JWT 通常本地验签、少查状态但难即时撤销；OAuth2 是授权，OIDC 才补充身份认证。

## 9. Servlet 容器：Tomcat 架构与调优
### 9.1 核心原理
Tomcat 可以想成一个大型服务大厅：Connector 是接电话/收网络请求的入口，Engine/Host/Context/Wrapper 则是逐层找到“哪个应用、哪个 Servlet 来处理”。Spring Boot 内嵌 Tomcat 时，Spring MVC 的 `DispatcherServlet` 就是其中一个 Servlet。

**Tomcat 整体架构**：`Server`（整个容器）→ `Service`（把 Connector 与 Engine 绑定）→ 一侧是 `Connector`（负责网络协议与 IO，内部由 `ProtocolHandler` + `Endpoint` + `Processor` 组成），另一侧是 `Engine` → `Host`（虚拟主机）→ `Context`（一个 Web 应用）→ `Wrapper`（一个 Servlet）。

把名称按“从大到小”记：

```text
Server：整个 Tomcat 进程
  └─ Service：把网络入口和应用处理部分组合起来
      ├─ Connector：监听端口、接收并解析 HTTP
      └─ Engine：负责分发到 Web 应用
          └─ Host：一个域名对应的虚拟主机
              └─ Context：一个 Web 应用
                  └─ Wrapper：一个 Servlet（Spring MVC 的 DispatcherServlet 也在这里）
```

**Connector 与 IO 模型**：`Http11NioProtocol`（NIO，Boot 默认）、`Http11Nio2Protocol`（NIO2/AIO 风格，Linux 上优势不明显）、APR（基于本地库，性能好但需装依赖，**在较新版本中已不推荐甚至移除，以官方文档为准**）。

| 参数（Tomcat 名 / Boot 属性） | 含义 | 说明 |
| --- | --- | --- |
| maxThreads / server.tomcat.threads.max | 处理请求的最大工作线程数 | Tomcat 默认 200；按 QPS×RT 估算，结合压测调整 |
| minSpareThreads / threads.min-spare | 最小空闲线程 | Tomcat 默认 10 |
| acceptCount / server.tomcat.accept-count | `maxConnections` 达到后，操作系统可排队的连接请求数 | Tomcat 默认 100；队列满后新连接可能被拒绝或连接超时 |
| maxConnections / server.tomcat.max-connections | 同时可接受并处理的连接数 | NIO 下 Tomcat 默认 8192 |
| connectionTimeout / server.tomcat.connection-timeout | 等待请求行/头的超时 | Tomcat 默认 60s（Boot 属性未设时沿用容器默认，以官方文档为准） |
| server.tomcat.max-http-form-post-size | POST 表单大小上限 | 默认 2MB |
| server.tomcat.threads.max-queue-capacity | Tomcat 执行器的工作任务队列容量（支持该属性的 Boot/Tomcat 版本） | 与操作系统连接 backlog 的 `accept-count` 不是同一个队列 |

**Tomcat 与 Undertow 对比**：Undertow 基于 XNIO，轻量、内存占用低、内置对 HTTP/2 与 WebSocket 的良好支持，阻塞式 Servlet 场景与 Tomcat 差距不大；Tomcat 生态与文档最成熟、排障资料最多、与各类框架贴合最好。**结论：默认用 Tomcat；资源敏感或需要更轻量运行时可选 Undertow/Jetty，但要以压测数据说话。**

**连接处理和业务线程是两件事**：以 Tomcat NIO Connector 为例，Acceptor 接收 TCP 连接，Poller 使用 Java NIO Selector 关注大量连接的读写就绪事件；当一条连接上有完整 HTTP 请求可处理时，Tomcat 将处理工作交给 Executor 中的工作线程。Servlet/Spring MVC Controller 默认是同步调用：工作线程会一直占用到方法返回并完成响应（等待 JDBC、Redis、Feign 时也占用）。所以“用 NIO”不代表 Controller 自动异步，也不等于线程数可以无限减少。

更直白地说：**连接数是“有多少通电话接进来”，工作线程数是“有多少客服正在处理问题”。**很多连接可以处于空闲等待状态，不必每条连接一直占一个业务线程；但同步 Controller 一旦开始处理，就会占用工作线程，直到方法结束。线程都忙时，请求只能排队；排队不是处理能力变强，只是把等待时间往后挪。

```text
客户端 TCP 连接
  -> Acceptor 接入连接
  -> Poller 监听读写就绪（NIO 可管理很多空闲连接）
  -> HTTP 解析 / Connector 交付 Servlet 请求
  -> 工作线程执行 Filter -> DispatcherServlet -> Controller
  -> 序列化响应并写回连接
```

`maxThreads` 限制同步请求处理工作线程；线程都忙时，Tomcat 仍可在 `maxConnections` 限制内接受连接并在服务器连接队列等待线程；达到 `maxConnections` 后，更多新连接进入操作系统 backlog，其大小由 `acceptCount` 影响。执行器任务队列容量（如支持的 `threads.max-queue-capacity`）则是另一个队列。队列越大不等于吞吐越高，只会增加排队延迟和资源占用。HTTP/1.1 keep-alive 连接可能空闲而不占工作线程；HTTP/2 一条连接可并发多个 stream，不能简单按连接数推算请求并发。具体参数默认值和实现以使用的 Tomcat 版本为准。

队列可以用“大厅、客服、等候区”记忆：`maxConnections` 限制大厅里同时保留多少连接；`maxThreads` 限制同时有多少个请求正在被工作线程处理；`acceptCount` 更接近操作系统接收新连接时的排队容量；Tomcat 执行器自身若配置任务队列，还会有一处不同的排队位置。任何一处队列满了都可能拒绝连接或让客户端超时。单纯把队列设得很大，往往只是让用户等更久，不会让服务处理得更快。

**线程池与下游容量要一起估算**：同步服务线程数大于数据库连接池时，请求会在连接池前排队；线程数过大又会增加上下文切换、堆栈内存和下游压力。用 Little 定律粗略估算平均并发 `L ≈ λ × W`（请求到达率 × 平均耗时），再通过压测观察 P95/P99、排队、拒绝、连接池等待和错误率确定上限。虚拟线程、Servlet 异步和响应式 I/O 会改变等待时线程的成本，但不会让数据库连接、CPU 或下游容量凭空增加；具体支持和开关随 Boot、Tomcat 版本而变。
### 9.2 面试问答
**Q1：Tomcat 的线程池怎么调？** A：先确定瓶颈类型：CPU 密集（线程数≈核数或核数×1~2）、IO 密集（线程数可放大，用压测找吞吐拐点）。再看 maxThreads（工作线程）、acceptCount（排队）、maxConnections（连接数）三者配比，并保证 `acceptCount` 有界以避免"请求无限排队、全部超时"。同时确认底层依赖（DB/Redis 连接池）不会成为新瓶颈。
**Q2：大量请求超时但 CPU 不高，可能是什么？** A：典型是"线程池被慢依赖占满"：maxThreads 被长耗时下游调用（无超时或超时过长）占住，导致排队时间超过客户端超时。排查手段：`/actuator/threaddump` 或 `jstack` 看线程栈是否集中在同一依赖；`/actuator/metrics/tomcat.threads.busy` 看线程占用；给所有外部调用设置超时（连接/读/总）与熔断限流。
**Q3：Tomcat 调优清单？** A：① 合理设置 threads.max 与 accept-count；② 外部调用超时（连接、读、总）必须配；③ 开启压缩（`server.compression.enabled`）减小响应体；④ 静态资源交给 Nginx/CDN；⑤ 合理设置 keepAlive 与 maxConnections；⑥ 关闭不必要的 access log 或改为异步；⑦ 结合 JVM（堆、GC）与操作系统（文件描述符、backlog）一起调；⑧ 一切以压测数据为准。
**Q4：maxConnections 与 maxThreads 的区别？** A：maxConnections 是**能同时建立的连接数上限**（超过则排队或按 acceptCount 排队），maxThreads 是**同时处理请求的线程数上限**。HTTP/1.1 keep-alive 下连接数通常远大于活跃请求数，所以 maxConnections 一般设得比 maxThreads 大；HTTP/2 多路复用会改变这个关系。
### 9.3 高频追问与陷阱
- 陷阱：`acceptCount` 设得很大并不能提高吞吐，只会让请求排队更久、客户端先超时——本质是"把过载伪装成慢"。
- 追问：Tomcat 的 `maxThreads` 与 Spring MVC 的异步请求（Callable/DeferredResult）配合时，业务线程来自另一个 TaskExecutor，容器线程可以被释放，这是异步的价值所在。
### 9.4 代码/配置示例
```yaml
server:
  tomcat:
    threads: { max: 400, min-spare: 20 }
    accept-count: 200
    max-connections: 10000
    connection-timeout: 5s
  compression:
    enabled: true
    mime-types: application/json,text/html,text/css,application/javascript
```
### 9.5 一句话速记
Connector 管协议与 IO、Engine 管容器层级；调优先给孩子依赖加超时，再调 maxThreads/acceptCount/maxConnections。

## 10. 异步请求、SSE 与 WebSocket
### 10.1 核心原理

同步请求像客服接电话后一直不挂断：即使在等数据库或第三方服务，这个请求线程通常也被占着。MVC 异步请求允许 Servlet 容器先释放原工作线程，等结果准备好后再派线程把响应写回；它减少的是“等待期间占着容器线程”，不是让工作本身消失，也不保证业务执行速度变快。

| 方式 | 返回类型 | 谁写结果 | 适用场景 |
| --- | --- | --- | --- |
| Callable | `Callable<T>` | Spring MVC 用 TaskExecutor 执行后自动回写 | 简单异步（把耗时逻辑交给线程池） |
| DeferredResult | `DeferredResult<T>` | **业务代码在其他线程调用 setResult** | 长轮询、与其他系统回调/事件联动 |
| WebAsyncTask | `WebAsyncTask<T>` | Spring 管理，可设超时与回调 | 需要超时与完成回调的异步 |
| ResponseBodyEmitter / SseEmitter | `SseEmitter` | 多次 `send()` | 服务端推送（SSE，单向、基于 HTTP） |
| StreamingResponseBody | `void` + 写 OutputStream | 直接写响应流 | 大文件/流式导出 |
| WebSocket | `WebSocketHandler` / STOMP | 双向 | 双向实时通信（IM、协同） |

关键机制：Servlet 3.0 异步（`request.startAsync()`）让容器线程在等待期间被释放；异步超时后 Spring 会重新 dispatch 并可用 `DeferredResult` 的 `onTimeout` 处理；**异步请求需要 Filter/Servlet 支持 async（`asyncSupported`）**，否则会报异常；默认异步超时由 `spring.mvc.async.request-timeout` 控制（以官方文档为准）。

**异步解决的是等待时占用线程，不是取消工作**：Callable 会使用 MVC 配置的异步执行器，DeferredResult 本身不创建业务线程，必须由应用的回调/MQ/线程池调用 `setResult` 或 `setErrorResult`。异步执行器应设置有界队列、并发上限、拒绝策略和监控；无界队列只会把过载延后并耗尽内存。请求超时或客户端断开后，也要尽量取消不再有价值的下游工作，但 Future 取消并不保证 JDBC 或远端调用一定停止。

**流式响应要考虑资源生命周期**：SSE 是单向文本事件流，需为每个连接设置心跳/超时、断线清理、代理缓冲策略和连接数预算；WebSocket 是双向协议，需要管理会话、认证、心跳、消息顺序及集群广播。二者都会长期占用连接资源，且负载均衡、网关 idle timeout 与滚动发布必须支持长连接。`StreamingResponseBody` 适合边读边写的大文件/导出，但业务代码应关闭输入流、处理客户端提前断开，并限制并发导出。

选择时可以先问“通信方向是什么、要不要一直连着”：

- 只需服务器持续推送进度/通知，客户端不需要通过同一通道频繁回传：优先考虑 SSE。
- 双方都要随时发消息，例如聊天、协同操作：考虑 WebSocket。
- 只是把一次较慢的 HTTP 请求交给别的线程处理：考虑 Callable/DeferredResult，不必为此改成 WebSocket。
- 大文件边生成边返回：考虑 `StreamingResponseBody`，同时限制并发和响应时间。
### 10.2 面试问答
**Q1：Callable 和 DeferredResult 的区别？** A：Callable 由 Spring MVC 把任务提交给配置的 `AsyncTaskExecutor` 并自动回写结果，业务不需要管结果设置；DeferredResult 由业务在**任意线程**（如 MQ 消费、回调、其他服务通知）调用 `setResult` 完成，适合"结果不由本请求线程产生"的场景，如长轮询、异步任务轮询。
**Q2：异步请求会占用容器线程吗？** A：不会一直占用。容器线程在处理完初始流程后释放，等待期间由异步机制挂起；结果就绪后由另一个线程（或容器派发线程）完成响应。**注意**：如果用的 TaskExecutor 线程池也被打满，异步只是把瓶颈从 Tomcat 线程搬到业务线程池。
**Q3：SSE 和 WebSocket 怎么选？** A：SSE 是**单向**服务端推送、基于 HTTP、天然支持断线重连与文本流，适合通知、进度、行情类场景；WebSocket 是**双向**、协议升级、适合 IM/协同编辑/游戏。SSE 的缺点是浏览器连接数限制与只能文本（二进制需编码），且需要处理网关/代理的缓冲。
**Q4：异步请求里 ThreadLocal（MDC、事务）会怎样？** A：不会自动传递。MDC 需要 `TaskDecorator` 或 `TransmittableThreadLocal` 手动透传；事务上下文绑在 ThreadLocal，异步线程**不在同一事务中**——这是"异步写库没回滚"的根因（详见[Spring 篇](./01-Spring核心IoC与AOP.md)）。
### 10.3 高频追问与陷阱
- 陷阱：SSE 长连接会被 Nginx/网关的读超时与缓冲影响，需要关闭代理缓冲（`proxy_buffering off`）并调大超时。
- 追问：异步请求的异常由 `AsyncUncaughtExceptionHandler` 或 `@ExceptionHandler(AsyncRequestTimeoutException.class)` 处理，普通异常处理器可能拦不到。
### 10.4 代码/配置示例
```java
@GetMapping("/orders/{id}/notify")
public DeferredResult<OrderVO> wait(@PathVariable Long id) {
    DeferredResult<OrderVO> dr = new DeferredResult<>(10_000L);   // 10s 超时
    dr.onTimeout(() -> log.warn("wait timeout, id={}", id));
    orderNotifier.register(id, dr);        // 其他线程/事件就绪时 dr.setResult(...)
    return dr;
}
@GetMapping(value = "/progress", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter progress() {
    SseEmitter emitter = new SseEmitter(0L);
    taskService.run(p -> { try { emitter.send(p); } catch (IOException e) { emitter.complete(); } });
    return emitter;
}
```
### 10.5 一句话速记
Callable 交给线程池、DeferredResult 交给业务线程设结果；SSE 单向推送、WebSocket 双向；异步不传递事务与 MDC。

## 11. 接口安全：幂等、限流、防重放与越权
### 11.1 核心原理

把安全问题放到真实业务里看会更容易区分：订单详情接口要防“看了别人的订单”（越权）；支付接口要防“同一笔钱被扣两次”（幂等）；签名 API 要防“抓到旧请求再发一次”（重放）；高峰接口要防“请求量超过系统承受能力”（限流）。这几个问题不同，不能指望一把 Redis 锁全部解决。

| 风险 | 攻击方式 | 防御手段 |
| --- | --- | --- |
| 越权 | 改 URL/参数里的 id 访问他人数据（水平）、访问高权限接口（垂直） | 服务端从会话取用户身份、数据权限过滤（SQL 层加 owner 条件）、URL 不可直接映射资源归属、接口级权限注解 |
| 重放 | 抓包后重复提交同一请求 | 时间戳 + nonce + 签名（HMAC），nonce 在 Redis 去重并设置窗口 |
| 参数篡改 | 篡改金额、数量等 | 签名校验；金额等关键值服务端重算 |
| 重复提交/幂等 | 网络重试、用户连点 | 唯一索引、幂等 token（先删后用）、状态机 + 乐观锁、分布式锁（见[分布式锁与分布式ID](../05-分布式与架构/03-分布式锁与分布式ID.md)） |
| 限流 | 恶意刷接口/爬虫 | 网关或应用层限流（令牌桶/漏桶/滑动窗口），见 [Nginx 与网关限流](../04-中间件/05-Nginx与网关限流.md) |
| 敏感数据泄露 | 日志/响应体带手机号、身份证 | 脱敏（响应序列化时打码）、日志脱敏、最小化返回字段 |

**签名设计要点**：参数按字典序拼接 + 时间戳 + 随机串 + 密钥 → HMAC-SHA256；服务端校验时间窗口（如 ±5 分钟）与 nonce 唯一性；密钥不落前端（App 场景可做签名混淆但不绝对安全）。**限流常用算法**：固定窗口（简单但有临界问题）、滑动窗口（精度与成本折中，Sentinel 用）、漏桶（恒定速率）、令牌桶（允许突发，Guava RateLimiter/Redis+Lua）。

签名/防重放按顺序理解：客户端把请求关键字段、时间戳和一次性随机值 `nonce` 按约定拼起来，用双方约定的密钥计算 HMAC 签名；服务端重新计算签名并比较，证明参数没有被改；检查时间戳是否太旧，减少旧请求被重放的时间窗口；再把 nonce 做短期唯一记录，拒绝同一个随机值再次出现。密钥必须留在可信服务端，放在浏览器或 App 包里的固定密钥可以被提取，因此不能把客户端签名当作绝对安全边界。

限流算法可以类比水流：固定窗口是每分钟数一次，但窗口边界可能突然放过两倍流量；滑动窗口把相邻时间段一起看，更平滑但要记录更多状态；漏桶以固定速度放请求，突发会排队或溢出；令牌桶平时积攒令牌，突发时可短暂多放行，但总体受令牌生成速度约束。选择算法时要先说清是要“平滑处理”还是“允许短时突发”，再结合压测定阈值。

**幂等不等于“加一个锁”**：目标是同一业务键重复提交时，只产生一次业务效果，并让重试方得到一致结果。Redis `SET NX` 可以暂时挡住并发请求，但 Redis 记录过期、服务在“扣款成功、写幂等结果前”崩溃、或处理超出 TTL 时，都可能再次执行；仅存一个“已处理”标记还可能让客户端拿不到第一次的结果。可靠写操作通常把幂等键与业务状态放在同一数据库事务中，用唯一约束作为竞争裁决，并缓存/保存首次响应；涉及外部支付、消息等副作用时再用对方幂等键、事务 Outbox 或状态机补偿。

```sql
-- 幂等键应限定调用方/业务范围，不能全局只按一个短字符串去重
CREATE TABLE api_idempotency (
  client_id VARCHAR(64) NOT NULL,
  idempotency_key VARCHAR(128) NOT NULL,
  request_hash VARCHAR(64) NOT NULL,
  state VARCHAR(16) NOT NULL,
  response_json TEXT,
  PRIMARY KEY (client_id, idempotency_key)
);
```
处理原则：首次请求插入 `PROCESSING` 并执行业务，在同一数据库事务成功时写 `COMPLETED` 和可重放响应；唯一键冲突时读取原记录，先核对 request hash，已完成则返回原结果，仍处理中则返回明确的处理中状态。若业务动作不在同一数据库事务内，不能假装这个表能提供跨系统原子性，应设计可恢复的中间状态与补偿。

几个名词可记成：

- **幂等**：同一个业务意图重试多次，最终效果仍像成功一次。比如网络超时后用户再次提交同一支付请求，不应重复扣款。
- **防重放**：同一份旧请求不能被攻击者拿来再次使用。时间戳限制请求新鲜度，nonce 让同一窗口内的随机标识只能使用一次，签名用于发现参数被篡改。
- **越权**：用户已经登录，但试图操作不属于自己的资源。登录成功不代表有权访问每一个订单；查询时仍要把当前用户/租户范围写进授权判断或 SQL 条件。
- **限流**：请求太多时按规则拒绝或排队，保护服务和下游，不是用来判断用户有没有权限。
### 11.2 面试问答
**Q1：怎么保证接口幂等？** A：按场景选：① 数据库唯一索引（最可靠，兜底）；② 幂等 token（下单前先取 token，提交时 Redis delete 成功才处理）；③ 状态机 + 乐观锁（`update ... where status = 'INIT'` 判断影响行数）；④ 分布式锁（并发量大时先锁再查）；⑤ 去重表/消息表（消费端幂等）。**关键是选一个落点并让它成为唯一真相**，不要依赖"前端只点一次"。
**Q2：水平越权怎么防？** A：核心是**永不信任客户端传入的身份与归属**：从会话/网关注入的上下文取 userId，查询时把 owner 条件写进 SQL（`where id = ? and user_id = ?`），而不是先查出来再判断（容易漏）；对外暴露的 ID 用不可枚举的 UUID/雪花 + 加盐哈希，避免遍历。
**Q3：防重放和幂等的区别？** A：防重放关注"同一请求被第三方重发"（安全视角，用时间戳+nonce+签名）；幂等关注"同一业务意图重复执行结果一致"（业务视角，用唯一键/状态机）。两者常配合：App 接口既有签名防重放，也有幂等键保证不重复下单。
**Q4：限流放在哪一层？** A：分层设防：① 接入层（Nginx/WAF）做 IP/连接级粗粒度限流；② 网关（Spring Cloud Gateway/Sentinel）做接口级限流与鉴权；③ 应用内做方法级/热点参数限流（Sentinel 注解）；④ 下游依赖（DB/Redis/第三方）做并发与超时保护。**只做一层都会被绕过或误伤**，且限流阈值必须来自压测。
### 11.3 高频追问与陷阱
- 陷阱：用 `if (record.userId != currentUserId) throw` 做越权校验时，容易漏掉列表/批量接口——列表接口必须在 SQL 层过滤。
- 陷阱：响应体直接返回实体导致敏感字段泄露（密码哈希、手机号），要用 VO/DTO 并显式选择字段。
### 11.4 代码/配置示例
```java
// 真实实现应在事务中做“唯一键抢占 + 业务写入 + 保存可重放结果”；
// 仅用 Redis SET NX 可以挡并发，但不能覆盖数据库提交与 Redis 写入之间的崩溃窗口。
@PostMapping("/payments")
ResponseEntity<PaymentResult> pay(@Valid @RequestBody PayCommand command,
        @RequestHeader("Idempotency-Key") String key) {
    return ResponseEntity.ok(paymentService.executeIdempotently(currentClientId(), key, command));
}
```
其中 `executeIdempotently` 的实现应以数据库唯一约束为最终裁决，并校验同一个 key 不可用于不同请求内容；不能在业务异常时无条件删除 key，因为异常可能发生在不可回滚的外部副作用之后。
### 11.5 一句话速记
越权永不信客户端身份（SQL 层过滤）、幂等靠唯一索引或 Redis SET NX、防重放靠时间戳+nonce+签名、限流分层做。

## 自测题（含答案要点）
1. **DispatcherServlet 九大组件？一次请求的处理顺序？** HandlerMapping、HandlerAdapter、HandlerExceptionResolver、ViewResolver、RequestToViewNameTranslator、LocaleResolver、ThemeResolver、MultipartResolver、FlashMapManager；顺序为 HandlerMapping → preHandle → HandlerAdapter（解析参数并调用 Controller）→ 返回值处理 → postHandle → 完成响应 → afterCompletion。
2. **Filter、Interceptor、Controller 的先后关系？** Filter 在 Servlet 容器层先执行；DispatcherServlet 随后查找 Handler 并执行 Interceptor 的 preHandle；最后才解析参数并调用 Controller。Filter 看不到最终 Handler，Interceptor 能看到 MVC Handler。
3. **Controller 线程安全吗？** 单例，不安全，不能放可变成员变量。
4. **Boot 3 路径匹配默认是什么？会踩什么坑？** PathPatternParser；`**` 只末尾、无后缀匹配、尾斜杠默认不等价。
5. **@RequestBody 与 @ModelAttribute 的区别？** 读消息体反序列化 vs 按参数名绑定（表单）。
6. **中文乱码三处源头？** 请求体编码、URL 编码、响应/页面编码（外加数据库连接字符集）。
7. **@Valid 与 @Validated 的区别？** 标准注解触发级联校验 vs Spring 扩展支持分组与方法级校验。
8. **嵌套校验为什么失效？** 嵌套字段缺 @Valid，Bean Validation 不会自动深入。
9. **Filter / Interceptor / AOP 的分层？** 容器级 / MVC 级 / Bean 级。
10. **afterCompletion 何时执行？** 请求结束（含异常）后，适合清理 ThreadLocal。
11. **简单请求与预检请求的判断条件？** 方法、自定义头与 Content-Type 均受限；不满足则先发 OPTIONS。
12. **带 Cookie 跨域的响应头要求？** Allow-Origin 必须是具体 Origin + Allow-Credentials: true；Cookie 需 SameSite=None; Secure。
13. **上传大小限制的配置项？** spring.servlet.multipart.max-file-size / max-request-size（默认 1MB / 10MB）。
14. **JWT 的三大缺点与登出方案？** 无法主动撤销、体积大、payload 可解码；登出靠客户端删 token + Redis 存 jti 黑名单，或短 access + 可撤销 refresh。
15. **分布式 Session 三种方案？** 粘滞、复制、集中存储（Spring Session + Redis 主流）。
16. **Tomcat 的 maxConnections 与 maxThreads 区别？架构层次是什么？** 连接数上限 vs 处理线程上限；层次为 Server → Service → Connector/Engine → Host → Context → Wrapper。
17. **幂等的落点有哪些？** 唯一索引、幂等 token、状态机+乐观锁、分布式锁、去重表。

## 背诵卡片
1. DispatcherServlet 九大组件：映射、适配、异常、视图、视图名、区域、主题、上传、Flash。
2. 请求流程：找 Handler → 适配器调用 → 拦截器 pre → 绑参 → 执行 → 出参 → post → 渲染 → afterCompletion。
3. 参数解析靠 HandlerMethodArgumentResolver，返回值靠 HandlerMethodReturnValueHandler。
4. Boot 3 默认 PathPatternParser：`**` 只末尾、后缀匹配关闭、尾斜杠不再等价。
5. RESTful：名词复数 + HTTP 方法 + 语义化状态码（201/204/400/401/403/404/409/429）。
6. query/form 用 @RequestParam，URI 用 @PathVariable，JSON 用 @RequestBody（只能一个）。
7. 乱码先查 Content-Type charset、encoding.force、数据库连接字符集。
8. @Valid 是标准注解，@Validated 是 Spring 的（groups、方法级校验）；嵌套校验必须加 @Valid。
9. 统一异常：@RestControllerAdvice + @ExceptionHandler + ResponseEntityExceptionHandler 兜底。
10. Filter 在容器层能改请求/响应流；Interceptor 在 MVC 层能拿 HandlerMethod。
11. HandlerInterceptor：preHandle / postHandle / afterCompletion（异常也执行）。
12. 清理 MDC 放 afterCompletion；静态资源与 /actuator 要进 excludePathPatterns。
13. CORS 是浏览器行为，服务端之间没有同源限制。
14. Allow-Origin 为 `*` 时不能携带凭证；预检 OPTIONS 必须返回 2xx。
15. SameSite=Lax 导致跨站 POST 不带 Cookie（登录态丢失常见原因）。
16. 上传必须限大小 + 白名单 + 对象存储，别存本地磁盘；下载中文名用 filename\* 的 RFC 5987 编码，大文件支持 Range。
18. 静态资源默认映射 classpath:/static/、/public/、/resources/、/META-INF/resources/。
19. Session 有状态可撤销；JWT 无状态难撤销、payload 明文可解，登出靠黑名单或短 access + 可撤销 refresh。
20. Cookie 安全属性：HttpOnly、Secure、SameSite；分布式 Session 用 Spring Session + Redis。
21. OAuth2 是授权框架，OIDC 才是认证；2.1 移除隐式与密码模式。
22. Tomcat 层次：Server → Service → Connector/Engine → Host → Context → Wrapper；Connector = ProtocolHandler + Endpoint + Processor。
24. Tomcat 默认 maxThreads 200、acceptCount 100、maxConnections 8192（NIO）。
25. 请求超时但 CPU 不高，先怀疑线程被慢依赖占满。
26. SSE 单向推送、WebSocket 双向；Callable 交给线程池、DeferredResult 由业务线程 setResult。
27. 异步不传递事务与 MDC，必须显式透传。
28. 越权防御：身份从会话取、SQL 层加 owner 条件、ID 不可枚举。
29. 幂等优先用数据库唯一索引兜底，接口层用幂等 token 或 SET NX。
30. 防重放 = 时间戳窗口 + nonce 去重 + 签名（HMAC-SHA256）。
31. 限流分层：接入层 → 网关 → 应用 → 下游依赖，阈值来自压测。
