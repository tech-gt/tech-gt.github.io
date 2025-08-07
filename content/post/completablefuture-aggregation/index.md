---
title: "用 CompletableFuture 聚合多个后端服务响应"
date: 2022-07-15T10:23:00+08:00
categories: ["Java并发编程", "后端架构"]
tags: ["CompletableFuture", "异步编程", "聚合服务", "性能优化"]
description: "如何用 CompletableFuture 优雅地聚合多个后端服务响应，提升动态中心等高并发场景下的响应速度。"
---

在实际的后端开发中，我们经常会遇到聚合多个服务响应的需求。比如在直播App的“动态中心”页面，需要同时展示用户信息、帖子内容、评论数等数据。这些数据往往来自不同的后端服务，传统的做法是串行调用各个RPC接口，导致整体响应时间受限于最慢的那个服务。

## 传统串行调用的痛点

以往的代码大致如下：

```java
UserInfo userInfo = userService.getUserInfo(userId); // 阻塞
Post post = postService.getPost(postId); // 阻塞
int commentCount = commentService.getCommentCount(postId); // 阻塞

DynamicVO vo = new DynamicVO(userInfo, post, commentCount);
return vo;
```

如果每个RPC耗时100ms，串行下来就要300ms，用户体验很差。

## 用 CompletableFuture 改造为并行异步

Java 8 引入的 `CompletableFuture`，让我们可以非常优雅地将这些阻塞调用改造成并行异步，大幅提升聚合接口的性能。

### 核心思路
- 每个RPC调用用 `CompletableFuture.supplyAsync` 包装，放到线程池里并行执行。
- 用 `thenCombine`、`allOf` 等方法组合多个Future，等所有结果都返回后再组装最终VO。

### 代码示例

```java
ExecutorService executor = Executors.newFixedThreadPool(3);

CompletableFuture<UserInfo> userFuture = CompletableFuture.supplyAsync(() -> userService.getUserInfo(userId), executor);
CompletableFuture<Post> postFuture = CompletableFuture.supplyAsync(() -> postService.getPost(postId), executor);
CompletableFuture<Integer> commentCountFuture = CompletableFuture.supplyAsync(() -> commentService.getCommentCount(postId), executor);

CompletableFuture<DynamicVO> resultFuture = userFuture.thenCombine(postFuture, (user, post) -> new Object[]{user, post})
    .thenCombine(commentCountFuture, (arr, commentCount) -> {
        UserInfo user = (UserInfo) arr[0];
        Post post = (Post) arr[1];
        return new DynamicVO(user, post, commentCount);
    });

DynamicVO vo = resultFuture.get(); // 阻塞等待所有结果
```

这样，整体耗时只取决于最慢的那个RPC，理论上可以缩短到100ms左右。

## 实际工作中的经验

- **线程池要合理配置**，避免线程资源耗尽。
- **异常处理要完善**，可以用 `exceptionally`、`handle` 等方法兜底，避免某个服务挂掉导致整体失败。
- **链式组合更优雅**，复杂场景下可以用 `allOf` 聚合多个Future，再统一处理结果。

### 更复杂的聚合场景

如果聚合的服务更多，可以这样：

```java
CompletableFuture.allOf(userFuture, postFuture, commentCountFuture)
    .thenApply(v -> {
        UserInfo user = userFuture.join();
        Post post = postFuture.join();
        int commentCount = commentCountFuture.join();
        return new DynamicVO(user, post, commentCount);
    });
```

## 流程图示意

{{< mermaid >}}
sequenceDiagram
    participant C as Controller
    participant U as UserService
    participant P as PostService
    participant M as CommentService
    C->>U: getUserInfo(userId)
    C->>P: getPost(postId)
    C->>M: getCommentCount(postId)
    U-->>C: UserInfo
    P-->>C: Post
    M-->>C: CommentCount
    C->>C: 聚合结果
{{< /mermaid >}}

## 总结

在高并发、聚合型接口场景下，合理利用 `CompletableFuture` 进行异步并发编排，可以极大提升接口性能和用户体验。实际项目中，建议优先考虑这种方式替代传统串行阻塞调用。 