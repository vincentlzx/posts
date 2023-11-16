## accessToken的数据结构

org.springframework.security.oauth2.common.OAuth2AccessToken

```java
@JsonSerialize(
    using = OAuth2AccessTokenJackson1Serializer.class
)
@JsonDeserialize(
    using = OAuth2AccessTokenJackson1Deserializer.class
)
@com.fasterxml.jackson.databind.annotation.JsonSerialize(
    using = OAuth2AccessTokenJackson2Serializer.class
)
@com.fasterxml.jackson.databind.annotation.JsonDeserialize(
    using = OAuth2AccessTokenJackson2Deserializer.class
)
public interface OAuth2AccessToken {
    String BEARER_TYPE = "Bearer";
    String OAUTH2_TYPE = "OAuth2";
    String ACCESS_TOKEN = "access_token";
    String TOKEN_TYPE = "token_type";
    String EXPIRES_IN = "expires_in";
    String REFRESH_TOKEN = "refresh_token";
    String SCOPE = "scope";

    Map<String, Object> getAdditionalInformation();

    Set<String> getScope();

    OAuth2RefreshToken getRefreshToken();

    String getTokenType();

    boolean isExpired();

    Date getExpiration();

    int getExpiresIn();

    String getValue();
}
```



## /oauth/token登录获取accessToken

1. 调用 /oauth/token 接口。this.getClientDetailsService().loadClientByClientId(clientId) 加载客户端配置信息，用于判断客户端请求是否可行。
2.  this.getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest) 会从 TokenGranter 集合中选择匹配当前 grand_type 的 ResourceOwnerPasswordTokenGranter 创建 accessToken 。
3. ResourceOwnerPasswordTokenGranter#getAccessToken 获取 accessToken 过程中 this.authenticationManager.authenticate(userAuth) 调用 UserDetailsService#loadUserByUsername 方法，实现该方法自定义身份验证逻辑。
4. 调用 tokenStore 保存 accessToken 再返回。

### grand_type为password的代码分析

org.springframework.security.oauth2.provider.endpoint.TokenEndpoint#postAccessToken

```java
@RequestMapping(
    value = {"/oauth/token"},
    method = {RequestMethod.POST}
)
public ResponseEntity<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
    if (!(principal instanceof Authentication)) {
        throw new InsufficientAuthenticationException("There is no client authentication. Try adding an appropriate authentication filter.");
    } else {
        String clientId = this.getClientId(principal);
        // 获取客户端配置信息
        ClientDetails authenticatedClient = this.getClientDetailsService().loadClientByClientId(clientId);
        // 根据请求参数创建TokenRequest对象
        TokenRequest tokenRequest = this.getOAuth2RequestFactory().createTokenRequest(parameters, authenticatedClient);
        if (clientId != null && !clientId.equals("") && !clientId.equals(tokenRequest.getClientId())) {
            throw new InvalidClientException("Given client ID does not match authenticated client");
        } else {
            if (authenticatedClient != null) {
                this.oAuth2RequestValidator.validateScope(tokenRequest, authenticatedClient);
            }

            if (!StringUtils.hasText(tokenRequest.getGrantType())) {
                throw new InvalidRequestException("Missing grant type");
            } else if (tokenRequest.getGrantType().equals("implicit")) {
                throw new InvalidGrantException("Implicit grant type not supported from token endpoint");
            } else {
                if (this.isAuthCodeRequest(parameters) && !tokenRequest.getScope().isEmpty()) {
                    this.logger.debug("Clearing scope of incoming token request");
                    tokenRequest.setScope(Collections.emptySet());
                }

                if (this.isRefreshTokenRequest(parameters)) {
                    tokenRequest.setScope(OAuth2Utils.parseParameterList((String)parameters.get("scope")));
                }
				// 根据grandType生成accessToken
                OAuth2AccessToken token = this.getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
                if (token == null) {
                    throw new UnsupportedGrantTypeException("Unsupported grant type: " + tokenRequest.getGrantType());
                } else {
                    return this.getResponse(token);
                }
            }
        }
    }
}

```

ClientDetails authenticatedClient = this.getClientDetailsService().loadClientByClientId(clientId); 获取认证中心配置的客户端信息。如下：

```java
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    clients.inMemory()
        .withClient("admin")//配置client_id
        .secret(passwordEncoder.encode("admin123456"))//配置client-secret
        .accessTokenValiditySeconds(20)//配置访问token的有效期
        .refreshTokenValiditySeconds(864000)//配置刷新token的有效期
//            .redirectUris("http://www.baidu.com")//配置redirect_uri，用于授权成功后跳转
        .redirectUris("http://localhost:9501/login")
        .autoApprove(true) //自动授权配置
        .scopes("all")//配置申请的权限范围
        .authorizedGrantTypes("authorization_code", "password", "refresh_token");//配置grant_type，表示授权类型
}
```

