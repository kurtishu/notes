---
title: DNS 相关知识
date: 2017-06-13 15:09:21
categories: [其他, DNS]
tags: [DNS, SRV, Zone file, CNAME]
---

### DNS 简介
DNS（Domain Name Server，域名服务器）是进行域名(domain name)和与之相对应的IP地址 (IP address)转换的服务器。域名是Internet上某一台计算机或计算机组的名称。但是计算机在互联网中的实际"住址"是IP，如何通过域名找到对应的计算机，这个过程就是通过DNS服务器来实现的，通过域名查找计算机的IP地址的过程叫做“域名解析”。

![DNS](http://up.2cto.com/2012/1018/20121018014622645.jpg)

### DNS Zone file

DNS zone file是一个文本文件，它包含了域名和对应的IP地址的映射关系，以及一些其他的资源记录。
> A Domain Name System (DNS) zone file is a text file that describes a DNS zone. A DNS zone is a subset, often a single domain, of the hierarchical domain name structure of the DNS. The zone file contains mappings between domain names and IP addresses and other resources, organized in the form of text representations of resource records (RR). A zone file may be either a DNS master file, authoritatively describing a zone, or it may be used to list the contents of a DNS cache. 



#### Zone file 格式

Zone 文件是有许许多多的资源记录(resource records)组成的，文件的每一行描述了对一种资源记录的定义，它由多部分组成，没部分通过空格或者Tab分割
具体格式如下：
 **  name	  ttl	 record class	 record type	 record data  **
 * name:  Record 名称，可为空，如果为空，前一个record的名称一样
 * ttl: (time to live) DNS缓存时间
 * record class: 表示记录的命令空间，通常为IN或者CHAOS
 * record type: 记录的类型，例如SRV，A记录， CNAME等
 * record data：根据记录类型，可能包含一个或多个元素。例如地址记录仅仅包含IP地址，如果是邮件交换记录则包含优先级(priority )、域名等。

下面以domain 为 example.com 为例：
```java
$ORIGIN example.com.     ; designates the start of this zone file in the namespace
$TTL 1h                  ; default expiration time of all resource records without their own TTL value
example.com.  IN  SOA   ns.example.com. username.example.com. ( 2007120710 1d 2h 4w 1h )
example.com.  IN  NS    ns                    ; ns.example.com is a nameserver for example.com
example.com.  IN  NS    ns.somewhere.example. ; ns.somewhere.example is a backup nameserver for example.com
example.com.  IN  MX    10 mail.example.com.  ; mail.example.com is the mailserver for example.com
@             IN  MX    20 mail2.example.com. ; equivalent to above line, "@" represents zone origin
@             IN  MX    50 mail3              ; equivalent to above line, but using a relative host name
example.com.  IN  A     192.0.2.1             ; IPv4 address for example.com
              IN  AAAA  2001:db8:10::1        ; IPv6 address for example.com
ns            IN  A     192.0.2.2             ; IPv4 address for ns.example.com
              IN  AAAA  2001:db8:10::2        ; IPv6 address for ns.example.com
www           IN  CNAME example.com.          ; www.example.com is an alias for example.com
wwwtest       IN  CNAME www                   ; wwwtest.example.com is another alias for www.example.com
mail          IN  A     192.0.2.3             ; IPv4 address for mail.example.com
mail2         IN  A     192.0.2.4             ; IPv4 address for mail2.example.com
mail3         IN  A     192.0.2.5             ; IPv4 address for mail3.example.com
```
### SRV record
SRV 记录在DNS域名系统中比较特殊的一种数据，它包含域名、端口号等信息。它一般为一些特定的服务器提供服务，还有一些互联网协议比如Session Initiation Protocol (SIP)以及 Extensible Messaging and Presence Protocol (XMPP)等都需要SRV记录。

#### SRV记录的格式
```java
_service._proto.name. TTL class SRV priority weight port target.
```
* service: 服务器的名称
* proto： 传输的端口号
* name： 合法的域名并以.结尾
* TTL：缓存有效时间
* class：标准DNS类别，一般为IN
* priority: m目标域名的优先级，数值越低优先级越高
* weight：权重和priority类似，权重越高优先级越高
* port：TCP/UDP的端口号
* target：主机的别名，并且以.结尾

举个例子：
```java
# _service._proto.name.  TTL   class SRV priority weight port target.
_sip._tcp.example.com.   86400 IN    SRV 10       60     5060 bigbox.example.com.
_sip._tcp.example.com.   86400 IN    SRV 10       20     5060 smallbox1.example.com.
_sip._tcp.example.com.   86400 IN    SRV 10       20     5060 smallbox2.example.com.
_sip._tcp.example.com.   86400 IN    SRV 20       0      5060 backupbox.example.com.
```

#### 查看SRV记录
```java
$ dig _sip._tcp.example.com SRV

$ host -t SRV _sip._tcp.example.com

$ nslookup -querytype=srv _sip._tcp.example.com

$ nslookup
> set querytype=srv
> _sip._tcp.example.com
```

### CNAME Record
经常在购买域名或者注册域名的时候经常听说CNAME，所谓CNAME记录，及别名记录。这种记录允许您将多个名字映射到同一台计算机。 通常用于同时提供WWW和MAIL服务的计算机。例如，有一台计算机名为“host.mydomain.com”（A记录）。 它同时提供WWW和MAIL服务，为了便于用户访问服务。可以为该计算机设置两个别名（CNAME）：WWW和MAIL。

例如：
```java
NAME                    TYPE   VALUE
--------------------------------------------------
bar.example.com.        CNAME  foo.example.com.
foo.example.com.        A      192.0.2.23
```
以上例子第一行的Type为CNAME，也就是说该行是别名记录（CNAME Record）, bar.example.com是域名foo.example.com的别名，通过别名访问域名，域名通过Type为A的记录，找到对应的IP地址。
上面提到 的Type为A的记录，这就是所谓的A记录。

### A记录
A记录记录的是一个32为的IPv4的IP地址，他通常被用记录域名与IP地址之间的映射关系。比如上面提到的，域名foo.example.com所映射的IP地址为192.0.2.23
DNS 记录有很多种类型，每种类型都有它特定的含义和作用。
[**List of DNS record types **](https://en.wikipedia.org/wiki/List_of_DNS_record_types) 

### 代码操作DNS Record
在项目开发中，有时候需求获取或者更新DNS 记录，这时候可以通过一个Java库dnsjava来实现。dnsjava是DNS的一个Java实现。支持所有定义的记录类型包括DNSSEC类型，和未知类型。它还可以用于查询，区域传输，动态更新。dnsjava还包含一个客户端使用的缓存，一个小型DNS服务器。

#### Java代码获取Srv Record
首先需要导入dnsJava的jar包，此处[下载 dnsjava](http://www.dnsjava.org/download/)。
Android 添加如下依赖：
```java
   compile "dnsjava:dnsjava:2.1.6"
```
具体代码实现：

```java
public class DNSLookup {

        public static void main(String[] args) {

          String query = "_sip._tcp.example.com" ;

          try {
            /*****HERE*********/
            Record[] records = new Lookup(query, Type.SRV).run();
            for (Record record : records) {
                 SRVRecord srv = (SRVRecord) record;
            String hostname = srv.getTarget().toString().replaceFirst("\\.$", "");
            int port = srv.getPort();
            System.out.println(hostname + ":" + port);
          }
          } catch (TextParseException e) {
             e.printStackTrace();
          }
       }
      }
```

### 总结
以上关于DNS的介绍仅仅只是设计到部分知识，后续或不断更新，如果以上有错误的地方还望提出指正。

### 参考
> [Domain Name System](https://en.wikipedia.org/wiki/Domain_Name_System)
> [SRV record](https://en.wikipedia.org/wiki/SRV_record)
> [Zone file](https://en.wikipedia.org/wiki/Zone_file)
> [DNS扫盲系列](http://www.2cto.com/net/201210/161839.html)