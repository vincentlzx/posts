---
title: SSE实时通讯
categories:
- [实时通讯]
tags:
- SSE
- 实时通讯
---



# SSE

在 Spring Boot 中实现实时通信可以使用多种技术和组件。以下是一些常用的实时通信方案：

- WebSocket
- Server-Sent Events（SSE）

SSE（Server-Sent Events）是基于 HTTP 的单向通信机制。它允许服务器通过单个 HTTP 连接将实时事件流推送到客户端。SSE 是一种轻量级的实时通信协议，适合于服务器向客户端发送实时更新的场景。SSE 建立在常规的 HTTP/HTTPS 协议之上，不需要特殊的网络配置或协议升级，因此易于实现和部署。客户端使用 EventSource API 来接收服务器发送的事件，并对事件进行处理。

WebSocket 是一种全双工的双向通信协议，可以在客户端和服务器之间建立持久的双向连接。WebSocket 建立了一个长久的连接通道，可以在客户端和服务器之间实时地发送消息。相比于 SSE，WebSocket 提供了双向通信的能力，允许客户端和服务器之间进行双向的实时数据交换。WebSocket 是一个独立的协议，它通过 HTTP 握手进行协议升级，之后通过保持长连接实现实时通信。客户端和服务器可以使用 WebSocket API 进行交互。

关键区别：

- SSE 是单向通信机制，只能服务器向客户端发送数据，而 WebSocket 是双向通信机制，允许双向的实时数据交换。

- SSE 基于 HTTP，使用普通的 HTTP 请求和响应，而 WebSocket 需要进行协议升级。

  > WebSocket 是一种独立的协议，与 HTTP 并非相同。虽然 WebSocket 的握手阶段使用 HTTP 请求和响应，但一旦握手成功，连接就升级为 WebSocket 协议，并且不再使用 HTTP。

- SSE 适合于服务器向客户端实时推送事件的场景，而 WebSocket 适用于需要实现双向通信的场景，例如聊天应用、协作工具等。

选择 SSE 还是 WebSocket 取决于具体的需求。如果只需要服务器向客户端推送实时事件，并且双向通信不是必需的，SSE 可能是更简单和合适的选择。如果需要实现双向通信，例如实时聊天应用，那么 WebSocket 是更适合的选择，因为它提供了双向的实时数据交换能力。

## 示例

引入依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```



```java
Executor executor = Executors.newFixedThreadPool(10);

@CrossOrigin
@GetMapping(value = "/api/stream/test", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter stream() {
  SseEmitter emitter = new SseEmitter();
  // 异步执行
  executor.execute(() -> {
    try {
      for (int i = 0; i < 3; i++) {
        emitter.send(SseEmitter.event().data("server time: " + LocalTime.now().toString()));
        Thread.sleep(1000);
      }
      // 结束标记，前端接收到该标志则断开连接
      emitter.send(SseEmitter.event().data("[Done]"));
      emitter.complete();
    } catch (IOException | InterruptedException e) {
      emitter.completeWithError(e);
      e.printStackTrace();
    }
  });
  return emitter;
}
```

!> 需要异步执行任务，及时返回 SseEmitter 对象；若同步执行任务，所有数据将一次性和 SseEmitter 对象返回到前端。

## nginx 缓冲机制

如果浏览器到服务端 sse 接口存在 nginx 转发时推送数据可能会发生问题。

```nginx
location /api/ {
  proxy_pass http://localhost:9265;
}
```

在当前配置下，默认开启缓冲机制。请求 sse 接口会将所有数据一次性返回到前端，并没有实现实时推送数据的效果。

关闭缓冲机制即可：

```nginx
location /api/ {
  proxy_pass http://localhost:9265;
  proxy_buffering off;
}
```



