---
title: 理解session、cookie、token
date: 2018-12-14 09:57:25
tags:
---
之前一直不太理解这几个概念，总听别人提起，所以查了一些资料学习一下，在此做个总结和记录，以备不时之需。。。

很久很久以前，Web 基本上就是文档的浏览而已，既然是浏览，作为服务器，不需要记录谁在某一段时间里都浏览了什么文档，每次请求都是一个新的HTTP协议，就是请求加响应，尤其是我不用记住是谁刚刚发了HTTP请求，每个请求对我来说都是全新的。这段时间很嗨皮。

但是随着交互式Web应用的兴起，像在线购物网站，需要登录的网站等等，马上就面临一个问题，那就是要管理会话，必须记住哪些人登录系统，哪些人往自己的购物车中放商品，也就是说我必须把每个人区分开，这就是一个不小的挑战，因为HTTP请求是无状态的，所以想出的办法就是给大家发一个会话标识(session id)， 说白了就是一个随机的字串，每个人收到的都不一样，每次大家向我发起HTTP请求的时候，把这个字符串给一并捎过来，这样我就能区分开谁是谁了。

这样大家很嗨皮了，可是服务器就不嗨皮了，每个人只需要保存自己的session id，而服务器要保存所有人的session id！如果访问服务器多了，就得有成千上万，甚至几十万个。

这对服务器说是一个巨大的开销，严重的限制了服务器扩展能力，比如说我用两个机器组成了一个集群，小F通过机器A登录了系统，那session id会保存在机器A上，  假设小F的下一次请求被转发到机器B怎么办？机器B可没有小F的session id啊。

有时候会采用一点小伎俩：session sticky，就是让小F的请求一直粘连在机器A上，但是这也不管用，要是机器A挂掉了，还得转到机器B去。

那只好做session的复制了，把session id在两个机器之间搬来搬去，快累死了。

![](理解session、cookie、token/session、cookie、token_1.png)

后来有个叫Memcached的支了招：把session id集中存储到一个地方，所有的机器都来访问这个地方的数据，这样一来，就不用复制了，但是增加了单点失败的可能性， 要是那个负责session的机器挂了，所有人都得重新登录一遍，估计得被人骂死。

![](理解session、cookie、token/session、cookie、token_2.png)

也尝试把这个单点的机器也搞出集群，增加可靠性，但不管如何，这小小的session 对我来说是一个沉重的负担。

于是有人就一直在思考，我为什么要保存这可恶的session呢，只让每个客户端去保存该多好？可是如果不保存这些session id，怎么验证客户端发给我的session id的确是我生成的呢？如果不去验证，我们都不知道他们是不是合法登录的用户，那些不怀好意的家伙们就可以伪造session id，为所欲为了。

嗯，对了，关键点就是验证 ！

比如说，小F已经登录了系统，我给他发一个令牌(token)，里边包含了小F的user id，下一次小F再次通过Http请求访问我的时候，把这个token 通过Http header带过来不就可以了。

不过这和session id没有本质区别啊，任何人都可以可以伪造，所以我得想点儿办法，让别人伪造不了。

那就对数据做一个签名吧，比如说我用HMAC-SHA256算法，加上一个只有我才知道的密钥，对数据做一个签名，把这个签名和数据一起作为token，由于密钥别人不知道，就无法伪造token了。

![](理解session、cookie、token/session、cookie、token_3.png)

这个token我不保存，当小F把这个token给我发过来的时候，我再用同样的HMAC-SHA256算法和同样的密钥，对数据再计算一次签名，和token中的签名做个比较，如果相同，我就知道小F已经登录过了，并且可以直接取到小F的user id，如果不相同，数据部分肯定被人篡改过，我就告诉发送者：对不起，没有认证。

![](理解session、cookie、token/session、cookie、token_4.png)

Token中的数据是明文保存的（虽然我会用Base64做下编码，但那不是加密），还是可以被别人看到的，所以我不能在其中保存像密码这样的敏感信息。

