---
title: "Java异常性能优化：使用Supplier和覆盖fillInStackTrace降低CPU占用"
date: 2019-09-12T16:45:00+08:00
categories:
  - Java
  - 性能优化
  - 高并发
  - 异常
tags:
  - 异常处理
  - 性能调优
  - 高并发
  - 异常
description: "深入分析Java异常创建的性能瓶颈，通过覆盖fillInStackTrace和使用Supplier模式实现异常性能优化。"
---

在高并发的Java应用中，异常处理往往是一个被忽视的性能瓶颈。最近在优化一个API网关项目时，发现大量的用户认证异常创建导致CPU占用过高，通过profiling工具分析后发现，异常对象的创建成本远比我们想象的要高。

## 异常创建的性能瓶颈

### 问题现象

在我们的网关系统中，有这样一段代码：

```java
public class AuthenticationFilter {
    public void doFilter(HttpServletRequest request, HttpServletResponse response) {
        String token = request.getHeader("Authorization");
        if (StringUtils.isEmpty(token)) {
            throw new UnauthorizedException("缺少认证token");
        }
        if (!tokenValidator.isValid(token)) {
            throw new UnauthorizedException("token已过期或无效");
        }
        // 继续处理请求...
    }
}
```

在高峰期，这个认证过滤器每秒要处理数万次请求，其中约15%的请求会因为token问题触发认证异常。通过JProfiler分析发现，`fillInStackTrace`方法占用了大量CPU时间。

### 根本原因分析

当我们创建一个异常对象时，JVM会自动调用`fillInStackTrace()`方法来收集当前线程的调用栈信息。这个过程包括：

1. **遍历调用栈**：从当前方法开始，逐层向上遍历整个调用链
2. **收集栈帧信息**：记录每个方法的类名、方法名、文件名、行号等
3. **创建StackTraceElement数组**：将收集到的信息封装成对象数组

这个过程在深层调用栈的情况下会变得非常耗时。

## 优化方案一：覆盖fillInStackTrace禁用堆栈跟踪

### 实现思路

既然填充堆栈是性能瓶颈，最直接的优化方法就是阻止这个行为。我们可以创建一个自定义的"快速"异常类：

```java
public class FastException extends RuntimeException {
    public FastException(String message) {
        super(message);
    }
    
    public FastException(String message, Throwable cause) {
        super(message, cause);
    }
    
    @Override
    public Throwable fillInStackTrace() {
        // 直接返回this，不填充堆栈信息
        return this;
    }
}
```

### 业务异常类的改造

基于FastException，我们可以创建具体的业务异常：

```java
public class UnauthorizedException extends FastException {
    private final String errorCode;
    
    public UnauthorizedException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public String getErrorCode() {
        return errorCode;
    }
}

public class AuthenticationFilter {
    public void doFilter(HttpServletRequest request, HttpServletResponse response) {
        String token = request.getHeader("Authorization");
        if (StringUtils.isEmpty(token)) {
            throw new UnauthorizedException("MISSING_TOKEN", "缺少认证token");
        }
        if (!tokenValidator.isValid(token)) {
            throw new UnauthorizedException("INVALID_TOKEN", "token已过期或无效");
        }
    }
}
```

### 性能测试对比

我们做了一个简单的性能测试：

```java
public class ExceptionPerformanceTest {
    private static final int ITERATIONS = 1_000_000;
    
    @Test
    public void testNormalException() {
        long start = System.currentTimeMillis();
        for (int i = 0; i < ITERATIONS; i++) {
            try {
                throw new RuntimeException("测试异常");
            } catch (Exception e) {
                // 捕获并忽略
            }
        }
        long end = System.currentTimeMillis();
        System.out.println("普通异常耗时: " + (end - start) + "ms");
    }
    
    @Test
    public void testFastException() {
        long start = System.currentTimeMillis();
        for (int i = 0; i < ITERATIONS; i++) {
            try {
                throw new FastException("测试异常");
            } catch (Exception e) {
                // 捕获并忽略
            }
        }
        long end = System.currentTimeMillis();
        System.out.println("快速异常耗时: " + (end - start) + "ms");
    }
}
```

测试结果显示，FastException的性能提升了约80%。

## 优化方案二：使用Supplier进行延迟创建

### 问题分析

覆盖`fillInStackTrace`解决了"创建时"的成本问题，但如果连创建这个动作本身都可以避免，性能会更好。考虑这样的场景：

```java
public class GatewayFilter {
    public void processRequest(HttpServletRequest request) {
        // 大部分情况下认证都会通过
        validateAuth(request, new UnauthorizedException("AUTH_FAILED", "认证失败"));
        
        // 处理请求逻辑...
    }
    
    private void validateAuth(HttpServletRequest request, RuntimeException exception) {
        String token = request.getHeader("Authorization");
        if (StringUtils.isEmpty(token)) {
            throw exception; // 只有在真正需要时才抛出
        }
    }
}
```

