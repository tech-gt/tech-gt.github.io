---
title: "Netty 网关问题的排查与修复记录"
date: 2019-08-12T10:00:00+08:00
categories: ["Netty", "网关"]
tags: ["Netty", "网关"]
---


## 前言

在压测期间遭遇了从 `IllegalReferenceCountException` 到各类 502 错误的。

## 一 `IllegalReferenceCountException`

始于一次常规的性能压测。当并发量提升到一定阈值时，控制台开始零星地喷出 `io.netty.util.IllegalReferenceCountException: refCnt: 0` 异常。

```
io.netty.util.IllegalReferenceCountException: refCnt: 0
        at io.netty.buffer.AbstractByteBuf.ensureAccessible(AbstractByteBuf.java:1454)
        at io.netty.buffer.AbstractByteBuf.copy(AbstractByteBuf.java:1194)
        at com.gateway.service.AsyncHttpForwardService.createBackendRequest(AsyncHttpForwardService.java:224)
        at com.gateway.service.AsyncHttpForwardService$2.operationComplete(AsyncHttpForwardService.java:168)
        ...
```

这个异常指向 `originalRequest.content().copy()`，明示我们试图操作一个已经被释放的 `ByteBuf`。

### 问题分析：竞态条件

Netty 的 `ByteBuf` 使用引用计数来管理内存。当我们尝试 `retain()` 一个已经被 `release()` 的 `ByteBuf` 时，就会触发此异常。

经过对代码逻辑的仔细梳理，我们定位到了一个典型的**异步竞态条件 (Race Condition)**：

1.  **获取连接**: 从连接池获取一个到后端服务的 `Channel`。
2.  **添加处理器**: **立即**将 `BackendResponseHandler` 添加到该 `Channel` 的 `pipeline` 中。
3.  **发送请求**: 异步调用 `channel.writeAndFlush(request)`。

问题就出在第 2 步和第 3 步之间：

*   **失败路径**: 如果 `writeAndFlush` 操作**因为网络原因瞬间失败**（例如，连接刚建立就被对端重置），它的 `FutureListener` 会被触发。
*   **连锁反应**: 失败的 `Listener` 会调用 `channel.close()`，这会触发 `pipeline` 上的 `channelInactive` 事件。
*   **提前释放**: `BackendResponseHandler` 捕获到 `channelInactive` 事件后，认为请求已经结束，便调用 `originalRequest.release()`，将请求的引用计数减为 0。
*   **致命一击**: 与此同时，`writeAndFlush` 失败的 `Listener` 在关闭连接后，还会继续执行**重试逻辑**，当它再次尝试使用那个已被释放的 `request` 对象去 `copy()` 内容时，异常爆发。

### 解决方案

我们必须保证只有在请求成功发送后，才将响应处理器加入 `pipeline`。

```java
// ...
backendChannel.writeAndFlush(backendRequest).addListener(writeFuture -> {
    if (writeFuture.isSuccess()) {
        // 请求发送成功后，再添加响应处理器
        backendChannel.pipeline().addLast("responseHandler",
                new BackendResponseHandler(...));
    } else {
        // 发送失败，销毁连接并重试
        // ...
    }
});
```

如果发送失败，`BackendResponseHandler` 根本没有机会被添加，自然也不会有提前释放资源的问题。

## 502问题

解决了引用计数问题后，压测过程平稳了许多。但新的问题随之而来，日志中开始出现 502 错误：`{"error": "Bad Gateway", "message": "Backend connection closed unexpectedly", ...}`。

### 问题分析：连接池的陷阱

这个问题指向了一个在长连接（Keep-Alive）和连接池模式下非常经典的陷阱——**陈旧连接（Stale Connection）**。

*   **Keep-Alive 机制**: 为了性能，网关与后端服务之间会复用 TCP 连接。
*   **服务器超时**: 后端服务（如 Tomcat）有自己的 Keep-Alive 超时设置。如果一个连接在网关的连接池里闲置过久，后端会单方面关闭它。
*   **“假活”状态**: 在网关侧，这个已被后端关闭的连接在被取用之前，可能仍然被认为是 `active` 的。
*   **失败的请求**: 当网关用这个“已死”的连接发送数据时，会立即触发 `channelInactive` 事件，而我们当时的处理器逻辑只是简单地向客户端返回 502，并未尝试挽救。

### 解决方案：为“意外断开”赋予重试能力