OAuth2AccessToken token = this.getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest); 根据 grantType 选择对应的 TokenGranter 创建 accessToken 。

org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerEndpointsConfigurer#tokenGranter()

```java
private TokenGranter tokenGranter() {
    if (this.tokenGranter == null) {
        this.tokenGranter = new TokenGranter() {
            private CompositeTokenGranter delegate;

            public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
                if (this.delegate == null) {
                    // 初始化，加载所有TokenGranters
                    this.delegate = new CompositeTokenGranter(AuthorizationServerEndpointsConfigurer.this.getDefaultTokenGranters());
                }
				// 根据grantType对应的TokenGranter创建accessToken
                return this.delegate.grant(grantType, tokenRequest);
            }
        };
    }

    return this.tokenGranter;
}
```

org.springframework.security.oauth2.provider.CompositeTokenGranter#grant

```java
public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
    Iterator i$ = this.tokenGranters.iterator();
	// 遍历TokenGranters，根据grantType对应的TokenGranter创建accessToken
    OAuth2AccessToken grant;
    do {
        if (!i$.hasNext()) {
            return null;
        }

        TokenGranter granter = (TokenGranter)i$.next();
        grant = granter.grant(grantType, tokenRequest);
    } while(grant == null);

    return grant;
}
```

选择 grantType 为 password 的 ResourceOwnerPasswordTokenGranter 。

ResourceOwnerPasswordTokenGranter#grant

```java
public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
    if (!this.grantType.equals(grantType)) {
        return null;
    } else {
        String clientId = tokenRequest.getClientId();
        // 获取客户端配置信息
        ClientDetails client = this.clientDetailsService.loadClientByClientId(clientId);
        this.validateGrantType(grantType, client);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Getting access token for: " + clientId);
        }
 		// 获取accessToken
        return this.getAccessToken(client, tokenRequest);
    }
}
```

ResourceOwnerPasswordTokenGranter#getAccessToken

```java
protected OAuth2AccessToken getAccessToken(ClientDetails client, TokenRequest tokenRequest) {
    // getOAuth2Authentication获取用户身份验证信息，根据OAuth2Authentication创建accessToken并保存到redis
    return this.tokenServices.createAccessToken(this.getOAuth2Authentication(client, tokenRequest));
}
```

ResourceOwnerPasswordTokenGranter#getOAuth2Authentication

```java
protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest tokenRequest) {
    Map<String, String> parameters = new LinkedHashMap(tokenRequest.getRequestParameters());
    String username = (String)parameters.get("username");
    String password = (String)parameters.get("password");
    parameters.remove("password");
    Authentication userAuth = new UsernamePasswordAuthenticationToken(username, password);
    ((AbstractAuthenticationToken)userAuth).setDetails(parameters);

    Authentication userAuth;
    try {
        // 注意，会调用自定义的UserDetailsService#loadUserByUsername方法
        userAuth = this.authenticationManager.authenticate(userAuth);
    } catch (AccountStatusException var8) {
        throw new InvalidGrantException(var8.getMessage());
    } catch (BadCredentialsException var9) {
        throw new InvalidGrantException(var9.getMessage());
    }

    if (userAuth != null && userAuth.isAuthenticated()) {
        OAuth2Request storedOAuth2Request = this.getRequestFactory().createOAuth2Request(client, tokenRequest);
        return new OAuth2Authentication(storedOAuth2Request, userAuth);
    } else {
        throw new InvalidGrantException("Could not authenticate user: " + username);
    }
}
```

this.authenticationManager.authenticate(userAuth) 会调用自定义的UserDetailsService#loadUserByUsername方法。

org.springframework.security.oauth2.provider.token.DefaultTokenServices#createAccessToken(org.springframework.security.oauth2.provider.OAuth2Authentication)

