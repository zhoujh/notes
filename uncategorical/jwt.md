# Bearer 与JWT(json web token) 

问题背景

在多个资源系统需要通过认证系统进认证的情况下，基于session/cookie的方式存在的问题

1）跨域的问题，不同域下的cookie/session无法共享。

2）即使使用同一个域名共享cookie ,资源系统通常需要回调认证中心验证cookie种用户token的有有效性。

解决办法

Http Bearer 认证方式是解决第一个问题的一种方式。客户端通过在 资源系统通过Http Header的Authentication获取Bearer Token信息完成身份验证。

JWT是Token的编码的一种规则。 采用json格式，格式如下：

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJuYmYiOjE2NTcxNjI4MzksImV4cCI6MTY1OTc1NDgzOSwiaWF0IjoxNjU3MTYyODM5LCJ1c2VyIjp7ImlkIjoxMDAwMywidXNlcm5hbWUxxxxxxxxxxx.9aBNWsGPauPBubGRi72Kr27KERxxxxxxxxxx
```

header、claim、签名  三个部分，用点号分隔 。 

header 部分内容结构固定，包含typ和alg两部分。

payload 部分内容认证中心自定义，一般包含用户信息和授权信息。

signature 通过hash算法对header和payload 进行签名的结果。

header:  {"typ":"JWT","alg":"HS256"}   
claim: {"nbf":1657162839,"exp":1659754839,"iat":1657162839,"user":{"id":10003,"username":"bot3"}} 

signature ：9aBNWsGPauPBubGRi72Kr27KERxxxxxxxxxx 



存在的问题，使用时需要避开。

1. 有效期保证在token种，无法使token提前失效，只能通过保存/同步强制失效的token，让资源服务器获得失效的token信息。
2. token中保存的信息不加密，可能泄露信息。
3. 如果token包含详细的权限的信息，token长度会变大， 类似keystone 中的PKI token 大小很容易超过10k 。



[HTTP authentication - HTTP | MDN (mozilla.org)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication)