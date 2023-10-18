---
title: Gateway中操作POST请求体DataBuffer
categories:
- [SpringCloud, Gateway]
tags:
- SpringCloud
- Gateway
---



## 修改POST请求体

实现跨站脚本过滤器，修改 post json 请求体内容为例：

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

## 获取POST请求体

自定义验证码过滤器，获取请求体内容为例：

登录/注册请求体为 {"userName": "liuzx", "password": "p@ssw0rd", "code": "0643"} ，验证码过滤器需要先验证请求体参数 code 有效，再转发到 auth 服务校验用户名和密码。

```java
public class ValidateCodeFilter extends AbstractGatewayFilterFactory<Object> {
    private final static List<String> VALIDATE_URL = new ArrayList<>(Arrays.asList("/auth/login", "/auth/register"));

    private static final String CODE = "code";

    private static final String UUID = "uuid";

    @Override
    public GatewayFilter apply(Object config) {
        return (exchange, chain) -> {
            log.info("【局部过滤器】：ValidateCodeFilter");
            // 非登录/注册请求或验证码关闭，不处理
            if (!VALIDATE_URL.contains(exchange.getRequest().getURI().getPath()))
            {
                return chain.filter(exchange);
            }
            // 使用flatMap来处理异步操作，并返回一个Mono<Void>
            return DataBufferUtils.join(exchange.getRequest().getBody())
                .flatMap(dataBuffer -> {
                    // 解码请求体内容
                    String requestBody = decodeRequestBody(dataBuffer);

                    // 解析请求体内容，验证验证码
                    log.info("Request Body: " + requestBody);
                    JSONObject obj = JSON.parseObject(requestBody);
                    validateCodeService.checkCaptcha(obj.getString(CODE), obj.getString(UUID));

                    // 释放DataBuffer
                    DataBufferUtils.release(dataBuffer);

                    // 继续处理请求
                    return chain.filter(exchange);
                });
        };
    }

    private String decodeRequestBody(DataBuffer dataBuffer) {
        byte[] bytes = new byte[dataBuffer.readableByteCount()];
        dataBuffer.read(bytes);
        // 解码请求体内容，这里可以根据实际情况选择合适的编码方式
        return new String(bytes, StandardCharsets.UTF_8);
    }
}
```

> 默认情况下，对于 POST 请求的请求体（request body）只能读取一次。一旦请求体被读取，它就会从输入流中移除，无法再次获取。所以验证码过滤器读取请求体验证验证码后，后面的过滤器将无法再次读取到请求体内容，和后续路由到相应的服务接口也无法获取到请求体内容.

例如后续网关将请求转发到 `http://localhost:8080/auth/login` 返回响应内容，可以看到 POST 请求体被过滤器读取一次后，就不能再次读取。

```java
{
    "msg": "Required request body is missing: public com.ruoyi.common.core.domain.R<?> com.ruoyi.auth.controller.TokenController.login(com.ruoyi.auth.form.LoginBody)",
    "code": 500
}
```

## 重复读取请求体内容

如果需要在 Gateway 过滤器中多次读取 POST 请求的请求体内容，可以使用 `ServerWebExchangeUtils.cacheRequestBodyAndRequest` 方法来缓存请求体，并创建一个新的 `ServerWebExchange` 对象。这样，在新的 `ServerWebExchange` 对象中可以多次读取缓存的请求体内容。

```java
public class CacheRequestFilter extends AbstractGatewayFilterFactory<CacheRequestFilter.Config> {
    public CacheRequestFilter() {
        super(Config.class);
    }

    @Override
    public String name() {
        return "CacheRequestFilter";
    }

    @Override
    public GatewayFilter apply(Config config) {
        return new OrderedGatewayFilter(new GatewayFilter() {
            @Override
            public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
                log.info("【局部过滤器】：{}", "CacheRequestGatewayFilter");
                // GET DELETE 不过滤
                HttpMethod method = exchange.getRequest().getMethod();
                if (method == null || method == HttpMethod.GET || method == HttpMethod.DELETE) {
                    return chain.filter(exchange);
                }
                return ServerWebExchangeUtils.cacheRequestBodyAndRequest(exchange, (serverHttpRequest) -> {
                    if (serverHttpRequest == exchange.getRequest()) {
                        return chain.filter(exchange);
                    }
                    return chain.filter(exchange.mutate().request(serverHttpRequest).build());
                });
            }
        }, config.getOrder());
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Collections.singletonList("order");
    }

    static class Config {
        private Integer order = 0;

        public Integer getOrder() {
            return order;
        }

        public void setOrder(Integer order) {
            this.order = order;
        }
    }
}
```

