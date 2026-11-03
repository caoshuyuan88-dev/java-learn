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

## 1. DispatcherServlet 与请求处理流程
### 1.1 核心原理
**九大组件**（Spring 5 的经典划分；Spring 6 中 Theme 相关能力已弱化，实际常用的是前八个，以官方文档为准）：HandlerMapping（映射 URL→处理器）、HandlerAdapter（适配不同类型的处理器）、HandlerExceptionResolver（异常解析）、ViewResolver（视图名→View）、RequestToViewNameTranslator（无视图名时推导）、LocaleResolver（国际化区域）、ThemeResolver（主题）、MultipartResolver（文件上传解析）、FlashMapManager（重定向携带参数）。

**一次请求的完整流程**：① 请求进入 DispatcherServlet#doDispatch；② 遍历 HandlerMapping 得到 `HandlerExecutionChain`（Handler + 拦截器链）；③ 通过 HandlerAdapter 执行 `preHandle`；④ 参数解析由 `HandlerMethodArgumentResolver` 完成（@RequestParam/@PathVariable/@RequestBody 等各有实现）；⑤ 执行 Controller 方法；⑥ 返回值由 `HandlerMethodReturnValueHandler` 处理（@ResponseBody 走 `HttpMessageConverter`，返回视图名走 ViewResolver）；⑦ `postHandle`；⑧ 渲染视图或写回响应体；⑨ `afterCompletion`。**中途任何异常都由 HandlerExceptionResolver 处理**，`@ExceptionHandler`/`@ControllerAdvice` 就是通过 `ExceptionHandlerExceptionResolver` 实现的。
### 1.2 面试问答
**Q1：DispatcherServlet 的工作流程？** A：收请求 → HandlerMapping 找 Handler 与拦截器 → HandlerAdapter 调用（先 preHandle）→ 参数解析 → 执行 Controller → 返回值处理（@ResponseBody 用 HttpMessageConverter 写 JSON，否则 ViewResolver 渲染）→ postHandle → 渲染 → afterCompletion。异常统一交给 HandlerExceptionResolver。
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

| 维度 | AntPathMatcher | PathPatternParser |
| --- | --- | --- |
| 默认启用 | Spring 5.3 之前 / Boot 2.5 及之前 | **Spring 6 / Boot 3 默认**（Boot 2.6 起改默认） |
| 匹配时机 | 每次请求解析一次 | 启动时预解析为模式树，性能更好 |
| `**` 语义 | 可出现在中间 | 只允许出现在末尾（更严格） |
| 后缀匹配 | 支持 `/user.json` | 默认关闭 |
| 尾斜杠匹配 | `/user/` 与 `/user` 等价 | Spring 6 中 `useTrailingSlashMatch` 废弃且默认 false |
| 配置项 | — | `spring.mvc.pathmatch.matching-strategy` |

**RESTful 设计要点**：用名词复数表示资源（`/orders`、`/orders/{id}`），用 HTTP 方法表达动作（GET 查、POST 增、PUT 全量改、PATCH 局部改、DELETE 删）；查询条件用 query 参数（分页 `?page=1&size=20`）；状态码语义正确（200 成功、201 创建、204 删除成功无返回、400 参数错、401 未认证、403 无权限、404 不存在、409 冲突、422 语义错误、429 限流、500 服务端错误、503 不可用）；统一响应体（`code`/`message`/`data`），并对**业务错误码与 HTTP 状态码做映射约定**。
### 2.2 面试问答
**Q1：Boot 3 的路径匹配变化会踩什么坑？** A：默认 PathPatternParser：`**` 只能放末尾、后缀匹配关闭、**尾斜杠默认不再等价**。老前端请求 `/api/user/` 会 404（Spring 6 中 `useTrailingSlashMatch` 已废弃，需自己加一条映射或规范 URL）。降级方式：`spring.mvc.pathmatch.matching-strategy=ant_path_matcher`（不推荐长期使用）。
**Q2：RESTful 里 PUT 和 PATCH 怎么选？** A：PUT 表示全量替换（未传字段按 null/默认处理，幂等）；PATCH 表示局部更新（只改传入字段）。实践中很多团队用 POST + 明确的动作端点（`/orders/{id}/cancel`）表达"业务动作"，因为严格的资源语义难以覆盖流程操作——面试时说明取舍即可。
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

