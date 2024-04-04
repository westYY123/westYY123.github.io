---
layout: post
title: About Authentication
date: 2024-04-04
categories: blog
tags: [IAM]
description: 
---


众所周知（如果不知道现在就知道了），我就职于一家公有云提供商的认证（Authentication）与鉴权（Authorization）团队的创新孵化组，业界一般把我们的业务叫IAM（Identity and Access Management），也有叫IDaas（Identify as a Service）与RAM（Resource Access Management）的。

如前文所说，我们的业务分为两个模块：认证与鉴权。认证就是对于一个请求或者说操作，我们需要先识别（Identify）出这是哪个主体（Principal），这个主体可以是很多种，在AWS中，这个主体可以是别的云服务（例如S3、EC2等），也可以是IAM user（一种IAM构建的身份，可以将一个IAM账号理解为一个公司，一个IAM user对应公司下的一个员工），还可以是一种AWS的高级身份AssumedRole，这是IAM Role的实例化，简单来说，IAM Role是定义的一个虚拟身份，这个虚拟身份有自己的权限集合，也有自己的信任策略（Trust Policy），权限集合是这个虚拟身份拥有的权限，信任策略则是限制什么主体（在什么条件下）可以获得（Assume）这个虚拟身份，在一个主体获取了这个Role后，会实例化为AssumesRole。这么说仍然有点晦涩难懂，可以类比一下，一个Role就是一类工卡，这类工卡拥有进入一些办公区的权限，这类工卡可以在一个员工（也就是主体）在满足信任策略时带上，在员工带上这个工卡后，就拥有了这个工卡的权限。

鉴权则是在认证后，一个主体想对某个资源执行某个操作时，需要鉴定此用户在这样的场景或者说上下文（Attributes）下是否拥有对此资源执行某操作的权限，主流的鉴权模型有RBAC（Role Based Access Control）与ABAC（Attributes Based Access Control）等，上文说到的AWS Role，其实是ABAC，AWS一直也是构建的基于ABAC的访问控制，因此我认为AWS取得Role名字其实是不太符合语义的，AWS的Role也并不是RBAC中的这个Role。RBAC依赖于精细化的分配权限到角色（Role）中，同时RBAC缺乏灵活性，不能够限制鉴权时的上下文，比如ABAC能够做到在鉴权时检查调用链，比如用户调用云服务A，云服务A调用云服务B的某个操作，ABAC能够很容易的限制（通过条件键）必须是用户-〉云服务A-〉云服务B的调用链，能够让其他的调用链皆鉴权失败。

本次我们不详细讨论鉴权模型与健全细节，也不讨论密码学中的加解密算法，先详细说说身份认证的几种方式与问题。

现代的计算机通信中，一般使用HTTP协议，HTTP协议是一个无状态的模型，比如你访问淘宝，首先需要你登录对吧，但是登陆后，怎么能够让你后续的HTTP请求（比如你加入购物车、付款与评价等）能够被服务端识别出是你在操作而不是别人的操作，也就是说，在你通过账号密码登陆后，服务端是通过了什么手段来维持一个和你之间的协议，能够让服务端识别出后续你的访问来自于你。

传统的方法是Cookie与Session，Cookie就是在你使用你的凭据登陆时，浏览器返回一串数据，在你后续的请求中，都带上这一串数据，浏览器便能识别后续请求来自于你。Cookie被明文保存在你的浏览器，很容易造成泄露，同时，浏览器并没有手段能验证这个Cookie是否就是发放给你的Cookie，很容易被伪造。在现代的微服务架构中，一个微服务会部署多个节点，也就是多台服务器，通过负载均衡（Load Balance），会将请求随机的分配到某台机器上执行，由于Cookie浏览器不保存任何数据，因此每台服务器皆能识别。

Cookie浏览器不保存数据，导致容易被伪造与篡改等风险，因此Session出现了，在登陆时，浏览器不光会返回一个Cookie，也会在浏览器端设置一个Session，来记录这个Cookie的状态、时间等，这个Session，一般存放在内存中，但在上述的多节点微服务架构中，会有你登陆时是机器A处理了你的登陆请求并将Session保存在机器A的内存中，但你后续的请求被随机到了机器B或者机器C，机器B与机器C并不能认出你的Cookie，一种解决方案是将Session保存在数据库中，每台机器需要鉴别Cookie时都去访问数据库，但这样纯属多此一举：我们设置状态的目的就是为了减少数据库的访问压力，如果将Session都保存在数据库中，为什么不每次请求都带上账号密码，通过在数据库检查账号密码来验证身份？

更要命的是，Cookie与Session皆没有加密或者签名来保证完整性与机密性。

根据以上的描述，我们已经知道了Cookie与Session存在的问题。接下来介绍一种目前较为主流的身份认证方式：JWT（JsonWebToken）。

JWT由三部分构成：Header、Payload与Signature。一个完整的JWT是{Base64(Header).Base64(Payload).Signature}的格式（Base64为一种编码方法，能够将任意长的字符串编码为定长的字符串，同时能够使用Base64解码）。

Header一般包含JWT的类型与签名算法等信息，例如：
{
    'alg': "HS256",
    'typ': "JWT"
}，Payload则包含用户信息以及一些定制信息如用户唯一身份ID、JWT过期时间与JWT颁发时间等，Signature则是{Header指定的加密算法}("Base64(Header).Base64(Payload)", SecretKey)。在处理登陆请求校验成功后，服务端会使用服务端的密钥来颁发一个JWT返回给客户端，客户端在后续的请求带上JWT，请求到达服务端后，服务器会通过Base64将Header与Payload解开获取加密方式信息、用户信息与JWT信息等，同时使用密钥来签名{Base64(Header).Base64(Payload)}，将结果与JWT中的签名对比，来验证JWT的有效性。

Cookie的不可校验通过签名解决，Session需要在服务器保存状态与不同服务器之间的Session同步，在JWT中，只需要所有的服务器保存用来签名的密钥就行，大大的降低了服务器的开销。

在公有云领域，还有另一种主流的认证方式，就是AK（Access Key ID）/SK（Secret Access Key），一个AK对应一个唯一SK，在请求时，请求会带上AK与使用了SK加密了整个请求内容的签名，服务器接收到请求后，查找服务端保存的请求中AK对应的SK，使用SK签名请求体，校验签名是否与请求中的签名一致。