在上面的代码中，即使大部分情况下认证都会通过，我们仍然创建了异常对象，这是不必要的开销。

### Supplier模式的应用

使用`Supplier<T>`可以实现真正的延迟创建：

```java
public class GatewayFilter {
    public void processRequest(HttpServletRequest request) {
        // 传递异常的创建逻辑，而不是异常对象本身
        validateAuth(request, () -> new UnauthorizedException("AUTH_FAILED", "认证失败"));
        
        // 处理请求逻辑...
    }
    
    private void validateAuth(HttpServletRequest request, Supplier<RuntimeException> exceptionSupplier) {
        String token = request.getHeader("Authorization");
        if (StringUtils.isEmpty(token)) {
            throw exceptionSupplier.get(); // 只有在需要时才创建异常
        }
    }
}
```

### 更优雅的工具类封装

我们可以创建一个工具类来简化这种模式的使用：

```java
public class Assertions {
    public static void require(boolean condition, Supplier<RuntimeException> exceptionSupplier) {
        if (!condition) {
            throw exceptionSupplier.get();
        }
    }
    
    public static void requireNonNull(Object obj, Supplier<RuntimeException> exceptionSupplier) {
        if (obj == null) {
            throw exceptionSupplier.get();
        }
    }
    
    public static void requireNonEmpty(String str, Supplier<RuntimeException> exceptionSupplier) {
        if (StringUtils.isEmpty(str)) {
            throw exceptionSupplier.get();
        }
    }
}
```

使用示例：

```java
public class AuthenticationFilter {
    public void validateAuth(HttpServletRequest request) {
        String token = request.getHeader("Authorization");
        
        Assertions.requireNonEmpty(token, 
            () -> new UnauthorizedException("MISSING_TOKEN", "缺少认证token"));
            
        Assertions.require(tokenValidator.isValid(token), 
            () -> new UnauthorizedException("INVALID_TOKEN", "token已过期或无效"));
            
        Assertions.require(hasPermission(token, request.getRequestURI()), 
            () -> new UnauthorizedException("NO_PERMISSION", "无权限访问该资源"));
    }
}
```

## 实际应用中的注意事项

### 1. 调试困难

使用FastException后，异常不再包含堆栈信息，这会给调试带来困难。建议：

```java
public class FastException extends RuntimeException {
    private static final boolean ENABLE_STACK_TRACE = 
        Boolean.parseBoolean(System.getProperty("fastexception.stacktrace", "false"));
    
    @Override
    public Throwable fillInStackTrace() {
        return ENABLE_STACK_TRACE ? super.fillInStackTrace() : this;
    }
}
```

通过JVM参数`-Dfastexception.stacktrace=true`可以在需要时开启堆栈跟踪。

### 2. 异常信息的完整性

在禁用堆栈跟踪后，要确保异常消息足够详细，包含必要的上下文信息：

```java
public void validateAuth(HttpServletRequest request) {
    String token = request.getHeader("Authorization");
    if (StringUtils.isEmpty(token)) {
        throw new UnauthorizedException("MISSING_TOKEN", 
            String.format("缺少认证token: uri=%s, method=%s, clientIp=%s", 
                request.getRequestURI(), request.getMethod(), getClientIp(request)));
    }
}
```

### 3. 监控和日志

建议在异常处理的地方添加详细的日志记录：

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorized(UnauthorizedException e) {
        // 记录详细的错误信息，包括当前线程的堆栈
        logger.warn("用户认证失败: errorCode={}, message={}, stackTrace={}", 
            e.getErrorCode(), e.getMessage(), getStackTrace());
            
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(e.getErrorCode(), e.getMessage()));
    }
    
    private String getStackTrace() {
        return Arrays.stream(Thread.currentThread().getStackTrace())
            .map(StackTraceElement::toString)
            .collect(Collectors.joining("\n"));
    }
}
```


{{< mermaid >}}
graph TD
    A[请求到达] --> B{参数验证}
    B -->|验证通过| C[业务处理]
    B -->|验证失败| D[Supplier.get]
    D --> E[FastException创建]
    E --> F[异常抛出]
    C --> G[返回结果]
    F --> H[异常处理]
    H --> I[错误响应]
{{< /mermaid >}}

## 总结

异常处理的性能优化往往被开发者忽视，但在高并发场景下，这种优化能带来显著的性能提升。通过覆盖`fillInStackTrace`和使用`Supplier`模式，我们可以在保持代码可读性的同时，大幅降低异常处理的CPU开销。