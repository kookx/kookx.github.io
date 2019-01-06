---
layout: post
title: Spring Security与OAuth2介绍
category: Blog
tags: [Spring Security,OAuth2]
---

# OAuth2 初识

OAuth2.0是[OAuth](http://en.wikipedia.org/wiki/OAuth)协议的延续版本，OAuth是一个关于授权（authorization）的开放网络标准，但不[向后兼容](https://baike.baidu.com/item/%E5%90%91%E5%90%8E%E5%85%BC%E5%AE%B9/94553)OAuth 1.0即完全废止了OAuth1.0，OAuth 2.0关注客户端开发者的简易性。要么通过组织在资源拥有者和HTTP服务商之间的被批准的交互动作代表用户，要么允许第三方应用代表用户获得访问的权限。同时为Web应用，桌面应用和手机，和起居室设备提供专门的认证流程，在全世界得到广泛应用。

### 应用场景

> OAuth2 可以方便第三方应用(如豆瓣)获取用户在其他应用(如QQ)的信息

- 比如用 QQ 账户登录优酷，优酷就会先让用户登录 QQ，然后让用户确认授权优酷访问 QQ 上的信息，确认后优酷就获得了 QQ 的 OAuth 服务器返回的 token，之后就可以通过 token 访问到 QQ 允许第三方应用访问的资源路径范围内的用户相关信息。

> 允许用户让第三方应用访问该用户在某一网站上存储的私密的资源（如照片，视频，联系人列表）

- 比如你浏览某个网站的技术文章，发现其中某段介绍的不够详细，想留言给作者提问，点击`评论`，结果发现需要有这个网站的账号才能留言，此时有两个选择，一个是新注册一个此网站的账号，二是点击通过github快速登录。前者你觉得过于繁琐，直接点击了github登录，此时，OAuth的认证流程就开始了。通过引导跳转到github界面，会提示你是否授权该网站使用你的github用户信息，点击确认，跳转回原网站，发现已经使用你的github账号默认注册了一个用户，而且还不需要用户名和密码，便捷高效。

- 假如有一个云冲印的网站，可以将你存储在Google的照片冲印出来，用户为了使用该服务，必须让云冲印读取Google上的照片。为了拿到照片，云冲印必须得拿到一个用户的授权，如何获取这个用户授权呢？传统方法是用户将用户名和密码告诉云冲印，那么云冲印就可以自由无限制的访问了（相当于用户自己访问），这样显然是不行的，有几个严重的缺点：

  - 云冲印为了保存后续服务，会保存用户的密码，这样很不安全。
  - 云冲印拥有了获取用户存储在Google的所有资料的权力，用户没法限制云冲印得到的授权范围和授权有效期。
  - 用户只有修改密码，才能收回赋予云冲印的权力，但是如果还授权给了其他的应用，那么密码的修改将影响到所有被授权应用。
  - 只要有一个第三方应用程序被破解，就会导致用户密码泄漏，以及所有被密码保护的数据泄漏。

OAuth 2协议以及详细说明可以参考以下是相关的文章： 

- [RFC 6749 ](https://tools.ietf.org/html/rfc6749)
- [阮一峰 – 理解 OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html) 
- [OAuth2 CSRF攻击](https://www.jianshu.com/p/c7c8f51713b6)
- [一张图搞定OAuth2.0](https://www.cnblogs.com/flashsun/p/7424071.html)

### 理解上可能存在的疑问

> 第一个疑问： CAS的单点登录和OAuth2的最大区别

SSO ：单点登录（Single sign-on）是在多个应用系统中，用户只需要登录一次就可以访问所有相互信任的应用系统。（来自百度百科）。

CAS ：中央认证服务（Central Authentication Service），一个基于Kerberos票据方式实现SSO单点登录的框架，为Web 应用系统提供一种可靠的单点登录解决方法（属于 Web SSO ）。

1. 　　CAS的单点登录时保障客户端的用户资源的安全 。

　　OAuth2则是保障服务端的用户资源的安全 。

2. 　　CAS客户端要获取的最终信息是，这个用户到底有没有权限访问我（CAS客户端）的资源。

　　OAuth2获取的最终信息是，我（oauth2服务提供方）的用户的资源到底能不能让你（oauth2的客户端）访问。

3. 　　CAS的单点登录，资源都在客户端这边，不在CAS的服务器那一方。 用户在给CAS服务端提供了用户名密码后，作为CAS客户端并不知道这件事。 随便给客户端个ST，那么客户端是不能确定这个ST是用户伪造还是真的有效，所以要拿着这个ST去服务端再问一下，这个用户给我的是有效的ST还是无效的ST，是有效的我才能让这个用户访问。

　　OAuth2认证，资源都在OAuth2服务提供者那一方，客户端是想索取用户的资源。 所以在最安全的模式下，用户授权之后，服务端并不能直接返回token，通过重定向送给客户端，因为这个token有可能被黑客截获，如果黑客截获了这个token，那用户的资源也就暴露在这个黑客之下了。 于是聪明的服务端发送了一个认证code给客户端（通过重定向），客户端在后台，通过https的方式，用这个code，以及另一串客户端和服务端预先商量好的密码，才能获取到token和刷新token，这个过程是非常安全的(整个oauth流程采用tls加密，会进行证书校验，这样降低了数据传输过程中被篡改或暴露的可能性)。 如果黑客截获了code，他没有那串预先商量好的密码，他也是无法获取token的。这样oauth2就能保证请求资源这件事，是用户同意的，客户端也是被认可的，可以放心的把资源发给这个客户端了。

所以cas登录和OAuth2在流程上的最大区别就是，通过ST或者code去认证的时候，需不需要预先商量好的密码。

以上转载：https://www.cnblogs.com/flying607/p/7652537.html

---

> 第二个疑问： OAuth2 和JWT区别与联系

场景：

1. 你已经或者正在实现API。

2. 你正在考虑选择一个合适的方法保证API的安全性。

要比较JWT和OAuth2，首先要明白一点就是，这两个根本没有可比性，是两个完全不同的东西。

**JWT是一种认证协议**

JWT提供了一种用于发布接入令牌（Access Token),并对发布的签名接入令牌进行验证的方法。 令牌（Token）本身包含了一系列声明，应用程序可以根据这些声明限制用户对资源的访问。

**OAuth2是一种授权框架**

另一方面，OAuth2是一种授权框架，提供了一套详细的授权机制（指导）。用户或应用可以通过公开的或私有的设置，授权第三方应用访问特定资源。

**为什么要比较**

既然JWT和OAuth2没有可比性，为什么还要把这两个放在一起说呢？实际中确实会有很多人拿JWT和OAuth2作比较。很多情况下，在讨论OAuth2的实现时，`会把JSON Web Token作为一种认证机制使用`。这也是为什么他们会经常一起出现。

简单来说：`应用场景不一样`

1. OAuth2用在使用第三方账号登录的情况(比如使用weibo, qq, github登录某个app)

2. JWT是用在前后端分离, 需要简单的对后台API进行保护时使用.(前后端分离无session, 频繁传用户密码不安全)

OAuth2是一个相对复杂的协议, 有4种授权模式, 其中的`access code模式在实现时可以使用jwt才生成code`, 也可以不用. 它们之间没有必然的联系；oauth2有client和scope的概念，jwt没有。如果只是拿来用于颁布token的话，二者没区别。常用的bearer算法oauth、jwt都可以用，只是应用场景不同而已。

---

# Spring Security OAuth2

> #### 什么是Spring Security OAuth2

Spring Security OAuth2建立在Spring Security的基础之上，实现了OAuth2的规范，官方原文链接：**[http://projects.spring.io/spring-security-oauth/docs/oauth2.html](http://projects.spring.io/spring-security-oauth/docs/oauth2.html)**

> #### Spring OAuth2.0 提供者实现原理

Spring OAuth2.0提供者实际上分为：

- 授权服务 Authorization Service.
- 资源服务 Resource Service.

虽然这两个提供者有时候可能存在同一个应用程序中，但在Spring Security OAuth中你可以把

他它们各自放在不同的应用上，而且你可以有多个资源服务，它们共享同一个中央授权服

务。

所有获取令牌的请求都将会在Spring MVC controller endpoints中进行处理，并且访问受保护

的资源服务的处理流程将会放在标准的Spring Security请求过滤器中(filters)。

下面是配置一个授权服务必须要实现的endpoints：

- AuthorizationEndpoint：用来作为请求者获得授权的服务，默认的URL是/oauth/authorize.
- TokenEndpoint：用来作为请求者获得令牌（Token）的服务，默认的URL是/oauth/token.

下面是配置一个资源服务必须要实现的过滤器：

- OAuth2AuthenticationProcessingFilter：用来作为认证令牌（Token）的一个处理流程过滤器。只有当过滤器通过之后，请求者才能获得受保护的资源。

配置提供者（授权、资源）都可以通过简单的Java注解@Configuration来进行适配，你也可以使用基于XML的声明式语法来进行配置，如果你打算这样做的话，那么请使用http://www.springframework.org/schema/security/spring-security-oauth2.xsd来作为XML的schema（即XML概要定义）以及使用http://www.springframework.org/schema/security/oauth2来作为命名空间。

> #### 授权服务配置

配置一个授权服务，你需要考虑几种授权类型（Grant Type），不同的授权类型为客户端（Client）提供了不同的获取令牌（Token）方式，为了实现并确定这几种授权，需要配置使用 ClientDetailsService 和 TokenService 来开启或者禁用这几种授权机制。到这里就请注意了，不管你使用什么样的授权类型（Grant Type），每一个客户端（Client）都能够通过明确的配置以及权限来实现不同的授权访问机制。这也就是说，假如你提供了一个支持"client_credentials"的授权方式，并不意味着客户端就需要使用这种方式来获得授权。下面是几种授权类型的列表，具体授权机制的含义可以参见RFC6749([中文版本](https://github.com/jeansfish/RFC6749.zh-cn))：

- authorization_code：授权码类型。
- implicit：隐式授权类型。
- password：资源所有者（即用户）密码类型。
- client_credentials：客户端凭据（客户端ID以及Key）类型。
- refresh_token：通过以上授权获得的刷新令牌来获取新的令牌。

可以用 @EnableAuthorizationServer 注解来配置OAuth2.0 授权服务机制，通过使用@Bean注解的几个方法一起来配置这个授权服务。下面咱们介绍几个配置类，这几个配置是由Spring创建的独立的配置对象，它们会被Spring传入AuthorizationServerConfigurer中：

- ClientDetailsServiceConfigurer：用来配置客户端详情服务（ClientDetailsService），客户端详情信息在这里进行初始化，你能够把客户端详情信息写死在这里或者是通过数据库来存储调取详情信息。
- AuthorizationServerSecurityConfigurer：用来配置令牌端点(Token Endpoint)的安全约束.
- AuthorizationServerEndpointsConfigurer：用来配置授权（authorization）以及令牌（token）的访问端点和令牌服务(token services)。

（译者注：以上的配置可以选择继承AuthorizationServerConfigurerAdapter并且覆写其中的三个configure方法来进行配置。）

配置授权服务一个比较重要的方面就是提供一个授权码给一个OAuth客户端（通过 authorization_code 授权类型），一个授权码的获取是OAuth客户端跳转到一个授权页面，然后通过验证授权之后服务器重定向到OAuth客户端，并且在重定向连接中附带返回一个授权码。

如果你是通过XML来进行配置的话，那么可以使用 <authorization-server/> 标签来进行配置。

（译者注：想想现在国内各大平台的社会化登陆服务，例如腾讯，用户要使用QQ登录到某个网站，这个网站是跳转到了腾讯的登陆授权页面，然后用户登录并且确定授权之后跳转回目标网站，这种授权方式规范在我上面提供的链接*RFC6749*的第4.1节有详细阐述。）

**配置客户端详情信息（Client Details)：**

ClientDetailsServiceConfigurer (AuthorizationServerConfigurer 的一个回调配置项，见上的概述) 能够使用内存或者JDBC来实现客户端详情服务（ClientDetailsService），有几个重要的属性如下列表：

- clientId：（必须的）用来标识客户的Id。
- secret：（需要值得信任的客户端）客户端安全码，如果有的话。
- scope：用来限制客户端的访问范围，如果为空（默认）的话，那么客户端拥有全部的访问范围。
- authorizedGrantTypes：此客户端可以使用的授权类型，默认为空。
- authorities：此客户端可以使用的权限（基于Spring Security authorities）。

客户端详情（Client Details）能够在应用程序运行的时候进行更新，可以通过访问底层的存储服务（例如将客户端详情存储在一个关系数据库的表中，就可以使用 JdbcClientDetailsService）或者通过 ClientDetailsManager 接口（同时你也可以实现 ClientDetailsService 接口）来进行管理。

（译者注：不过我并没有找到 ClientDetailsManager 这个接口文件，只找到了 ClientDetailsService）

**管理令牌（Managing Token）：**

AuthorizationServerTokenServices 接口定义了一些操作使得你可以对令牌进行一些必要的管理，在使用这些操作的时候请注意以下几点：

- 当一个令牌被创建了，你必须对其进行保存，这样当一个客户端使用这个令牌对资源服务进行请求的时候才能够引用这个令牌。
- 当一个令牌是有效的时候，它可以被用来加载身份信息，里面包含了这个令牌的相关权限。

当你自己创建 AuthorizationServerTokenServices 这个接口的实现时，你可能需要考虑一下使用 DefaultTokenServices 这个类，里面包含了一些有用实现，你可以使用它来修改令牌的格式和令牌的存储。默认的，当它尝试创建一个令牌的时候，是使用随机值来进行填充的，除了持久化令牌是委托一个 TokenStore 接口来实现以外，这个类几乎帮你做了所有的事情。并且 TokenStore 这个接口有一个默认的实现，它就是 InMemoryTokenStore ，如其命名，所有的令牌是被保存在了内存中。除了使用这个类以外，你还可以使用一些其他的预定义实现，下面有几个版本，它们都实现了TokenStore接口：

- InMemoryTokenStore：这个版本的实现是被默认采用的，它可以完美的工作在单服务器上（即访问并发量压力不大的情况下，并且它在失败的时候不会进行备份），大多数的项目都可以使用这个版本的实现来进行尝试，你可以在开发的时候使用它来进行管理，因为不会被保存到磁盘中，所以更易于调试。
- JdbcTokenStore：这是一个基于JDBC的实现版本，令牌会被保存进关系型数据库。使用这个版本的实现时，你可以在不同的服务器之间共享令牌信息，使用这个版本的时候请注意把"spring-jdbc"这个依赖加入到你的classpath当中。
- JwtTokenStore：这个版本的全称是 JSON Web Token（JWT），它可以把令牌相关的数据进行编码（因此对于后端服务来说，它不需要进行存储，这将是一个重大优势），但是它有一个缺点，那就是撤销一个已经授权令牌将会非常困难，所以它通常用来处理一个生命周期较短的令牌以及撤销刷新令牌（refresh_token）。另外一个缺点就是这个令牌占用的空间会比较大，如果你加入了比较多用户凭证信息。JwtTokenStore 不会保存任何数据，但是它在转换令牌值以及授权信息方面与 DefaultTokenServices 所扮演的角色是一样的。

**JWT令牌（JWT Tokens）：**

使用JWT令牌你需要在授权服务中使用 JwtTokenStore，资源服务器也需要一个解码的Token令牌的类 JwtAccessTokenConverter，JwtTokenStore依赖这个类来进行编码以及解码，因此你的授权服务以及资源服务都需要使用这个转换类。Token令牌默认是有签名的，并且资源服务需要验证这个签名，因此呢，你需要使用一个对称的Key值，用来参与签名计算，这个Key值存在于授权服务以及资源服务之中。或者你可以使用非对称加密算法来对Token进行签名，Public Key公布在/oauth/token_key这个URL连接中，默认的访问安全规则是"denyAll()"，即在默认的情况下它是关闭的，你可以注入一个标准的 SpEL 表达式到 AuthorizationServerSecurityConfigurer 这个配置中来将它开启（例如使用"permitAll()"来开启可能比较合适，因为它是一个公共密钥）。

如果你要使用 JwtTokenStore，请务必把"spring-security-jwt"这个依赖加入到你的classpath中。

**配置授权类型（Grant Types）：**

授权是使用 AuthorizationEndpoint 这个端点来进行控制的，你能够使用 AuthorizationServerEndpointsConfigurer 这个对象的实例来进行配置(AuthorizationServerConfigurer 的一个回调配置项，见上的概述) ，如果你不进行设置的话，默认是除了资源所有者密码（password）授权类型以外，支持其余所有标准授权类型的（RFC6749），我们来看一下这个配置对象有哪些属性可以设置吧，如下列表：

- authenticationManager：认证管理器，当你选择了资源所有者密码（password）授权类型的时候，请设置这个属性注入一个 AuthenticationManager 对象。
- userDetailsService：如果啊，你设置了这个属性的话，那说明你有一个自己的 UserDetailsService 接口的实现，或者你可以把这个东西设置到全局域上面去（例如 GlobalAuthenticationManagerConfigurer 这个配置对象），当你设置了这个之后，那么 "refresh_token" 即刷新令牌授权类型模式的流程中就会包含一个检查，用来确保这个账号是否仍然有效，假如说你禁用了这个账户的话。
- authorizationCodeServices：这个属性是用来设置授权码服务的（即 AuthorizationCodeServices 的实例对象），主要用于 "authorization_code" 授权码类型模式。
- implicitGrantService：这个属性用于设置隐式授权模式，用来管理隐式授权模式的状态。
- tokenGranter：这个属性就很牛B了，当你设置了这个东西（即 TokenGranter 接口实现），那么授权将会交由你来完全掌控，并且会忽略掉上面的这几个属性，这个属性一般是用作拓展用途的，即标准的四种授权模式已经满足不了你的需求的时候，才会考虑使用这个。

在XML配置中呢，你可以使用 "authorization-server" 这个标签元素来进行设置。

**配置授权端点的URL（Endpoint URLs）：**

AuthorizationServerEndpointsConfigurer 这个配置对象(AuthorizationServerConfigurer 的一个回调配置项，见上的概述) 有一个叫做 pathMapping() 的方法用来配置端点URL链接，它有两个参数：

- 第一个参数：String 类型的，这个端点URL的默认链接。
- 第二个参数：String 类型的，你要进行替代的URL链接。

以上的参数都将以 "/" 字符为开始的字符串，框架的默认URL链接如下列表，可以作为这个 pathMapping() 方法的第一个参数：

- /oauth/authorize：授权端点。
- /oauth/token：令牌端点。
- /oauth/confirm_access：用户确认授权提交端点。
- /oauth/error：授权服务错误信息端点。
- /oauth/check_token：用于资源服务访问的令牌解析端点。
- /oauth/token_key：提供公有密匙的端点，如果你使用JWT令牌的话。

需要注意的是授权端点这个URL应该被Spring Security保护起来只供授权用户访问，我们来看看在标准的Spring Security中 WebSecurityConfigurer 是怎么用的。
