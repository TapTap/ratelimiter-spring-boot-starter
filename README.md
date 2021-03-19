# ratelimiter-spring-boot-starter
基于 redis 的偏业务应用的分布式限流组件，使得项目拥有分布式限流能力变得很简单。限流的场景有很多，常说的限流一般指网关限流，控制好洪峰流量，以免打垮后方应用。这里突出`偏业务应用的分布式限流`的原因，是因为区别于网关限流，业务侧限流可以轻松根据业务性质做到细粒度的流量控制。比如如下场景，

- 案例一：

有一个公开的 openApi 接口， openApi 会给接入方派发一个 appId，此时，如果需要根据各个接入方的 appId 限流，网关限流就不好做了，只能在业务侧实现

- 案例二：

公司内部的短信接口，内部对接了多个第三方的短信通道，每个短信通道对流量的控制都不尽相同，假设有的第三方根据手机号和短信模板组合限流，网关限流就更不好做了

以上举例的场景，通过 ratelimiter-spring-boot-starter 可以轻松解决限流问题

## 1、快速开始

### 1.1、添加组件依赖，已上传到maven中央仓库
maven
```xml
<dependency>
    <groupId>com.github.taptap</groupId>
    <artifactId>ratelimiter-spring-boot-starter</artifactId>
    <version>1.1.1</version>
</dependency>

```
gradle
```groovy
implementation 'com.github.taptap:ratelimiter-spring-boot-starter:1.1.1'
```

### 1.2、application.properties 配置
```properties
spring.ratelimiter.enabled = true

spring.ratelimiter.redis-address = redis://127.0.0.1:6379
spring.ratelimiter.redis-password = xxx
```
启用 ratelimiter 的配置必须加，默认不会加载。redis 相关的连接是非必须的，如果你的项目里已经使用了 `Redisson` ，则不用配置限流框架的 redis 连接
### 1.3、在需要加限流逻辑的方法上，添加注解 @RateLimit，如：
```java
@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/get")
    @RateLimit(rate = 5, rateInterval = "10s")
    public String get(String name) {
        return "hello";
    }
}
```

#### 1.3.1 @RateLimit 注解说明
@RateLimit 注解可以添加到任意被 spring 管理的 bean 上，不局限于 controller ,service 、repository 也可以。在最基础限流功能使用上，以上三个步骤就已经完成了。@RateLimit 有两个最基础的参数，rateInterval 设置了时间窗口，rate 设置了时间窗口内允许通过的请求数量
#### 1.3.2 限流的粒度，限流 key
。限流的粒度是通过限流的 key 来做的，在最基础的设置下，限流的 key 默认是通过方法名称拼出来的，规则如下：
```properties
key = RateLimiter_ + 类名 + 方法名
```
除了默认的 key 策略，ratelimiter-spring-boot-starter 充分考虑了业务限流时的复杂性，提供了多种方式。结合业务特征，达到更细粒度的限流控制。
#### 1.3.3 触发限流后的行为
默认触发限流后 程序会返回一个 http 状态码为 429 的响应，响应值如下：
```json
{
  "code":429,
  "msg":"Too Many Requests"
}
```
同时，响应的 header 里会携带一个 Retry-After 的时间值，单位 s，用来告诉调用方多久后可以重试。当然这一切都是可以自定义的，进阶用法可以继续往下看

## 2、进阶用法
### 2.1、自定义限流的 key
自定义限流 key 有三种方式，当自定义限流的 key 生效时，限流的 key 就变成了（默认的 key + 自定义的 key）。下面依次给出示例

#### 2.1.1、@RateLimitKey 的方式
```java
@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/get")
    @RateLimit(rate = 5, rateInterval = "10s")
    public String get(@RateLimitKey String name) {
        return "get";
    }
}
```
@RateLimitKey 注解可以放在方法的入参上，要求入参是基础数据类型，上面的例子，如果 name = kl。那么最终限流的 key 如下：
```properties
key = RateLimiter_com.taptap.ratelimiter.web.TestController.get-kl
```

