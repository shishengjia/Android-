#Http协议
超文本传输协议（HyperText Transfer Protocol)是互联网上应用最为广泛的一种网络协议。


**Http版本区别**
 * 1.1
  * 默认持久连接
  * 支持缓存
  * 支持管道方式发送多个请求
 * 2.0
  * 多路复用允许同时通过单一的HTTP/2连接发起多重的请求-响应消息
    * 单连接多资源的方式，减少服务端的链接压力，内存占用更少，连接吞吐量更大
    * 由于TCP连接的减少而使网络拥塞情况得以改善，同时![慢启动](http://blog.csdn.net/wykwdy007/article/details/6720254)时间的减少，使拥塞和丢包恢复速度更快
  * 头部压缩(HPACK算法)
  * 对请求划分优先级
  * 服务器推送流(Server Push)

**Http请求方式**
 * Get:请求获取Requeat-URI所标识得资源
 * POST:在Request-URI所标识得资源后附加新的数据
 * HEAD,PUT,DELETE,TRACE,CONNECT,OPTIONS
 
**Http协议特点**
 * 支持C/S模式
 * 简单快速:客户向服务器请求服务时，只需传送请求方法和路径。
 * 灵活:HTTP允许传输任意类型的数据对象
 * 无连接
 * 无状态
 
 **Request Headers和Response Headers**
 
 ![image](https://github.com/shishengjia/Android-NetworkArchitecture-Design/blob/master/Headers.jpg)
 
 其中Status Code:200 表示成功
 
 **响应码**
  * 100-199 信息提示
  * 200-299 成功
  * 300-399 重定向
  * 400-499 客户端错误
  * 500-599 服务器错误
 
 User-Agent标明设备信息，浏览器信息等