当然，如果一个人的token被别人偷走了，那我也没办法，我也会认为小偷就是合法用户，这其实和一个人的session id被别人偷走是一样的。

这样一来，我就不保存session id了，我只是生成token，然后验证token，我用我的CPU计算时间获取了我的session存储空间！

解除了session id这个负担，可以说是无事一身轻，我的机器集群现在可以轻松地做水平扩展，用户访问量增大，直接加机器就行。这种无状态的感觉实在是太好了！

### Cookie

cookie 是一个非常具体的东西，指的就是浏览器里面能永久存储的一种数据，仅仅是浏览器实现的一种数据存储功能。

cookie由服务器生成，发送给浏览器，浏览器把cookie以kv形式保存到某个目录下的文本文件内，下一次请求同一网站时会把该cookie发送给服务器。由于cookie是存在客户端上的，所以浏览器加入了一些限制确保cookie不会被恶意使用，同时不会占据太多磁盘空间，所以每个域的cookie数量是有限的。

### Session

session 从字面上讲，就是会话。这个就类似于你和一个人交谈，你怎么知道当前和你交谈的是张三而不是李四呢？对方肯定有某种特征（长相等）表明他就是张三。

session 也是类似的道理，服务器要知道当前发请求给自己的是谁。为了做这种区分，服务器就要给每个客户端分配不同的“身份标识”，然后客户端每次向服务器发请求的时候，都带上这个“身份标识”，服务器就知道这个请求来自于谁了。至于客户端怎么保存这个“身份标识”，可以有很多种方式，对于浏览器客户端，大家都默认采用 cookie 的方式。

服务器使用session把用户的信息临时保存在了服务器上，用户离开网站后session会被销毁。这种用户信息存储方式相对cookie来说更安全，可是session有一个缺陷：如果web服务器做了负载均衡，那么下一个操作请求到了另一台服务器的时候session会丢失。

### Token

以下几点特性会让你在程序中使用基于Token的身份验证

1. 无状态、可扩展
2. 支持移动设备
3. 跨程序调用
4. 安全

    #### 传统身份验证的方法

    HTTP 是一种没有状态的协议，也就是它并不知道是谁是访问应用。这里我们把用户看成是客户端，客户端使用用户名还有密码通过了身份验证，不过下回这个客户端再发送请求时候，还得再验证一下。
    
    解决的方法就是，当用户请求登录的时候，如果没有问题，我们在服务端生成一条记录，这个记录里可以说明一下登录的用户是谁，然后把这条记录的 ID 号发送给客户端，客户端收到以后把这个 ID 号存储在 Cookie 里，下次这个用户再向服务端发送请求的时候，可以带着这个 Cookie ，这样服务端会验证一个这个 Cookie 里的信息，看看能不能在服务端这里找到对应的记录，如果可以，说明用户已经通过了身份验证，就把用户请求的数据返回给客户端。

    上面说的就是 Session，我们需要在服务端存储为登录的用户生成的 Session ，这些 Session 可能会存储在内存，磁盘，或者数据库里。我们可能需要在服务端定期的去清理过期的 Session 。

    #### 基于Token的验证原理

    基于Token的身份验证是无状态的，我们不将用户信息存在服务器或Session中。

    这种概念解决了在服务端存储信息时的许多问题

    NoSession意味着你的程序可以根据需要去增减机器，而不用去担心用户是否登录。

    基于Token的身份验证的过程如下:

    1. 用户通过用户名和密码发送请求。
    2. 程序验证。
    3. 程序返回一个签名的token 给客户端。
    4. 客户端储存token,并且每次用于每次发送请求。
    5. 服务端验证token并返回数据。

    实现思路：

    ![](理解session、cookie、token/session、cookie、token_5.png)

    1. 用户登录校验，校验成功后就返回Token给客户端。
    2. 客户端收到数据后保存在客户端
    3. 客户端每次访问API是携带Token到服务器端。
    4. 服务器端采用filter过滤器校验。校验成功则返回请求数据，校验失败则返回错误码