```java
@Transactional
public OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException {
    OAuth2AccessToken existingAccessToken = this.tokenStore.getAccessToken(authentication);
    OAuth2RefreshToken refreshToken = null;
    if (existingAccessToken != null) {
        if (!existingAccessToken.isExpired()) {
            this.tokenStore.storeAccessToken(existingAccessToken, authentication);
            return existingAccessToken;
        }

        if (existingAccessToken.getRefreshToken() != null) {
            refreshToken = existingAccessToken.getRefreshToken();
            this.tokenStore.removeRefreshToken(refreshToken);
        }

        this.tokenStore.removeAccessToken(existingAccessToken);
    }

    if (refreshToken == null) {
        refreshToken = this.createRefreshToken(authentication);
    } else if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
        ExpiringOAuth2RefreshToken expiring = (ExpiringOAuth2RefreshToken)refreshToken;
        if (System.currentTimeMillis() > expiring.getExpiration().getTime()) {
            refreshToken = this.createRefreshToken(authentication);
        }
    }

    OAuth2AccessToken accessToken = this.createAccessToken(authentication, refreshToken);
    this.tokenStore.storeAccessToken(accessToken, authentication);
    refreshToken = accessToken.getRefreshToken();
    if (refreshToken != null) {
        this.tokenStore.storeRefreshToken(refreshToken, authentication);
    }

    return accessToken;
}
```

可以看到，先调用 tokenStore 保存 accessToken 后返回 accessToken 。

## /oauth/token更新accessToken

流程和 grant_type 为 password 的差不多，主要有一下区别：

1. TokenGranter 对应 grant_type 为 refresh_token 的是 RefreshTokenGranter 。

2. getAccessToken 步骤主要是更新 accessToken ，同样会调用自定义的 UserDetailsService#loadUserByUsername 方法。

   org.springframework.security.oauth2.provider.refresh.RefreshTokenGranter#getAccessToken

   ```java
   protected OAuth2AccessToken getAccessToken(ClientDetails client, TokenRequest tokenRequest) {
       String refreshToken = (String)tokenRequest.getRequestParameters().get("refresh_token");
       return this.getTokenServices().refreshAccessToken(refreshToken, tokenRequest);
   }
   ```

   org.springframework.security.oauth2.provider.token.DefaultTokenServices#refreshAccessToken

   ```java
   @Transactional(
       noRollbackFor = {InvalidTokenException.class, InvalidGrantException.class}
   )
   public OAuth2AccessToken refreshAccessToken(String refreshTokenValue, TokenRequest tokenRequest) throws AuthenticationException {
       if (!this.supportRefreshToken) {
           throw new InvalidGrantException("Invalid refresh token: " + refreshTokenValue);
       } else {
           OAuth2RefreshToken refreshToken = this.tokenStore.readRefreshToken(refreshTokenValue);
           if (refreshToken == null) {
               throw new InvalidGrantException("Invalid refresh token: " + refreshTokenValue);
           } else {
               OAuth2Authentication authentication = this.tokenStore.readAuthenticationForRefreshToken(refreshToken);
               if (this.authenticationManager != null && !authentication.isClientOnly()) {
                   Authentication user = new PreAuthenticatedAuthenticationToken(authentication.getUserAuthentication(), "", authentication.getAuthorities());
                   // 注意，会调用自定义的UserDetailsService#loadUserByUsername方法
                   Authentication user = this.authenticationManager.authenticate(user);
                   Object details = authentication.getDetails();
                   authentication = new OAuth2Authentication(authentication.getOAuth2Request(), user);
                   authentication.setDetails(details);
               }
   
               String clientId = authentication.getOAuth2Request().getClientId();
               if (clientId != null && clientId.equals(tokenRequest.getClientId())) {
                   // 根据refreshToken移除accessToken
                   this.tokenStore.removeAccessTokenUsingRefreshToken(refreshToken);
                   if (this.isExpired(refreshToken)) {
                       this.tokenStore.removeRefreshToken(refreshToken);
                       throw new InvalidTokenException("Invalid refresh token (expired): " + refreshToken);
                   } else {
                       authentication = this.createRefreshedAuthentication(authentication, tokenRequest);
                       if (!this.reuseRefreshToken) {
                           // 若不是重复使用refreshToken，则移除refreshToken创建新的refreshToken
                           this.tokenStore.removeRefreshToken(refreshToken);
                           refreshToken = this.createRefreshToken(authentication);
                       }
   					// 根据refreshToken创建新的accessToken
                       OAuth2AccessToken accessToken = this.createAccessToken(authentication, refreshToken);
                       // 保存新的accessToken
                       this.tokenStore.storeAccessToken(accessToken, authentication);
                       if (!this.reuseRefreshToken) {
                           // 保存新的refreshToken
                           this.tokenStore.storeRefreshToken(accessToken.getRefreshToken(), authentication);
                       }
   
                       return accessToken;
                   }
               } else {
                   throw new InvalidGrantException("Wrong client for this refresh token: " + refreshTokenValue);
               }
           }
       }
   }
   ```

   

