# OkHttp3--详细使用及源码分析系列之初步介绍(一).md

[toc]

## HTTP诞生

* HTTP(hypertex protocol超文本传输协议)最初由美国人 Ted Nels提出。后来他和WWW协会，IETF组织发布了一系列RFC(Request for Comments请求意见稿，征求意见的公开文件)，最著名的是HTTP1.1。
* HTTP是用来从WWW服务器中把网页提取到本地浏览器的一种服务。
* 客户端发送的HTTP 请求(Request)格式：
  * 请求行(request line)
  * 请求头部(header)
  * 空行
  * 请求数据
* 客户端接收到的响应(respond)格式：
  * 状态行
  * 消息报头
  * 空行
  * 响应正文

## SPDY的诞生

* 诞生原因：网络传输内容越来越复杂，HTTP无连接无状态由优点转为缺点。

  * 无连接：在完成一次客户的请求后，立即断开连接，这样可以避免一直占用网络资源。但后期网络传输的图片增多，出现了每传输一个图片就要重新连接一次的现象，反而浪费了资源。这时候Keep-Alive出现，确保客户端到服务器间网络一直连接。
  * 无状态：也就是说对请求完成响应后，完全不记录之前进行请求响应的任何信息，下一次如果要处理前面的信息，就需要一切全部重新连接。在动态交互的web式程序出现后，非常不方便。连最基础的购物车功能都会变得很麻烦。后来出现了**Cookie**和**Session**,前者保存在本地(客户端)浏览器上，保存用户的登录账户等信息，后者会在服务器端保存一个**Sessionid**，每次响应都会把**Sessionid**返回给客户端，关闭浏览器后，会被释放。

* SPDY的特点

  * 支持多路复用
    * 多路复用模型：
      * 一个老师让30个学生做一道题，然后还需要检查。你站在讲台上等，谁解答完谁举手。这时C、D举手，表示他们解答问题完毕，你下去依次检查C、D的答案，然后继续回到讲台上等。此时E、A又举手，然后去处理E和A。这就是**I/O复用模型**。
      * Linux下的select、poll和epoll就是干这个的。将用户socket对应的fd注册进epoll，然后epoll帮你监听哪些socket上有消息到达，这样就避免了大量的无用操作。此时的socket应该采用非阻塞模式。
  * 对请求划分等级
  * 压缩请求头，减少网络传输数据量

## HTTP2.0

  * HTTP2.0 是 IETF 组织基于 SPDY 定制的新一代 http 协议，它比 SPDY 有更安全的 SSL 协议，采用了更安全的加密算法。

## ***重点：OkHttp简介***

  ### 简介：

    * 是Square公司贡献的轻量开源网络请求框架，也是目前Android端最火的，用来替代HttpURLConnection(谷歌在Android4.4版本开始替代，6.0版本已正式移除HttpURLConnection相关API)。同时现在流行的 **Retrofit 框架底层**同样是使用 OKHttp。

  ### 官方学习途径：

    * 官网网站：OkHttp，里面有详细的 API 介绍，及使用案例
      项目托管：Github，有任何使用问题可以 issue。

  ### OkHttp优点：

    * 支持 SPDY HTTP2.0；如果服务器和客户端都支持这两种 它会用同一个 socket 来处理同一个服务器的所有请求 提高 http 连接效率
    * 如果服务器不支持 SPDY 则通过连接池来减少请求延时
    * 无缝的支持 GZIP 来减少数据流量
    * 缓存响应数据减少重复的网络请求
    * 请求失败时自动重试主机的其他 ip，自动重定向
    * 多路复用机制

  ### 使用到的设计模式

    * 单例模式
    * 外观模式
    * 构建者模式
    * 工厂方法模式
    * 享元模式
    * 责任链模式
    * 策略模式

  ### 使用流程

    * ![img](https://img-blog.csdn.net/20180905093307103?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwOTkzNTk1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

  ### 使用前准备

    * 在 build.gradle 中添加依赖
    
    * ```java
      compile 'com.squareup.okhttp3:okhttp:3.7.0'
      compile 'com.squareup.okio:okio:1.12.0'
      ```
    
    * 同时在 Manifest 里添加网络权限
    
    * ```xml
      <uses-permission android:name="android.permission.INTERNET"></uses-permission>
      ```
    
    * 如果需要缓存还要添加写内存的权限
    
    * ```xml
      <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"></uses-permission>
      ```

## 参考文章

* [(33条消息) OKHttp3--详细使用及源码分析系列之初步介绍【一】_Mango先生的博客-CSDN博客](https://blog.csdn.net/qq_30993595/article/details/82079586#_2)
* [(33条消息) 如何理解HTTP协议的 “无连接，无状态” 特点？_秋叶原 && Mike || 麦克-CSDN博客_无状态连接](https://blog.csdn.net/tennysonsky/article/details/44562435)
* [彻底理解 IO 多路复用实现机制 - 掘金 (juejin.cn)](https://juejin.cn/post/6882984260672847879)

