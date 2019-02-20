---
title: HTTP和HTTPS
date: 2019-02-20 14:02:19
categories: 网络
tags: 
    - HTTP
    - HTTPS
    - TCP
    - RSA加密
    - DH加密
---

<img src="https://www.techalyst.com/content/uploads/editor-photos/blog/1ecaab6c899ab15bc9aabf9533060047d17ddbf3.png" class="full-image" />
HTTP和HTTPS属于计算机网络范畴，但作为开发人员，不管是后台开发或是前台开发，都很有必要掌握它们。
<!-- more -->

## 网络层结构
网络结构有两种主流的分层方式：**OSI七层模型和TCP/IP四层模型。**

OSI是指Open System Interconnect，意为开放式系统互联。

TCP/IP是指传输控制协议/网间协议，是目前世界上应用最广的协议

![OSI七层协议](http://image.lkd-ykr.top/OSI%E4%B8%83%E5%B1%82%E6%A8%A1%E5%9E%8B.jpeg)

### 两种模型区别
1. OSI采用七层模型，TCP/IP是四层模型
2. TCP/IP网络接口层没有真正的定义，只是概念性的描述。OSI把它分为2层，每一层功能详尽。
3. 在协议开发之前，就有了OSI模型，所以OSI模型具有共通性，而TCP/IP是基于协议建立的模型，不适用于非TCP/IP的网络。
4. 实际应用中，OSI模型是理论上的模型，没有成熟的产品；而TCP/IP已经成为国际标准。

## HTTP协议
HTTP是基于TCP/IP协议的**应用程序协议**，不包括数据包的传输，主要规定了客户端和服务器的通信格式，默认使用80端口。
### HTTP协议的发展历史
1. 1991年发布HTTP/0.9版本，只有Get命令，且服务端直返HTML格式字符串，服务器响应完毕就关闭TCP连接。
2. 1996年发布HTTP/1.0版本，优点：可以发送任何格式内容，包括文字、图像、视频、二进制。也丰富了命令Get，Post，Head。请求和响应的格式加入头信息。缺点：每个TCP连接只能发送一个请求，而新建TCP连接的成本很高，导致HTTP/1.0新能很差。
3. 1997发布HTTP/1.1版本，完善了HTTP协议，直至20年后的今天仍是最流行的版本。
- 优点：
   - a. 引入持久连接，TCP默认不关闭，可被多个请求复用，对于一个域名，多数浏览器允许同时建立6个持久连接。
   - b. 引入管道机制，即在同一个TCP连接中，可以同时发送多个请求，不过服务器还是按顺序响应。
   - c. 在头部加入Content-Length字段，一个TCP可以同时传送多个响应，所以就需要该字段来区分哪些内容属于哪个响应。
   - d. 分块传输编码，对于耗时的动态操作，用流模式取代缓存模式，即产生一块数据，就发送一块数据。
   - e. 增加了许多命令，头信息增加Host来指定服务器域名，可以访问一台服务器上的不同网站。
- 缺点：TCP连接中的响应有顺序，服务器处理完一个回应才能处理下一个回应，如果某个回应特别慢，后面的请求就会排队等着（对头堵塞）。
4. 2015年发布HTTP/2版本，它有几个特性：二进制协议、多工、数据流、头信息压缩、服务器推送。

### HTTP请求和响应格式
Request格式：

```
GET /barite/account/stock/groups HTTP/1.1
QUARTZ-SESSION: MC4xMDQ0NjA3NTI0Mzc0MjAyNg.VPXuA8rxTghcZlRCfiAwZlAIdCA
DEVICE-TYPE: ANDROID
API-VERSION: 15
Host: shitouji.bluestonehk.com
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.10.0
```

Response格式：

```
HTTP/1.1 200 OK
Server: nginx/1.6.3
Date: Mon, 15 Oct 2018 03:30:28 GMT
Content-Type: application/json;charset=UTF-8
Pragma: no-cache
Cache-Control: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Content-Encoding: gzip
Transfer-Encoding: chunked
Proxy-Connection: Keep-alive

{"errno":0,"dialogInfo":null,"body":{"list":[{"flag":2,"group_id":1557,"group_name":"港股","count":1},{"flag":3,"group_id":1558,"group_name":"美股","count":7},{"flag":1,"group_id":1556,"group_name":"全部","count":8}]},"message":"success"}
```

说明一下请求头和响应头的部分字段：
- `Host`：指定服务器域名，可用来区分访问一个服务器上的不同服务
- `Connection`：keep-alive表示要求服务器不要关闭TCP连接，close表示明确要求关闭连接，默认值是keep-alive
- `Accept-Encoding`：说明自己可以接收的压缩方式
- `User-Agent`：用户代理，是服务器能识别客户端的操作系统（Android、IOS、WEB）及相关的信息。作用是帮助服务器区分客户端，并且针对不同客户端让用户看到不同数据，做不同操作。
- `Content-Type`：服务器告诉客户端数据的格式，常见的值有text/plain，image/jpeg，image/png，video/mp4，application/json，application/zip。这些数据类型总称为MIME TYPE。
- `Content-Encoding`：服务器数据压缩方式
- `Transfer-Encoding`：chunked表示采用分块传输编码，有该字段则无需使用Content-Length字段。
- `Content-Length`：声明数据的长度，请求和回应头部都可以使用该字段。

## TCP三次握手
HTTP和HTTPS协议请求时都会通过**TCP三次握手建立TCP连接**。那么，三次握手是指什么呢？

![TCP三次握手](http://image.lkd-ykr.top/tcp%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B.jpeg)

**那么，为什么一定要三次握手呢，一次可以吗？两次可以吗？**

带着这些问题，我们来分析一下为什么必须是三次握手。

1. 第一次握手，A向B发送信息后，B收到信息。B可确认A的发信能力和B的收信能力
2. 第二次握手，B向A发消息，A收到消息。A可确认A的发信能力和收信能力，A也可确认B的收信能力和发信能力
3. 第三次握手，A向B发送消息，B接收到消息。B可确认A的收信能力和B的发信能力

通过三次握手，A和B都能确认自己和对方的收发信能力，相当于建立了互相的信任，就可以开始通信了。

下面，我们介绍一下三次握手具体发送的内容，用一张图描述如下：

![三次握手](http://image.lkd-ykr.top/%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B.jpeg)

首先，介绍一下几个概念：

- `ACK`：响应标识，1表示响应，连接建立成功之后，所有报文段ACK的值都为1
- `SYN`：连接标识，1表示建立连接，连接请求和连接接受报文段- SYN=1，其他情况都是0
- `FIN`：关闭连接标识，1标识关闭连接，关闭请求和关闭接受报文段- FIN=1，其他情况都是0，跟SYN类似
- `seq number`：序号，一个随机数X，请求报文段中会有该字段，响应报文段没有
- `ack number`：应答号，值为请求seq+1，即X+1，除了连接请求和连接接受响应报文段没有该字段，其他的报文段都有该字段

知道了上面几个概念后，看一下三次握手的具体流程：

1. **第一次握手**：建立连接请求。客户端发送连接请求报文段，将SYN置为1，seq为随机数x。然后，客户端进入SYN_SEND状态，等待服务器确认。
2. **第二次握手**：确认连接请求。服务器收到客户端的SYN报文段，需要对该请求进行确认，设置ack=x+1（即客户端seq+1）。同时自己也要发送SYN请求信息，即SYN置为1，seq=y。服务器将SYN和ACK信息放在一个报文段中，一并发送给客户端，服务器进入SYN_RECV状态。
3. **第三次握手**：客户端收到SYN+ACK报文段，将ack设置为y+1，向服务器发送ACK报文段，这个报文段发送完毕，客户端和服务券进入ESTABLISHED状态，完成TCP三次握手。

从图中可以看出，建立连接经历了三次握手，当数据传输完毕，需要断开连接，而**断开连接经历了四次挥手**：

1. **第一次挥**手：主机1（可以是客户端或服务器），设置seq和ack向主机2发送一个FIN报文段，此时主机1进入FIN_WAIT_1状态，表示没有数据要发送给主机2了
2. **第二次挥手**：主机2收到主机1的FIN报文段，向主机1回应一个ACK报文段，表示同意关闭请求，主机1进入FIN_WAIT_2状态。
3. **第三次挥手**：主机2向主机1发送FIN报文段，请求关闭连接，主机2进入LAST_ACK状态。
4. **第四次挥手**：主机1收到主机2的FIN报文段，想主机2回应ACK报文段，然后主机1进入TIME_WAIT状态；主机2收到主机1的ACK报文段后，关闭连接。此时主机1等待主机2一段时间后，没有收到回复，证明主机2已经正常关闭，主机1页关闭连接。

下面是**TCP报文段首部**格式图，对于理解TCP协议很重要：

![TCP报文段首部](http://image.lkd-ykr.top/tcp%E6%8A%A5%E6%96%87%E9%A6%96%E9%83%A8.jpeg)

## HTTPS协议/SSL协议

HTTPS协议是以安全为目标的HTTP通道，简单来说就是HTTP的安全版。主要是**在HTTP下加入SSL层（现在主流的是SLL/TLS）**，SSL是HTTPS协议的安全基础。HTTPS默认端口号为443。

### HTTP存在的风险

1. **窃听风险**：HTTP采用明文传输数据，第三方可以获知通信内容
2. **篡改风险**：第三方可以修改通信内容
3. **冒充风险**：第三方可以冒充他人身份进行通信

SSL/TLS协议就是为了解决这些风险而设计，希望达到：

1. 所有信息加密传输，三方窃听通信内容
2. 具有校验机制，内容一旦被篡改，通信双发立刻会发现
3. 配备身份证书，防止身份被冒充

下面主要介绍SSL/TLS协议。

### SSL发展史（互联网加密通信）

1. 1994年NetSpace公司设计SSL协议（Secure Sockets Layout）1.0版本，但未发布。
2. 1995年NetSpace发布SSL/2.0版本，很快发现有严重漏洞
3. 1996年发布SSL/3.0版本，得到大规模应用
4. 1999年，发布了SSL升级版TLS/1.0版本，目前应用最广泛的版本
5. 2006年和2008年，发布了TLS/1.1版本和TLS/1.2版本

### SSL原理及运行过程

SSL/TLS协议基本思路是采用**公钥加密法**（最有名的是**RSA加密算法**）。大概流程是，**客户端向服务器索要公钥，然后用公钥加密信息，服务器收到密文，用自己的私钥解密**。

为了防止公钥被篡改，把公钥放在数字证书中，证书可信则公钥可信。公钥加密计算量很大，为了提高效率，服务端和客户端都生成对话秘钥，用它加密信息，而对话秘钥是对称加密，速度非常快。而公钥用来机密对话秘钥。

下面用一张图表示**SSL加密传输过程**：

![SSL加密传输过程](https://www.google.com/url?sa=i&source=images&cd=&cad=rja&uact=8&ved=2ahUKEwj55MXP4MngAhUL5oMKHbE6BZkQjRx6BAgBEAU&url=%2Furl%3Fsa%3Di%26source%3Dimages%26cd%3D%26ved%3D%26url%3Dhttp%253A%252F%252Fwww.ruanyifeng.com%252Fblog%252F2014%252F09%252Fillustration-ssl.html%26psig%3DAOvVaw33EPMiWfIQcIHROr2xlzUq%26ust%3D1550732941970451&psig=AOvVaw33EPMiWfIQcIHROr2xlzUq&ust=1550732941970451)

详细介绍一下图中过程：

1、客户端给出协议版本号、一个客户端随机数A（Client random）以及客户端支持的加密方式
2、服务端确认双方使用的加密方式，并给出数字证书、一个服务器生成的随机数B（Server random）
3、客户端确认数字证书有效，生成一个新的随机数C（Pre-master-secret），使用证书中的公钥对C加密，发送给服务端
4、服务端使用自己的私钥解密出C
5、客户端和服务器根据约定的加密方法，使用三个随机数ABC，生成对话秘钥，之后的通信都用这个对话秘钥进行加密。

### SSL证书

上面提到了，HTTPS协议中需要使用到SSL证书。

SSL证书是一个二进制文件，里面包含经过认证的网站公钥和一些元数据，需要从经销商购买。
证书有很多类型，按认证级别分类：

- **域名认证（DV=Domain Validation）**：最低级别的认证，可以确认申请人拥有这个域名
- **公司认证（OV=Organization Validation）**：确认域名所有人是哪家公司，证书里面包含公司的信息
- **扩展认证（EV=Extended Validation）**：最高级别认证，浏览器地址栏会显示公司名称。

按覆盖范围分类：

- 单域名证书：只能用于单域名，foo.com证书不能用不www.foo.com
- 通配符证书：可用于某个域名及所有一级子域名，比如*.foo.com的证书可用于foo.com，也可用于www.foo.com
- 多域名证书：可用于多个域名，比如foo.com和bar.com

认证级别越高，覆盖范围越广的证书，价格越贵。也有免费的证书，为了推广HTTPS，电子前哨基金会成立了Let's Encrypt提供免费证书。

证书的经销商也很多，知名度比较高的有亚洲诚信(Trust Asia)。

## RSA加密和DH加密

### 加密算法分类

加密算法分为**对称加密**、**非对称加密**和**Hash加密算法**。

- **对称加密**：甲方和乙方使用同一种加密规则对信息加解密
- **非对称加密**：乙方生成两把秘钥（公钥和私钥）。公钥是公开的，任何人都可以获取，私钥是保密的，只存在于乙方手中。甲方获取公钥，然后用公钥加密信息，乙方得到密文后，用私钥解密。
- **Hash加**密：Hash算法是一种单向密码体制，即只有加密过程，没有解密过程

对称加密算法加解密效率高，速度快，适合大数据量加解密。常见的堆成加密算法有DES、AES、RC5、Blowfish、IDEA

非对称加密算法复杂，加解密速度慢，但安全性高，一般与对称加密结合使用（对称加密通信内容，非对称加密对称秘钥）。

常见的非对称加密算法有`RSA`、`DH`、`DSA`、`ECC`

Hash算法特性是：输入值一样，经过哈希函数得到相同的散列值，但并非散列值相同则输入值也相同。常见的Hash加密算法有MD5、SHA-1、SHA-X系列

下面着重介绍一下RSA算法和DH算法。

#### RSA加密算法

HTTPS协议就是使用RSA加密算法，可以说RSA加密算法是宇宙中最重要的加密算法。

RSA算法用到一些数论知识，包括互质关系，欧拉函数，欧拉定理。此处不具体介绍加密的过程，如果有兴趣，可以参照[RSA算法加密过程]
(http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)。

RSA算法的安全保障基于大数分解问题，目前破解过的最大秘钥是700+位，也就代表1024位秘钥和2048位秘钥可以认为绝对安全。

大数分解主要难点在于计算能力，如果未来计算能力有了质的提升，那么这些秘钥也是有可能被破解的。

#### DH加密算法

DH也是一种非对称加密算法，[DH加密算法过程](https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B)。

DH算法的安全保障是基于离散对数问题。

## HTTP协议和HTTPS协议的对比

HTTP和HTTPS的**区别**如下：

1. HTTPS协议需要到CA申请证书，大多数情况下需要一定费用
2. HTTP是超文本传输协议，信息采用明文传输，HTTPS则是具有安全性SSL加密传输协议
3. HTTP和HTTPS端口号不一样，HTTP是80端口，HTTPS是443端口
4. HTTP连接是无状态的，而HTTPS采用HTTP+SSL构建可进行加密传输、身份认证的网络协议，更安全。
5. HTTP协议建立连接的过程比HTTPS协议快。因为HTTPS除了TCP三次握手，还要经过SSL握手。连接建立之后数据传输速度，二者无明显区别。

> 本文转发自
> 作者：左大人
> 链接：https://www.jianshu.com/p/27862635c077