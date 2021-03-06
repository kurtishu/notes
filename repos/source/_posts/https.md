---
title: 聊聊HTTPS
date: 2016-03-16
categories: Security
tags: [HTTPS , SSL, 对称加密, 非对称加密, CA 证书]
---

###  什么是HTTPS  
在说HTTPS之前先说说什么是HTTP，HTTP就是我们平时浏览网页时候使用的一种协议。HTTP协议传输的数据都是未加密的，也就是明文的，因此使用HTTP协议传输隐私信息非常不安全。为了保证这些隐私数据能加密传输，于是网景公司设计了SSL（Secure Sockets Layer）协议用于对HTTP协议传输的数据进行加密，从而就诞生了HTTPS。
HTTPS（全称：Hyper Text Transfer Protocol over Secure Socket Layer），是以安全为目标的HTTP通道，简单讲是HTTP的安全版。 实际上我们现在的HTTPS都是用的TLS协议，TLS（"Transport Layer Security"），中文叫做"传输层安全协议" 或者"传输层加密协议", 它是IETF将SSL标准化之后更改的名字。SSL/TLS 这两者可以视作同一个东西的不同阶段。

###  HTTPS的工作原理
HTTPS在传输数据之前需要客户端（浏览器）与服务端（网站）之间进行一次握手，在握手过程中将确立双方加密传输数据的密码信息。TLS/SSL协议不仅仅是一套加密传输的协议,使用了非对称加密，它使用了对称加密以及HASH算法。    
** 握手过程的简单描述如下： **   
![](http://ww2.sinaimg.cn/mw690/6941baebjw1erta80zxylj20l80f0q44.jpg)
①客户端的浏览器向服务器传送客户端SSL 协议的版本号，加密算法的种类，产生的随机数，以及其他服务器和客户端之间通讯所需要的各种信息。    
②服务器向客户端传送SSL 协议的版本号，加密算法的种类，随机数以及其他相关信息，同时服务器还将向客户端传送自己的证书。  
③客户利用服务器传过来的信息验证服务器的合法性，服务器的合法性包括：证书是否过期，发行服务器证书的CA 是否可靠，发行者证书的公钥能否正确解开服务器证书的“发行者的数字签名”，服务器证书上的域名是否和服务器的实际域名相匹配。如果合法性验证没有通过，通讯将断开；如果合法性验证通过，将继续进行第四步。   
④用户端随机产生一个用于后面通讯的“对称密码”，然后用服务器的公钥（服务器的公钥从步骤②中的服务器的证书中获得）对其加密，然后将加密后的“预主密码”传给服务器。  
⑤如果服务器要求客户的身份认证（在握手过程中为可选），用户可以建立一个随机数然后对其进行数据签名，将这个含有签名的随机数和客户自己的证书以及加密过的“预主密码”一起传给服务器。  
⑥如果服务器要求客户的身份认证，服务器必须检验客户证书和签名随机数的合法性，具体的合法性验证过程包括：客户的证书使用日期是否有效，为客户提供证书的CA 是否可靠，发行CA 的公钥能否正确解开客户证书的发行CA 的数字签名，检查客户的证书是否在证书废止列表（CRL）中。检验如果没有通过，通讯立刻中断；如果验证通过，服务器将用自己的私钥解开加密的“预主密码”，然后执行一系列步骤来产生主通讯密码（客户端也将通过同样的方法产生相同的主通讯密码）。   
⑦服务器和客户端用相同的主密码即“通话密码”，一个对称密钥用于SSL 协议的安全数据通讯的加解密通讯。同时在SSL 通讯过程中还要完成数据通讯的完整性，防止数据通讯中的任何变化。  
⑧客户端向服务器端发出信息，指明后面的数据通讯将使用的步骤⑦中的主密码为对称密钥，同时通知服务器客户端的握手过程结束。  
⑨服务器向客户端发出信息，指明后面的数据通讯将使用的步骤⑦中的主密码为对称密钥，同时通知客户端服务器端的握手过程结束。  
⑩SSL 的握手部分结束，SSL 安全通道的数据通讯开始，客户和服务器开始使用相同的对称密钥进行数据通讯，同时进行通讯完整性的检验。   

###  ”对称加密“和“非对称加密”的概念
 #### ”对称加密“  
   所谓的“对称加密技术”，意思就是说：“加密”和“解密”使用【相同的】密钥。
  #### ”非对称加密“  
   所谓的“非对称加密技术”，意思就是说：“加密”和“解密”使用【不同的】密钥。 不同的钥匙指的是公钥和私钥，公钥是暴露给被人用的，比如我们从网站的证书中获取的就是公钥，私钥是私人持有，绝不对外公开的。
### 数字证书  
数字证书是一个经证书授权中心数字签名的包含公开密钥拥有者信息以及公开密钥的文件。
** 数字证书有两个作用 **
1. 身份授权。确保浏览器访问的网站是经过 CA 验证的可信任的网站。
2. 分发公钥。每个数字证书都包含了注册者生成的公钥。在 SSL 握手时会通过 certificate 消息传输给客户端。比如前文提到的 RSA 证书公钥加密及 ECDHE 的签名都是使用的这个公钥。  
CA是证书的签发机构，它是PKI的核心。CA是负责签发证书、认证证书、管理已颁发证书的机关。它要制定政策和具体步骤来验证、识别用户身份，并对用户证书进行签名，以确保证书持有者的身份和公钥的拥有权，CA是可以信任的第三方。
申请拿到CA签发的证书成为CA证书，还有就是开发者自己生成的证书，称为自签名证书，自签名证书在很多应用程序中用到。

### 数字签名
  ![](http://ww3.sinaimg.cn/mw690/6941baebjw1erta7ycyf8j215n0ridmj.jpg)

  ---
  ####     链接   
  [HTTPS通信中的身份认证机制](https://segmentfault.com/a/1190000004631778)
  [详解https是如何确保安全的？](http://www.wxtlife.com/2016/03/27/%E8%AF%A6%E8%A7%A3https%E6%98%AF%E5%A6%82%E4%BD%95%E7%A1%AE%E4%BF%9D%E5%AE%89%E5%85%A8%E7%9A%84%EF%BC%9F/)
  
<br/>