问题的核心在于，`channelInactive` 应当被视为一种**可重试的瞬态网络错误**。

为了实现这一点，我们进行了一次重构：

1.  **提取统一的重试方法**: 我们创建了一个 `handleConnectionFailure(...)` 方法，它封装了所有重试逻辑（检查次数、延迟执行等）。
2.  **让 Handler 变得更“聪明”**: `BackendResponseHandler` 不再是一个简单的响应转换器。我们让它持有重试所需的上下文信息，如 `BackendService` 和剩余重试次数 `retriesLeft`。
3.  **改造 `channelInactive`**: 修改 `channelInactive` 的实现，让它在捕获到连接意外关闭时，调用 `handleConnectionFailure` 来触发重试，而不是直接报错。

```java
// 在 BackendResponseHandler 中
@Override
public void channelInactive(ChannelHandlerContext ctx) throws Exception {
    if (!processed) {
        processed = true;
        // 后端连接意外关闭，这是一个典型的可重试场景 (stale connection)
        if (gatewayCtx.channel().isActive()) {
            // 调用统一的失败处理/重试逻辑
            handleConnectionFailure(gatewayCtx, originalRequest, service, retriesLeft, "Backend connection closed unexpectedly, retrying...", null);
        } else {
            originalRequest.release();
        }
    }
}
```

##超时与重置

网关的健壮性已大大提升，但压测日志中仍有概率出现：`ReadTimeoutException` 和 `IOException: Connection reset by peer`。

*   `ReadTimeoutException`: 网关在指定时间内未收到后端响应。
*   `IOException`: 后端服务可能崩溃或重启，强行中止了连接。

这两种同样是典型的瞬态网络错误。我们现有的 `exceptionCaught` 处理器只会记录日志并返回 502，它需要变得更智能。

### 解决方案：精准识别，分类处理

我们不能对所有异常都进行重试（例如 `NullPointerException`），这可能会掩盖真正的程序 Bug。因此，我们需要对异常进行**分类处理**。

1.  **创建 `isRetryableException` 方法**: 新增一个辅助方法，专门用于判断一个 `Throwable` 是否属于可重试的网络异常。

    ```java
    private boolean isRetryableException(Throwable cause) {
        // 读超时，意味着后端可能繁忙，重试可能成功
        if (cause instanceof io.netty.handler.timeout.ReadTimeoutException) {
            return true;
        }
        // 连接被重置等IO异常，是典型的网络问题
        if (cause instanceof java.io.IOException) {
            return true;
        }
        return false;
    }
    ```

2.  **升级 `exceptionCaught` 方法**:
    *   首先，任何导致 `exceptionCaught` 的 `Channel` 都被认为是有问题的，必须**强制关闭**，绝不放回连接池。
    *   然后，调用 `isRetryableException` 判断异常类型。
    *   如果是可重试的，就调用 `handleConnectionFailure`。
    *   如果不是，则快速失败，记录严重错误并返回 502。

```java
@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
    // 抛出异常的 Channel 必须关闭
    cleanupAndRelease(ctx, true);

    if (processed) { return; }
    processed = true;
    
    // 如果是可重试的瞬态网络错误，则尝试重试
    if (isRetryableException(cause)) {
        handleConnectionFailure(gatewayCtx, originalRequest, service, retriesLeft, "Backend connection error, retrying...", cause);
    } else {
        // 对于其他未知错误，快速失败
        logger.error("Non-retryable backend error: ", cause);
        sendErrorResponse(gatewayCtx, HttpResponseStatus.BAD_GATEWAY, "Backend processing error: " + cause.getMessage());
        originalRequest.release();
    }
}
```

## 总结与心得

1.  **异步编程，时序为王**: 在 Netty 的异步世界里，操作的执行顺序至关重要。错误的顺序是许多并发 Bug 的根源。
2.  **重试是服务可靠性的基石**: 构建任何网络服务，都必须有一套完善的重试机制。关键在于识别哪些错误是“值得”重试的。
3.  **连接池不是银弹**: 连接池带来了性能，也带来了“陈旧连接”等管理复杂性。必须主动处理这些问题，而不是期望它能“自动工作”。
4.  **异常处理必须精细化**: 不要用一个 `catch(Exception e)` 来处理所有问题。对异常进行分类，区分瞬态网络错误和永久性应用错误，是构建健壮系统的关键。