**编码与乱码三处源头**：① 请求体编码——`server.servlet.encoding.charset=UTF-8`、`force=true`（Boot 已默认 UTF-8，Spring 6 中 `force` 相关配置的默认值以官方文档为准，遇乱码显式设置最稳妥）；② URL 编码——Tomcat 8 起 `URIEncoding` 默认 UTF-8，老版本/老容器需显式设置；③ 响应与页面编码——`Content-Type: application/json;charset=UTF-8`、HTML 的 `<meta charset>`；此外数据库连接需 `useUnicode=true&characterEncoding=utf8`（MySQL 8 推荐 `utf8mb4`）。
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
**JSR-303/Bean Validation 常用注解**：`@NotNull`、`@NotBlank`（字符串）、`@NotEmpty`（集合/字符串）、`@Size`、`@Min/@Max`、`@DecimalMin/@DecimalMax`、`@Pattern`、`@Email`、`@Past/@Future`、`@Positive`、`@Digits`；嵌套对象需在字段上加 `@Valid` 才会级联校验；集合元素校验需 `List<@Valid Item>` 或字段参数级 `@Valid`。

| 维度 | @Valid | @Validated |
| --- | --- | --- |
| 来源 | JSR-303（jakarta.validation） | Spring（org.springframework.validation.annotation） |
| 分组校验 | 不支持 | **支持 groups** |
| 位置 | 方法参数、字段、方法（级联） | 类、方法参数、字段 |
| 方法级校验 | 不触发（需 @Validated 在类上 + AOP） | 支持（@RequestParam 校验需类上加 @Validated） |
| 常用场景 | 嵌套/级联校验 | 分组 + 方法参数校验 |

**全局异常处理**：`@RestControllerAdvice` + `@ExceptionHandler` 拦截指定异常并返回统一响应体；继承 `ResponseEntityExceptionHandler` 可接管 Spring MVC 内置异常（参数绑定失败、405、415、404 等，Spring 6 中方法签名以官方文档为准）；Spring 6 支持 `ProblemDetail`（RFC 7807）。
### 4.2 面试问答
**Q1：@Valid 和 @Validated 的区别？** A：@Valid 是 JSR-303 标准注解，用于触发级联/嵌套校验；@Validated 是 Spring 的扩展，**支持分组校验**，并且加在类上时能对方法参数（含 @RequestParam）做方法级校验。实践：DTO 上写 @Valid（或 @Validated 指定分组），Controller 类上写 @Validated 让查询参数校验生效。
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
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResult<Void>> invalid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ":" + f.getDefaultMessage()).collect(Collectors.joining("; "));
        return ResponseEntity.badRequest().body(ApiResult.fail("PARAM_INVALID", msg));
    }
}
```
### 4.5 一句话速记
@Valid 管嵌套级联、@Validated 管分组与方法级校验；统一异常用 @RestControllerAdvice + ResponseEntityExceptionHandler。

## 5. 拦截器 vs 过滤器 vs AOP
### 5.1 核心原理

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
    @Override public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**").excludePathPatterns("/api/login", "/actuator/**");
    }
}
```
### 5.5 一句话速记
Filter 在容器层（能改流）、Interceptor 在 MVC 层（能拿 HandlerMethod）、AOP 在 Bean 层（与 Web 无关的横切）。

## 6. 跨域 CORS
### 6.1 核心原理
**同源策略**：协议 + 域名 + 端口三者一致才同源；`http://a.com` 与 `https://a.com`、`http://a.com:8080` 都不同源。同源策略限制的是**读取响应**（跨域请求本身可能已发出），所以会出现"服务端已处理、前端拿不到结果"的现象。

**简单请求**：方法为 GET/POST/HEAD 且 Content-Type 仅限 `application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`，且无自定义头 → 浏览器直接发送。**预检请求**：不满足简单请求条件时，浏览器先发 `OPTIONS`，带 `Access-Control-Request-Method`/`Access-Control-Request-Headers`，服务端需返回 `Access-Control-Allow-Origin`、`Allow-Methods`、`Allow-Headers`、`Max-Age`，通过后才发真实请求。

| 方案 | 写法 | 适用场景 |
| --- | --- | --- |
| 方法/类级注解 | `@CrossOrigin(origins = "https://a.com", allowCredentials = "true")` | 少量接口 |
| 全局配置 | `WebMvcConfigurer#addCorsMappings` 或 `CorsFilter` Bean | 统一跨域策略（推荐） |
| 网关/Nginx | 反向代理 + `add_header Access-Control-*` | 生产常用，业务零改动 |