### cookie和session的区别

1. cookie数据存放在客户的浏览器上，session数据放在服务器上。
2. cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗
   考虑到安全应当使用session。
3. session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能

   考虑到减轻服务器性能方面，应当使用COOKIE。
4. 单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。
5. 所以个人建议：
   
   将登陆信息等重要信息存放为SESSION
   
   其他信息如果需要保留，可以放在COOKIE中

### token和session的区别

token就是令牌，比如你授权（登录）一个程序时，他就是个依据，判断你是否已经授权该软件；cookie就是写在客户端的一个txt文件，里面包括你登录信息之类的，这样你下次在登录某个网站，就会自动调用cookie自动登录用户名；session和cookie差不多，只是session是写在服务器端的文件，也需要在客户端写入cookie文件，但是文件里是你的浏览器编号。Session的状态是存储在服务器端，客户端只有session id；而Token的状态是存储在客户端。

### JWT

实施 Token 验证的方法挺多的，还有一些标准方法，比如 JWT，读作：jot ，表示：JSON Web Tokens 。JWT 标准的 Token 有三个部分：

- header
- payload
- signature

中间用点分隔开，并且都会使用 Base64 编码，所以真正的 Token 看起来像这样：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJuaW5naGFvLm5ldCIsImV4cCI6IjE0Mzg5NTU0NDUiLCJuYW1lIjoid2FuZ2hhbyIsImFkbWluIjp0cnVlfQ.SwyHTEx_RQppr97g4J5lKXtabJecpejuef8AqKYMAJc
```

#### Header

header 部分主要是两部分内容，一个是 Token 的类型，另一个是使用的算法，比如下面类型就是 JWT，使用的算法是 HS256。

```
{
  "typ": "JWT",
  "alg": "HS256"
}
```

上面的内容要用 Base64 的形式编码一下，所以就变成这样：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

#### Payload

Payload 里面是 Token 的具体内容，这些内容里面有一些是标准字段，你也可以添加其它需要的内容。下面是标准字段：
- iss：Issuer，发行者
- sub：Subject，主题
- aud：Audience，观众
- exp：Expiration time，过期时间
- nbf：Not before
- iat：Issued at，发行时间
- jti：JWT ID

比如下面这个 Payload ，用到了 iss 发行人，还有 exp 过期时间。另外还有两个自定义的字段，一个是 name ，还有一个是 admin 。

```
{
 "iss": "ninghao.net",
 "exp": "1438955445",
 "name": "wanghao",
 "admin": true
}
```

使用 Base64 编码以后就变成了这个样子：

```
eyJpc3MiOiJuaW5naGFvLm5ldCIsImV4cCI6IjE0Mzg5NTU0NDUiLCJuYW1lIjoid2FuZ2hhbyIsImFkbWluIjp0cnVlfQ
```

#### Signature

JWT 的最后一部分是 Signature ，这部分内容有三个部分，先是用 Base64 编码的 header.payload ，再用加密算法加密一下，加密的时候要放进去一个 Secret ，这个相当于是一个密码，这个密码秘密地存储在服务端。

- header
- payload
- secret

```
var encodedString = base64UrlEncode(header) + "." + base64UrlEncode(payload); 
HMACSHA256(encodedString, 'secret');
```

处理完成以后看起来像这样：

```
SwyHTEx_RQppr97g4J5lKXtabJecpejuef8AqKYMAJc
```

最后这个在服务端生成并且要发送给客户端的 Token 看起来像这样：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJuaW5naGFvLm5ldCIsImV4cCI6IjE0Mzg5NTU0NDUiLCJuYW1lIjoid2FuZ2hhbyIsImFkbWluIjp0cnVlfQ.SwyHTEx_RQppr97g4J5lKXtabJecpejuef8AqKYMAJc
```

客户端收到这个 Token 以后把它存储下来，下回向服务端发送请求的时候就带着这个 Token 。服务端收到这个 Token ，然后进行验证，通过以后就会返回给客户端想要的资源。