---
layout:     post
title:      "聊聊HTTPS"
subtitle:   " \"提高http安全性，发展趋势\""
date:       2017-08-04 10:51
header-img: "img/post-bg-unix-linux.jpg"
author:     "pepperliu"
catalog:      true
tags:
    - https
    - SSL/TLS
---

### 1. 先来聊聊SSL/TLS

- SSL全称"Secure Socket Layer"，翻译过来就是“安全套接字”。它是在20世纪90年代中期由王经公司设计的。因为原本的HTTP是明文的，其传输内容易被偷窥和篡改。发明SSL协议就是解决此类问题
- TLS全称"Transport Layer Security"，中文名称叫做“传输层安全协议”。TLS是IETF就SSL再次标准化的产物。

经常有人将两者并称（SSL/TLS），这两者可以视作同一个东西的不同阶段

### 2. HTTP存在的隐患

1. 无法确定请求发送至目标的 Web 服务器是否是按真实意图返回响应的那台服务器。有可能是已伪装的 Web 服务器
2. 无法确定响应返回到的客户端是否是按真实意图接收响应的那个客户端。有可能是已伪装的客户端
3. 无法确定正在通信的对方是否具备访问权限。因为某些Web 服务器上保存着重要的信息， 只想发给特定用户通信的权限
4. 无法判定请求是来自何方、出自谁手
5. 即使是无意义的请求也会照单全收。无法阻止海量请求下的DoS 攻击（ Denial of Service， 拒绝服务攻击）

### 3. 查明对方的证书

虽然使用 HTTP 协议无法确定通信方，但如果使用 SSL 则可以。SSL 不仅提供加密处理，而且还使用了一种被称为证书的手段，可用于确定方。证书由值得信任的第三方机构颁发，用以证明服务器和客户端是实际存在的。另外，伪造证书从技术角度来说是异常困难的一件事。所以只要能够确认通信方（服务器或客户端）持有的证书，即可判断通信方的真实意图

### 4. HTTPS是身披SSL外壳的HTTP

HTTPS并非是应用层的一种新协议。只是HTTP通信接口部分用SSL/TLS协议代替而已。通常，HTTP直接和TCP通信。当使用SSL/TLS时，则演变成HTTP先和SSL/TLS通信，再由SSL/TLS和TCP通讯。  
简而言之，HTTPS，其实就是身披SSL外壳的HTTP

![image](http://blog.lpc-win32.com/img/2017-08-04/http-https.png)

### 5. 非对称加密

公开密钥加密使用一对非对称的密钥。一把叫做私有密钥（private key），另一把叫做公开密钥（public key）。顾名思义，私有密钥不能让其他任何人知道，而公开密钥则可以随意发布，任何人都可以获得。公开密钥和私有密钥是配对的一套密钥

使用公开密钥加密方式，发送密文的一方使用对方的公开密钥进行加密处理，对方收到被加密的信息后，再使用自己的私有密钥进行解密。利用这种方式，不需要发送用来解密的私有密钥，也不必担心密钥被攻击者窃听而盗走

### 6. HTTPS的安全通信机制

HTTPS通信步骤如下图所示：

![image](http://blog.lpc-win32.com/img/2017-08-04/https-communicate.png)

1. 客户端通过发送Client Hello报文开始SSL通信。报文中包含客户端支持的SSL的指定版本、加密组件（Cipher Suite）列表
2. 服务器可进行SSL通信时，会以Server Hello报文作为应答。和客户端一样，在报文中包含SSL版本以及加密组件。服务器的价目组件内容是从接收到的客户端加密组件内筛选出来的
3. 之后服务器发送Certificate报文。报文中包含公开密钥证书
4. 最后服务器发送Server Hello Done报文通知客户端，最初阶段的SSL握手协商部分结束
5. SSL第一次握手结束之后，客户端以Client Key Exchange报文作为回应。报文中包含通信加密中使用的一种被称为Pre-master secret的随机密码串。该报文已用步骤3中的公开密钥进行加密
6. 接着客户端继续发送Change Cipher Spec报文。该报文会提示服务器，在此报文之后的通信会采用Pre-master secret密钥加密
7. 客户端发送Finished报文。该报文包含连接至今全部报文的整体校验值。这次握手协商是否能够成功，要以服务器是否能够正确解密该报文作为判定标准
8. 服务器同样发送Change Cipher Spec报文
9. 服务器同样发送Finished报文
10. 服务器和客户端的Finished报文交换完毕之后，SSL连接就算建立完成。当然，通信会受到SSL的保护。从此处开始进行应用层协议的通信，即发送HTTP请求
11. 应用层协议通信，即发送HTTP响应
12. 最后由客户端断开连接。断开连接时，发送close notify报文。之后发送TCP FIN报文来关闭与TCP的通信

在以上的流程中，应用层发送数据时会附加一种叫做MAC（MEssage Authentication Code）的报文摘要。MAC能够查知报文是否遭到篡改，从而保护报文的完整性