**带 Cookie 的跨域**：`Access-Control-Allow-Origin` **不能是 `*`**，必须回显具体 Origin，同时 `Access-Control-Allow-Credentials: true`，前端 `xhr.withCredentials = true`；另外 Cookie 需要 `SameSite=None; Secure`（HTTPS）才能跨站发送（Chrome 80+ 默认 SameSite=Lax）。
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
**上传**：Boot 自动配置 `MultipartResolver`（实际是 `StandardServletMultipartResolver` + Servlet 3.0 `Part` API），配置项 `spring.servlet.multipart.max-file-size`（单文件）与 `max-request-size`（整请求）默认各为 1MB/10MB，生产必须按业务调大。参数用 `MultipartFile` 接收，注意 `transferTo` 与临时文件清理、文件名安全（防目录穿越：`../`）、类型白名单、大小与数量限制、存储到对象存储而非本地磁盘（多实例部署）。

**下载**：用 `ResponseEntity<Resource>` 或 `StreamingResponseBody`，设置 `Content-Disposition: attachment; filename*=UTF-8''...`（中文名需 URL 编码），大文件要支持 Range（`ResourceHttpRequestHandler` 与 `ResourceRegion` 支持断点续传），并注意流式写出避免全量读入内存。

**静态资源映射**：Boot 默认映射 `classpath:/static/`、`/public/`、`/resources/`、`/META-INF/resources/` 到 `/**`；可用 `spring.web.resources.static-locations`（Boot 2.4+ 的键名，2.4 之前是 `spring.resources.static-locations`）与 `WebMvcConfigurer#addResourceHandlers` 自定义，并可配置 `spring.web.resources.cache.cachecontrol.max-age` 做缓存与版本化。
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

| 维度 | Cookie | Session | Token（随机串） | JWT |
| --- | --- | --- | --- | --- |
| 存储位置 | 浏览器 | 服务端（ID 在 Cookie） | 客户端 | 客户端（自包含） |
| 服务端状态 | 无 | 有 | 有（可存 Redis） | 无状态 |
| 横向扩展 | — | 需共享（Session 集中存储） | 需共享 Redis | 天然支持 |
| 主动失效 | — | 支持 | 支持（删 Redis key） | **难点**（需黑名单/短过期） |
| 体积 | 小 | 小（仅 ID） | 小 | 大（含声明） |
| 安全风险 | CSRF | 会话劫持 | 泄露即失效 | 泄露后无法撤销、payload 可解码 |

**JWT 结构**：`Header.Payload.Signature`，Base64URL 编码；Header 声明算法（HS256 对称 / RS256 非对称），Payload 放标准声明（iss、sub、exp、iat、jti）与自定义声明，Signature 用密钥对前两段签名。**注意 payload 只是编码不是加密**，不能放敏感信息。

**续期方案**：① 双 Token——短效 access token + 长效 refresh token（refresh 只用于换新，可撤销）；② 滑动过期——Redis 记录 token→过期时间，每次请求刷新 TTL（本质变成有状态）；③ 网关续签——统一在网关校验并下发新 token。

