#Netflix-Eureka
##Eureka快速一览
###Eureka是什么
Eureka是一个REST（代表性状态传输）的服务，主要是在AWS的云用于实现负载均衡和中间层服务器故障转移的目的定位服务。

Eureka由两个组件组成：**Eureka服务器**和**Eureka客户端**。Eureka服务器用作服务注册服务器。Eureka客户端是一个java客户端，用来简化与服务器的交互、作为轮询负载均衡器，并提供服务的故障切换支持。Netflix在其生产环境中使用的是另外的客户端，它提供基于流量、资源利用率以及出错状态的加权负载均衡。

###Eureka客户端与服务器如何通信
Eureka可以帮助你找到你想与沟通的服务器的信息，但并没有对协议或通信方式作出任何限制。例如，你可以使用Eureka获取目标服务器地址和使用协议如thrift，HTTP（S）或任何其他RPC机制进行通信。

###Eureka系统结构
![High level architecture](https://github.com/Netflix/eureka/raw/master/images/eureka_architecture.png)


##配置Eureka
Eureka有两个组件

* **Application Client**: 使用Eureka Cleint对Application Service发起请求
* **Application Service** : 接受Application Client的请求并且相应 

设置涉及如下三个部分：

* Eureka Server
* Eureka Client for the application client
* Eureka Client for the application service

> 运行在AWS上启动时需加入java参数： -Deureka.datacenter=cloud

###配置Eureka Client
####配置需求
* JDK1.8+
* 下载Eureka Client([直接下载](http://search.maven.org/#search%7Cga%7C1%7Ceureka-client) OR 使用maven)

~~~xml
 <dependency>
  <groupId>com.netflix.eureka</groupId>
  <artifactId>eureka-client</artifactId>
  <version>1.1.16</version>
 </dependency>
~~~