#### 2.1.2、指定 keys 的方式
```java
@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/get")
    @RateLimit(rate = 5, rateInterval = "10s",keys = {"#name"})
    public String get(String name) {
        return "get";
    }

    @GetMapping("/hello")
    @RateLimit(rate = 5, rateInterval = "10s",keys = {"#user.name","user.id"})
    public String hello(User user) {
        return "hello";
    }
}
```
keys 这个参数比 @RateLimitKey 注解更智能，基本可以包含 @RateLimitKey 的能力，只是简单场景下，使用起来没有 @RateLimitKey 那么便捷。keys 的语法来自 spring 的 `Spel`，可以获取对象入参里的属性，支持获取多个，最后会拼接起来。使用过 spring-cache 的同学可能会更加熟悉 如果不清楚 `Spel` 的用法，可以参考 spring-cache 的注解文档

#### 2.1.3、自定义 key 获取函数
```java
@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/get")
    @RateLimit(rate = 5, rateInterval = "10s",customKeyFunction = "keyFunction")
    public String get(String name) {
        return "get";
    }

    public String keyFunction(String name) {
        return "keyFunction" + name;
    }
}
```
当 @RateLimitKey 和 keys 参数都没法满足时，比如入参的值是一个加密的值，需要解密后根据相关明文内容限流。可以通过在同一类里自定义获取 key 的函数，这个函数要求和被限流的方法入参一致，返回值为 String 类型。返回值不能为空，为空时，会回退到默认的 key 获取策略。 

### 2.2、自定义限流后的行为
#### 2.2.1、配置响应内容
```properties
spring.ratelimiter.enabled=true
spring.ratelimiter.response-body=Too Many Requests
spring.ratelimiter.status-code=509
```
添加如上配置后，触发限流时，http 的状态码就变成了 509 。响应的内容变成了 Too Many Requests 了

#### 2.2.2 自定义限流触发异常处理器
默认的触发限流后，限流器会抛出一个异常，限流器框架内定义了一个异常处理器来处理。自定义限流触发处理器，需要先禁用系统默认的限流触发处理器，禁用方式如下：
```properties
spring.ratelimiter.exceptionHandler.enable=false
```
然后在项目里添加自定义处理器，如下：
```java
@ControllerAdvice
public class RateLimitExceptionHandler {

    private final  RateLimiterProperties limiterProperties;

    public RateLimitExceptionHandler(RateLimiterProperties limiterProperties) {
        this.limiterProperties = limiterProperties;
    }

    @ExceptionHandler(value = RateLimitException.class)
    @ResponseBody
    public String exceptionHandler(HttpServletResponse response, RateLimitException e){
        response.setStatus(limiterProperties.getStatusCode());
        response.setHeader("Retry-After", String.valueOf(e.getRetryAfter()));
        return limiterProperties.getResponseBody();
    }
}
```
#### 2.2.3 自定义触发限流处理函数，限流降级
```java
@RequestMapping("/test")
public class TestController {

    @GetMapping("/get")
    @RateLimit(rate = 5, rateInterval = "10s",fallbackFunction = "getFallback")
    public String get(String name) {
        return "get";
    }

    public String getFallback(String name){
        return "Too Many Requests" + name;
    }

}
```
这种方式实现和使用和 2.1.3、自定义 key 获取函数类似。但是多一个要求，返回值的类型需要和原限流函数的返回值类型一致，当触发限流时，框架会调用 fallbackFunction 配置的函数执行并返回，达到限流降级的效果

## 3、集成示例测验
启动 src/test/java/com/taptap/ratelimiter/Application.java 后，访问 http://localhost:8080/swagger-ui.html 

## 4、版本更新
### 4.1、（v1.1.1）版本更新内容
触发限流时，header 的 Retry-After 值，单位由 ms ，调整成了 s