**分布式 Session 三种方案**：① 会话粘滞（Nginx ip_hash，扩容/迁移时丢会话，不推荐）；② 会话复制（Tomcat 集群广播，网络开销大，仅小集群）；③ **集中存储**（Spring Session + Redis，主流方案；Boot 3 需对应 Spring Session 版本，`spring-session-data-redis`）。**SSO/OAuth2 简述**：SSO 让一次登录访问多个系统（CAS 是典型协议，核心是中心认证 + 票据校验）；OAuth2 是**授权**框架（授权码、隐式、密码、客户端凭证四种模式，OAuth 2.1 已移除隐式与密码模式），OIDC 在其上补了身份认证（id_token）。企业内做登录+授权通常用 "OAuth2/OIDC + 网关统一鉴权"。
### 8.2 面试问答
**Q1：Session 和 JWT 怎么选？** A：看是否需要"主动踢人"和横向扩展方式。Session + Redis：可主动失效、实现简单、有状态；JWT：无状态、适合多服务/跨域/移动端，但无法即时撤销、体积大、需要处理续期与登出黑名单。混合常见做法：JWT 承载身份 + Redis 存会话/黑名单以换取撤销能力。
**Q2：JWT 登出怎么做？** A：JWT 本身无状态，登出只能：① 客户端删除 token（防不住已泄露的 token）；② 服务端维护黑名单（Redis 存 jti 到过期，代价是有状态）；③ 用短 access + 可撤销 refresh 的组合，把"撤销"放在 refresh 层。
**Q3：Cookie 的安全属性有哪些？** A：`HttpOnly`（JS 不可读，防 XSS 窃取）、`Secure`（仅 HTTPS 发送）、`SameSite`（Lax/Strict/None 防 CSRF）、`Domain/Path`（作用域）、`Max-Age/Expires`（有效期）。**重要凭证建议只放 HttpOnly Cookie，而不是 localStorage**。
**Q4：CSRF 和 XSS 与这套机制的关系？** A：XSS 是注入脚本窃取凭证（防御：转义/Content-Security-Policy、HttpOnly）；CSRF 是借用浏览器自动携带 Cookie 发起请求（防御：SameSite、CSRF Token、校验 Origin/Referer）。使用 Authorization header 传 Token 的方式天然规避 CSRF（但要注意 XSS 风险更高）。
**Q5：Spring Session + Redis 的原理？** A：用 `SessionRepositoryFilter` 包装 request/response，把 `HttpSession` 实现替换为 Redis 支持的实现（key 含 sessionId，写入时序列化），从而多实例共享；配合 `spring.session.redis.namespace` 隔离环境。注意序列化方式（JDK 序列化 vs JSON）会影响兼容性与安全。
### 8.3 高频追问与陷阱
- 陷阱：Session 粘滞方案在扩容/发布时导致用户被踢出，是"发布后大批 401"的常见原因。
- 陷阱：JWT 用对称密钥在多服务场景下需要共享密钥，泄露风险高；推荐 RS256 由认证中心签名、业务服务只持公钥验签。
### 8.4 代码/配置示例
```java
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
Session 有状态可撤销、JWT 无状态难撤销；分布式 Session 选 Spring Session + Redis；OAuth2 是授权、OIDC 才是认证。

## 9. Servlet 容器：Tomcat 架构与调优
### 9.1 核心原理
**Tomcat 整体架构**：`Server`（整个容器）→ `Service`（把 Connector 与 Engine 绑定）→ 一侧是 `Connector`（负责网络协议与 IO，内部由 `ProtocolHandler` + `Endpoint` + `Processor` 组成），另一侧是 `Engine` → `Host`（虚拟主机）→ `Context`（一个 Web 应用）→ `Wrapper`（一个 Servlet）。

**Connector 与 IO 模型**：`Http11NioProtocol`（NIO，Boot 默认）、`Http11Nio2Protocol`（NIO2/AIO 风格，Linux 上优势不明显）、APR（基于本地库，性能好但需装依赖，**在较新版本中已不推荐甚至移除，以官方文档为准**）。

| 参数（Tomcat 名 / Boot 属性） | 含义 | 说明 |
| --- | --- | --- |
| maxThreads / server.tomcat.threads.max | 处理请求的最大工作线程数 | Tomcat 默认 200；按 QPS×RT 估算，结合压测调整 |
| minSpareThreads / threads.min-spare | 最小空闲线程 | Tomcat 默认 10 |
| acceptCount / server.tomcat.accept-count | 等待队列长度（全连接队列塞满后） | Tomcat 默认 100；队列满后新连接被拒/超时 |
| maxConnections / server.tomcat.max-connections | 同时可接受并处理的连接数 | NIO 下 Tomcat 默认 8192 |
| connectionTimeout / server.tomcat.connection-timeout | 等待请求行/头的超时 | Tomcat 默认 60s（Boot 属性未设时沿用容器默认，以官方文档为准） |
| server.tomcat.max-http-form-post-size | POST 表单大小上限 | 默认 2MB |
| threads.max-queue-capacity / accept-count | 请求排队策略 | 队列无界会掩盖问题，需与超时配套 |

**Tomcat 与 Undertow 对比**：Undertow 基于 XNIO，轻量、内存占用低、内置对 HTTP/2 与 WebSocket 的良好支持，阻塞式 Servlet 场景与 Tomcat 差距不大；Tomcat 生态与文档最成熟、排障资料最多、与各类框架贴合最好。**结论：默认用 Tomcat；资源敏感或需要更轻量运行时可选 Undertow/Jetty，但要以压测数据说话。**
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

| 方式 | 返回类型 | 谁写结果 | 适用场景 |
| --- | --- | --- | --- |
| Callable | `Callable<T>` | Spring MVC 用 TaskExecutor 执行后自动回写 | 简单异步（把耗时逻辑交给线程池） |
| DeferredResult | `DeferredResult<T>` | **业务代码在其他线程调用 setResult** | 长轮询、与其他系统回调/事件联动 |
| WebAsyncTask | `WebAsyncTask<T>` | Spring 管理，可设超时与回调 | 需要超时与完成回调的异步 |
| ResponseBodyEmitter / SseEmitter | `SseEmitter` | 多次 `send()` | 服务端推送（SSE，单向、基于 HTTP） |
| StreamingResponseBody | `void` + 写 OutputStream | 直接写响应流 | 大文件/流式导出 |
| WebSocket | `WebSocketHandler` / STOMP | 双向 | 双向实时通信（IM、协同） |

关键机制：Servlet 3.0 异步（`request.startAsync()`）让容器线程在等待期间被释放；异步超时后 Spring 会重新 dispatch 并可用 `DeferredResult` 的 `onTimeout` 处理；**异步请求需要 Filter/Servlet 支持 async（`asyncSupported`）**，否则会报异常；默认异步超时由 `spring.mvc.async.request-timeout` 控制（以官方文档为准）。
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

| 风险 | 攻击方式 | 防御手段 |
| --- | --- | --- |
| 越权 | 改 URL/参数里的 id 访问他人数据（水平）、访问高权限接口（垂直） | 服务端从会话取用户身份、数据权限过滤（SQL 层加 owner 条件）、URL 不可直接映射资源归属、接口级权限注解 |
| 重放 | 抓包后重复提交同一请求 | 时间戳 + nonce + 签名（HMAC），nonce 在 Redis 去重并设置窗口 |
| 参数篡改 | 篡改金额、数量等 | 签名校验；金额等关键值服务端重算 |
| 重复提交/幂等 | 网络重试、用户连点 | 唯一索引、幂等 token（先删后用）、状态机 + 乐观锁、分布式锁（见[分布式锁与分布式ID](../05-分布式与架构/03-分布式锁与分布式ID.md)） |
| 限流 | 恶意刷接口/爬虫 | 网关或应用层限流（令牌桶/漏桶/滑动窗口），见 [Nginx 与网关限流](../04-中间件/05-Nginx与网关限流.md) |
| 敏感数据泄露 | 日志/响应体带手机号、身份证 | 脱敏（响应序列化时打码）、日志脱敏、最小化返回字段 |

**签名设计要点**：参数按字典序拼接 + 时间戳 + 随机串 + 密钥 → HMAC-SHA256；服务端校验时间窗口（如 ±5 分钟）与 nonce 唯一性；密钥不落前端（App 场景可做签名混淆但不绝对安全）。**限流常用算法**：固定窗口（简单但有临界问题）、滑动窗口（精度与成本折中，Sentinel 用）、漏桶（恒定速率）、令牌桶（允许突发，Guava RateLimiter/Redis+Lua）。
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
@PostMapping("/pay")
public ApiResult pay(@RequestBody @Valid PayCmd cmd, @RequestHeader("Idempotency-Key") String key) {
    // SET NX：只有第一次能拿到，天然幂等；过期时间覆盖业务最长执行时间
    Boolean first = redis.opsForValue().setIfAbsent("idem:pay:" + key, "1", Duration.ofMinutes(10));
    if (Boolean.FALSE.equals(first)) throw new BizException("请求处理中，请勿重复提交");
    try { return payService.pay(cmd); }
    catch (Exception e) { redis.delete("idem:pay:" + key); throw e; }   // 失败释放以便重试
}
```
### 11.5 一句话速记
越权永不信客户端身份（SQL 层过滤）、幂等靠唯一索引或 Redis SET NX、防重放靠时间戳+nonce+签名、限流分层做。

## 自测题（含答案要点）
1. **DispatcherServlet 九大组件？一次请求的处理顺序？** HandlerMapping、HandlerAdapter、HandlerExceptionResolver、ViewResolver、RequestToViewNameTranslator、LocaleResolver、ThemeResolver、MultipartResolver、FlashMapManager；顺序为 HandlerMapping → HandlerAdapter → preHandle → 参数解析 → Controller → 返回值处理 → postHandle → 渲染 → afterCompletion。
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
