---
layout: post
title: Apple Push Notification Service(翻译)
tags: [notes]
description: >
  由于英语水平有限(我没参加过四级考试、哈哈)所以如果有条件的话还是建议去看原官方文档。 
---
Apple Push Notification service(APNs)是远程通知功能的核心。APNs对于传播信息给iOS，tvOS，OS X设备非常稳定和高效。
每一个设备会和APNs建立一个符合标准并加密的IP连接并通过此长连接接收通知。如果当app未运行时接收到一个通知，设备会提示用户此app收到了通知数据。   

你需要提供你自己的服务器来为你的用户生成远程通知。为用户搜集数据并决定通知何时需要被发送出去，这样的服务器被称为provider。
provider为每一个通知生成通知负载(notification payload)，并把此负载添加到一个HTTP/2请求上，其然后用一个使用HTTP/2多路传输协议（multiplex protocol）具有长效性和保密性的通道发送到APNs。
在收到你的请求后，由APNs来处理对你在用户设备上的app通知的分发工作。   

关于发送给APNs的请求格式的信息以及provider会接收到的响应和错误，可以查看[APNs Provider API](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/APNsProviderAPI.html#//apple_ref/doc/uid/TP40008194-CH101-SW1)。
关于怎样在App里实现对通知的支持，查看[Registering, Scheduling, and Handling User Notifications](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/IPhoneOSClientImp.html#//apple_ref/doc/uid/TP40008194-CH103-SW1)。

### 远程通知的流程
APNs会从你的provider到每一个用户的设备，为你的App传输和路由远程通知。下面插图展示了每一个通知的流程   
![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/remote_notif_simple_2x.png)   
当provider确定一个需要发送的通知时，provider要发送通知和设备token到APNs。APNs处理通知发至准确的用户设备的路由，系统处理通知至客户端App的分发。   

提供给APNs的设备token是一个类似于电话号码的东西；它包含允许APNs寻找安装着客户端App的设备的信息。APNs也用它来验证通知的路由。设备token由注册了远程通知协议后接受到token的客户端App提供给provider。

通知负载是一个包含你要发送给设备数据的JSON字典。负载包含关于你要怎么通知你的用户的信息，例如使用alert，badge或者声音。它也可以包含自定义数据。下面插图展示了虚拟网络APNs在provider和设备间的一个更真实的写照。
![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/remote_notif_multiple_2x.png)
APNs两边的provider方和设备方都有多个连接点。provider方面，这些被叫做网关。这是典型的多provider，每一个都通过网关创建一个或多个与APNs具有长效和保密性的连接。这些provider通过APNs给多个装有他们的客户端App的设备发送通知。
   
><small>关于获取设备token的信息，请查看[Token Generation and Dispersal](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/APNsProviderAPI.html#//apple_ref/doc/uid/TP40008194-CH101-SW1)。</small>
><small>关于通知负载，请查看[The Remote Notification Payload](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/IPhoneOSClientImp.html#//apple_ref/doc/uid/TP40008194-CH103-SW1)。<small>

### 服务质量(Quality of Service)
APNs包含一个执行“保存——发送”功能的默认服务质量(Quality of Service  QoS)组件。如果APNs试图发送一个通知但是设备却离线，通知会被保存在一个有限时期内，当设备可以接收通知时将它发出去。
**注意**，*针对一个特定App只有一个最近的通知会被保存。当设备离线时如果有多个通知被发送，新通知会导致之前的通知被抛弃掉。*这种只保存最新通知的方式简称合并通知(coalescing notifications)。   

如果设备长时间保持离线状态，所有被保存的通知都会被抛弃掉。

### 安全架构(Security Architecture)
为了确保安全通讯，APNs用“连接信任(connection trust)”和“token信任(token trust)”两种不同级别的信任(trust)来管理provider和设备间的入口点。   

连接信任建立确定了APNs连接的是苹果已经同意分发通知的被授权provider。APNs也把连接信任用于其和设备之间以确保设备的合法性。设备的连接信任是被APNs自动处理的，但你必须经过一些步骤来确定provider和APNs的连接信任的存在。   

Token信任确保了通知只在合法的起始和终结点间路由。Token信任涉及一个分配在特定设备的特定应用的不可知标识的设备token的使用。每一个应用实例当其向APNs注册时会接收它的独一无二的token，并且应用实例必须跟它的provider共享此token。因此，token必须伴随每一个通过你的provider发出的通知。提供token确保了此通知只被分发给预期的应用/设备组合。

> <small>**注意：**一个设备token并不是一个你可以用来标识设备的唯一ID。设备token在设备的操作系统更新后可能会变。结果就是，应用应该发送它的设备token。</small>

APNs服务也有必要的证书，CA证书，还有用于验证连接以及provider和设备的密钥(私钥和公钥)。

### Provider到APNs的连接信任
每一个provider必须有用于验证provider和APNs的连接的一个独一无二的provider证书和私钥。此provider证书(由苹果颁发)鉴定provider所支持的主题(topics)。(主题是跟你的App关联的bundle ID)   

Provider通过TLS 点对点验证来建立与APNs的连接信任。在TLS连接初始化后，会从APNs获得服务证书然后在你的provider端验证此证书。然后你发送要在APNs端验证的provider证书给APNs。在这步完成后，一个安全的TLS连接就创建了。APNs现在因为此连接通过一个合法的provider已经被创建而满意。（此句话怎么翻译怎么别扭，不知道是不是想拟人 - -!）   
![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/service_provider_ct_2x.png)   
HTTP/2 provider连接对分发至一个特定App有效，通过在证书中特定的主题(bundle ID)来标识。依靠你怎样设置和准备你的APNs SSL 证书，连接也对远程通知到与主App关联的任何Apple Watch复杂的或后台的VoIP服务有效。具体查看[APNs Provider API](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/APNsProviderAPI.html#//apple_ref/doc/uid/TP40008194-CH101-SW1)。APNs 也维护一个证书废弃列表；如果一个provider的证书在此列表上，APNs可以撤除provider信任(也就是拒绝连接)。

### APNs到设备的连接信任(APNs-to-Device Connection Trust)
每个设备都有被用于验证设备与APNs的连接的一个设备证书和私钥。设备在设备激活时期获取它的证书和密钥并把他们保存在keychain中。   
你不必做任何事来创建APNs和设备间的连接信任。APNs通过TLS 点对点验证来确定正在连接设备的一致性。在这步的流程中，设备初始化一个和APNs的TLS连接，它会返回服务证书。设备验证此证书然后把设备证书发送到APNs，APNs会验证设备证书。   
![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/service_device_ct_2x.png)

### Token的生成和传输(Token Generation and Dispersal)
一个应用必须向系统注册接收远程通知，就像[Registering for Remote Notifications](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/IPhoneOSClientImp.html#//apple_ref/doc/uid/TP40008194-CH103-SW2)所描述的。在接收到一个注册请求后，系统会把请求转发到APNs，其使用设备的证书中包含的信息为此App生成一个唯一的设备token。APNs然后会用token密钥加密此token并把它返回给设备，如下图所示。   
![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/token_generation_2x.png)
 
> <small>**重要：**APNs设备token都是不定长的。别对它们的大小做硬编码。</small>

下图解释了token的生成和发送顺序，除此之外还展示了token被APNs返回以及随后转发给provider的过程。   
![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/registration_sequence_2x.png)

### Token信任(通知)
Provider发送给APNs的每一个通知必须伴随着通知意欲达到的设备的设备token。APNs使用token密钥来解密token以确保此通知来源的合法性——正是你的Provider。APNs使用在设备token包含的设备ID来确定目标设备身份。APNs然后把通知发送到相对的设备上，如下所示。   
![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/token_trust_2x.png)