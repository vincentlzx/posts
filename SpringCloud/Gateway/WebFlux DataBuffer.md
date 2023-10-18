---
title: WebFlux DataBuffer是什么
categories:
- [SpringCloud, Gateway]
tags:
- SpringCloud
- Gateway
- WebFlux
---



## Webflux DataBuffer

`DataBuffer` 是 Spring Framework 5 中的一个重要概念，它是用于表示字节数据的抽象。在 Spring WebFlux 中，`DataBuffer` 主要用于处理请求体、响应体以及其他字节数据的场景。

以下是有关 `DataBuffer` 的详细解释：

1. **概念：** `DataBuffer` 是 Spring 框架提供的一种字节数据缓冲区抽象。它的设计目的是为了在异步编程模型中更高效地处理字节数据，同时也可以在阻塞环境中使用。
2. **用途：** 在 Spring WebFlux 中，`DataBuffer` 用于表示 HTTP 请求和响应的字节数据，如请求体、响应体等。它还可以用于文件上传和下载、缓冲区数据的处理等场景。
3. **创建和获取：** 你可以使用 `DataBufferFactory` 来创建和获取 `DataBuffer` 实例。`DefaultDataBufferFactory` 是 Spring 提供的默认实现，你可以使用它来创建 `DataBuffer`。
4. **操作：** 你可以使用 `DataBuffer` 的方法来操作字节数据，例如读取数据、写入数据、获取字节长度等。操作与常见的字节流操作类似。
5. **释放资源：** 由于 `DataBuffer` 通常在异步环境中使用，你应该在使用完 `DataBuffer` 后显式释放它，以回收内存资源。你可以使用 `DataBufferUtils.release()` 方法来释放 `DataBuffer`。
6. **组合和切割：** 你可以通过组合多个 `DataBuffer` 来构建更大的字节数据块，或者通过切割一个 `DataBuffer` 来获取特定的字节范围。
7. **适用于响应式：** `DataBuffer` 是响应式编程的一部分，它可以与 `Flux` 和 `Mono` 一起使用，实现异步、非阻塞的数据处理。
8. **适用于网络：** 在网络编程中，`DataBuffer` 可以用于表示收到的网络数据、发送的网络数据等。例如，在处理 HTTP 请求时，请求体就可以被表示为一个 `Flux<DataBuffer>`。

总之，`DataBuffer` 是 Spring WebFlux 中用于处理字节数据的重要组件，特别适用于异步、非阻塞的编程模型。在处理请求体、响应体、文件上传下载等场景时，使用 `DataBuffer` 可以帮助你高效地进行字节数据的操作和处理。

**示例1：使用 `DataBuffer` 处理请求体中的数据**

```java
@PostMapping("/post")
public Mono<String> post(ServerWebExchange exchange) {
  return exchange.getRequest()
    .getBody()
    // 获取缓冲区内容集合
    .collectList()
    .map(dataBuffers -> {
      DataBufferFactory dataBufferFactory = new DefaultDataBufferFactory();
      DataBuffer join = dataBufferFactory.join(dataBuffers);
      byte[] content = new byte[join.readableByteCount()];
      join.read(content);
      DataBufferUtils.release(join);
      return new String(content, StandardCharsets.UTF_8);
    });
}
```

在上述示例中，使用 `exchange.getRequest().getBody().collectList()` 获取请求体的 `DataBuffer` 集合，使用 DataBufferFactory 并将这些 `DataBuffer` 逐段地读取并拼接成一个完整的内容。

**示例2：实现跨站脚本过滤器，重写 WebFlux 的请求体内容**

主要思路就是读取 WebFlux DataBuffer 缓冲区获取字节数组，将字节数组转换为字符串后清除其 HTML 内容，将修改后的字符串在转换为 DataBuffer 对象返回。

```java
public class XssFilter implements GlobalFilter, Ordered {
    private final XssProperties xss;

    public XssFilter(XssProperties xss) {
        this.xss = xss;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("【全局过滤器】：XssFilter");
        ServerHttpRequest request = exchange.getRequest();
        // xss开关未开启 或 通过nacos关闭，不过滤
        if (!xss.getEnabled()) {
            return chain.filter(exchange);
        }
        // GET DELETE 不过滤
        HttpMethod method = request.getMethod();
        if (method == null || method == HttpMethod.GET || method == HttpMethod.DELETE) {
            return chain.filter(exchange);
        }
        // 非json类型，不过滤
        if (!isJsonRequest(exchange)) {
            return chain.filter(exchange);
        }
        // excludeUrls 不过滤
        String url = request.getURI().getPath();
        if (xss.getExcludeUrls().contains(url)) {
            return chain.filter(exchange);
        }
        ServerHttpRequestDecorator httpRequestDecorator = requestDecorator(exchange);
        return chain.filter(exchange.mutate().request(httpRequestDecorator).build());

    }

    private ServerHttpRequestDecorator requestDecorator(ServerWebExchange exchange) {
        return new ServerHttpRequestDecorator(exchange.getRequest()) {
            @Override
            public Flux<DataBuffer> getBody() {
                Flux<DataBuffer> body = super.getBody();
                return body.buffer().map(dataBuffers -> {
                    DataBufferFactory dataBufferFactory = new DefaultDataBufferFactory();
                    // 使用DefaultDataBufferFactory.join组合DataBuffer集合，读取DataBuffer
                    DataBuffer join = dataBufferFactory.join(dataBuffers);
                    byte[] content = new byte[join.readableByteCount()];
                    join.read(content);
                    // 释放资源
                    DataBufferUtils.release(join);
                    String bodyStr = new String(content, StandardCharsets.UTF_8);
                    log.info("Request Body: " + bodyStr);
                    // 防xss攻击过滤，即清除HTML内容，修改字符串内容
                    bodyStr = EscapeUtil.clean(bodyStr);
                    // 转成字节
                    byte[] bytes = bodyStr.getBytes();
                    // 使用NettyDataBufferFactory创建一个新的DataBuffer返回
                    NettyDataBufferFactory nettyDataBufferFactory = new NettyDataBufferFactory(ByteBufAllocator.DEFAULT);
                    DataBuffer buffer = nettyDataBufferFactory.allocateBuffer(bytes.length);
                    buffer.write(bytes);
                    return buffer;
                });
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders httpHeaders = new HttpHeaders();
                httpHeaders.putAll(super.getHeaders());
                // 由于修改了请求体的body，导致content-length长度不确定，因此需要删除原先的content-length
                httpHeaders.remove(HttpHeaders.CONTENT_LENGTH);
                httpHeaders.set(HttpHeaders.TRANSFER_ENCODING, "chunked");
                return httpHeaders;
            }

        };
    }

    /**
     * 是否是Json请求
     *
     * @param exchange HTTP请求
     */
    public boolean isJsonRequest(ServerWebExchange exchange) {
        String contentType = exchange.getRequest().getHeaders().getFirst(HttpHeaders.CONTENT_TYPE);
        return StrUtil.isNotEmpty(contentType) && contentType.startsWith(MediaType.APPLICATION_JSON_VALUE);
    }

    @Override
    public int getOrder() {
        return -100;
    }
}
```







