## 角色定义

**Resource Owner**：资源所有者，通常是指登录用户。

**Client**：客户端，第三方客户端，被授权访问的应用。

**Authorization server**：认证服务器，即服务提供商专门用来处理认证的服务器。

**Resource server**：资源服务器，即服务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器。

> 比如我参与开发的客来鸟LOGO网站，通过关注微信公众号登录。Resource Owner 就是我本人；Client 就是客来鸟LOGO网站；Authorization server 就是微信公众号的认证服务器，比如获取 Access token；Resource server 就是微信公众号的资源服务器，比如验证来自客户端的访问令牌并决定是否允许获取微信用户基本信息。
>
> Authorization server 和 Resource server 是一体的，都是来自同一个服务提供商，比如微信。在一些简单的应用或小规模系统中，可能会选择将认证服务器和资源服务器集成在一起。这种情况下，认证和资源管理都在同一服务器上完成。
>
> 认证服务器和资源服务器的区别：
>
> 1. **认证服务器：** 负责处理用户身份验证、颁发访问令牌等认证流程。通常，认证服务器专注于身份验证和令牌管理。
> 2. **资源服务器：** 负责管理和提供受保护资源。资源服务器验证来自客户端的访问令牌，并决定是否允许客户端访问资源。

## 运行流程

![](https://file.liuzx.com.cn/docsify-pic/20231121151931.png)

来源：https://www.youtube.com/watch?app=desktop&v=ZV5yTm4pT8g

假设 Client 第三方应用需要调用 Google 服务提供商的服务。

1. 用户打开客户端以后，客户端要求用户给予授权。（授予访问 Google 服务的权限）

2. 用户同意给予客户端授权。

   > Google 服务提供商需要考虑使用什么模式使用户给予客户端授权。授权码模式是最严密的授权模式，用户授权过程中不会暴露账号密码给第三方应用；密码模式，用户向客户端提供自己的用户名和密码。客户端使用这些信息，向"服务商提供商"索要授权。

3. 客户端使用上一步获得的授权，向认证服务器申请令牌。

4. 认证服务器对客户端进行认证以后，确认无误，同意发放令牌。

5. 客户端使用令牌，向资源服务器申请获取资源。

6. 资源服务器确认令牌无误，同意向客户端开放资源。

   > 资源服务器验证令牌有效性方式有两种：1. 向认证服务器的 Token Introspection Endpoint （/oatuh/check_token）发送 HTTP 请求；2. 直接验证 JWT ，不需要与认证服务器通信；3. 资源服务器和认证服务器共享 Redis 数据库，判断 Redis 中是否存在 accessToken 。

## 客户端的授权模式

客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0定义了四种授权方式。

- 授权码模式（authorization code）
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）

## 授权码模式

授权码模式（authorization code）是功能最完整、流程最严密的授权模式。它的特点就是通过客户端的后台服务器，与"服务提供商"的认证服务器进行互动。

![](https://file.liuzx.com.cn/docsify-pic/20231121155016.png)

来源：https://portswigger.net/web-security/oauth/grant-types#authorization-code-grant-type

它的步骤如下：

1. 用户访问客户端，后者将前者导向认证服务器。

2. 用户选择是否给予客户端授权。
3. 假设用户给予授权，认证服务器将用户导向客户端事先指定的"重定向URI"（redirection URI），同时附上一个授权码。

4. 客户端收到授权码，附上早先的"重定向URI"，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。

5. 认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。
6. 客户端收到令牌后，可以通过携带令牌请求资源服务器获取资源。
7. 资源服务器验证来自客户端的访问令牌，并决定是否允许客户端访问资源。

结合微信公众号网页授权分析流程：

步骤 1 中，客户端引导关注者打开微信公众号的授权页面：

https://open.weixin.qq.com/connect/oauth2/authorize?appid=APPID&redirect_uri=REDIRECT_URI&response_type=code&scope=SCOPE&state=STATE#wechat_redirect

![](https://file.liuzx.com.cn/docsify-pic/20231121155923.png)

步骤 3 中，如果用户同意授权，微信认证服务器回应客户端的 URI ，页面将跳转至 redirect_uri/?code=CODE&state=STATE。

步骤 4 中，客户端通过 code 向微信认证服务器换取网页授权 access_token ：

https://api.weixin.qq.com/sns/oauth2/access_token?appid=APPID&secret=SECRET&code=CODE&grant_type=authorization_code

步骤 5 中，微信认证服务器返回 JSON 数据包：

```json
{
  "access_token":"ACCESS_TOKEN",
  "expires_in":7200,
  "refresh_token":"REFRESH_TOKEN",
  "openid":"OPENID",
  "scope":"SCOPE",
  "is_snapshotuser": 1,
  "unionid": "UNIONID"
}
```

步骤 6 中，客户端可以通过 access_token 和 openid 向微信资源服务器拉取用户信息了：

https://api.weixin.qq.com/sns/userinfo?access_token=ACCESS_TOKEN&openid=OPENID&lang=zh_CN

## 参考

理解OAuth 2.0：https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

OAuth2 协议中如何对 accessToken 进行校验：https://blog.51cto.com/u_15989526/6288503

