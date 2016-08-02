#Spring Cloud
##特性
* 分布式配置管理
* 服务注册于发现
* 路由
* 服务间调用
* 负责均衡
* 断路器
* 全局锁
* 领导选举和集群状态监控
* 分布式消息

##云原生程序 ： Cloud Native Applications
云原生是一种应用程序开发的风格，鼓励在连续交付和价值驱动开发领域的最佳实践的最佳做法。

###12因子应用程序 12-factor-Apps：
[参考链接1](http://12factor.net)
<br/>
[参考链接2](http://www.the12factorapp.com/)

###Spring 对云原生的支持体现：
**Spring Cloud Context:** Spring Cloud Context provides utilities and special services for the ApplicationContext of a Spring Cloud application (bootstrap context, encryption, refresh scope and environment endpoints).

**Spring Cloud Commons:** Spring Cloud Commons is a set of abstractions and common classes used in different Spring Cloud implementations (eg. Spring Cloud Netflix vs. Spring Cloud Consul).

>注：加密需要额外安装JAVA JCE

##Spring Cloud Context: Application Context Services
Spring Cloud建立在Spring Boot的基础之上。

###The Bootstrap Application Context
Spring Cloud 通过创建'bootstrap'作为程序的父上下文context.<br/>
bootstrap context 使用不同的规则去定位外部配置，即使用bootstrap.yml区分应用程序的 application.yml。例如：<br/>
<font color=red>bootstrap.yml</font>

```
spring:
  application:
    name: foo
  cloud:
    config:
      uri: ${SPRING_CONFIG_URI:http://localhost:8888}
```

###Application Context Hierarchies
如果你使用了***SpringApplication*** 或者 ***SpringApplicationBuilder*** 来构建一个应用程序上下文Context, Spring Cloud自动创建一个父Cotnext,即上述的Bootstrap。
子Context会继承父Context的一些属性：
		
* **"bootstrap"**: an optional CompositePropertySource appears with high priority if any PropertySourceLocators are found in the Bootstrap context, and they have non-empty properties. An example would be properties from the Spring Cloud Config Server. 来自Spring Cloud Server的配置
*  **"applicationConfig: [classpath:bootstrap.yml]"**: If you have a bootstrap.yml (or properties) then those properties are used to configure the Bootstrap context, and then they get added to the child context when its parent is set. They have lower precedence than the application.yml (or properties) and any other property sources that are added to the child as a normal part of the process of creating a Spring Boot application. 优先级比较低.

>bootstrap.yml 优先级较低，适合设定一些默认值

###Changing the Location of Bootstrap Properties
bootstrap.yml (or .properties) 可以通过设定环境变量：*spring.cloud.bootstrap.name* 或者 *spring.cloud.bootstrap.location*
来改变。

###Overriding the Values of Remote Properties
bootstrap context往程序里面添加的属性(通常来自配置服务器)是不能被本地修改的，除非使用命令行工具。或者，远程配置服务器需要指定：
*spring.cloud.config.allowOverride=true* <br/>
*spring.cloud.config.overrideNone=true* to override with any local property source, <br/>
and *spring.cloud.config.overrideSystemProperties=false* if only System properties and env vars should override the remote settings, but not the local config files.

###Customizing the Bootstrap Configuration
[见链接](https://segmentfault.com/a/1190000004319673)

###Customizing the Bootstrap Property Sources
添加额外的配置,实现PropertySourceLocator

```java
@Configuration
public class CustomPropertySourceLocator implements PropertySourceLocator {

    @Override
    public PropertySource<?> locate(Environment environment) {
        return new MapPropertySource("customProperty",
                Collections.<String, Object>singletonMap("property.from.sample.custom.source", "worked as intended"));
    }

}
```
如果把这个类打包成jar,需要在 *META-INF/spring.factories* 中添加：

~~~
org.springframework.cloud.bootstrap.BootstrapConfiguration=sample.custom.CustomPropertySourceLocator
~~~

###Environment Changes
应用程序会监听 ***EnvironmentChangedEvent*** 事件并对变更做出反应(additional ApplicationListeners can be added as @Beans by the user in the normal way)。

当发生 ***EnvironmentChangedEvent*** 事件被观察到时会触发以下事件：

1. Re-bind any @ConfigurationProperties beans in the context
2. Set the logger levels for any properties in logging.level.*

###Refresh Scope
A Spring @Bean that is marked as @RefreshScope will get special treatment when there is a configuration change. This addresses the problem of stateful beans that only get their configuration injected when they are initialized. For instance if a DataSource has open connections when the database URL is changed via the Environment, we probably want the holders of those connections to be able to complete what they are doing. Then the next time someone borrows a connection from the pool he gets one with the new URL.

Refresh scope beans are lazy proxies that initialize when they are used (i.e. when a method is called), and the scope acts as a cache of initialized values. To force a bean to re-initialize on the next method call you just need to invalidate its cache entry.

The RefreshScope is a bean in the context and it has a public method refreshAll() to refresh all beans in the scope by clearing the target cache. There is also a refresh(String) method to refresh an individual bean by name. This functionality is exposed in the /refresh endpoint (over HTTP or JMX).
> @RefreshScope works (technically) on an @Configuration class, but it might lead to surprising behaviour: e.g. it does not mean that all the @Beans defined in that class are themselves @RefreshScope. Specifically, anything that depends on those beans cannot rely on them being updated when a refresh is initiated, unless it is itself in @RefreshScope (in which it will be rebuilt on a refresh and its dependencies re-injected, at which point they will be re-initialized from the refreshed @Configuration).

###Encryption and Decryption
需要JCE支持。

###Endpoints 终端
几个拓展终端：

* POST to **/env** to update the Environment and rebind @ConfigurationProperties and log levels 更新、重新绑定
* **/refresh** for re-loading the boot strap context and refreshing the @RefreshScope beans 重新加载
* **/restart** for closing the ApplicationContext and restarting it (disabled by default) 重启
* **/pause** and **/resume** for calling the Lifecycle methods (stop() and start() on the ApplicationContext) 暂停、重启

##Spring Cloud Commons: Common Abstractions
模式如服务发现、负载平衡和断路器本身是一个抽象层，可以被所有Spring Cloud客户消费，独立的实现（e.g. discovery via Eureka or Consul）。

###Spring RestTemplate as a Load Balancer Client
RestTemplate can be automatically configured to use ribbon. To create a load balanced RestTemplate create a RestTemplate @Bean and use the @LoadBalanced qualifier.

```java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    public String doOtherStuff() {
        String results = restTemplate.getForObject("http://stores/stores", String.class);
        return results;
    }
}
```

###Multiple RestTemplate objects
If you want a RestTemplate that is not load balanced, create a RestTemplate bean and inject it as normal. To access the load balanced RestTemplate use the `@LoadBalanced qualifier when you create your @Bean.

~~~java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate loadBalanced() {
        return new RestTemplate();
    }

    @Primary
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    @LoadBalanced
    private RestTemplate loadBalanced;

    public String doOtherStuff() {
        return loadBalanced.getForObject("http://stores/stores", String.class);
    }

    public String doStuff() {
        return restTemplate.getForObject("http://example.com", String.class);
    }
}
~~~

###Ignore Network Interfaces
<font color=red>application.yml</font>

~~~
spring:
  cloud:
    inetutils:
      ignoredInterfaces:
        - docker0
        - veth.*
~~~

##Spring Cloud Config
分布式配置管理。

###Spring Cloud Config Server
服务端配置，提供一个以资源为基础的HTTP API用以额外的配置。使用 **@EnableConfigServer** 来启用配置服务器。

~~~java
@SpringBootApplication
@EnableConfigServer
public class ConfigServer {
  public static void main(String[] args) {
    SpringApplication.run(ConfigServer.class, args);
  }
}
~~~

<font color=red>application.yml</font>

~~~
#内置服务器端口
server:
  port: 8082
spring:
  cloud:
    config:
      server:
        git:
          # 配置位置
          uri: "http://www.panypopo.pw:93/panypopo/global-configs.git"
        #URL 前缀
        prefix: "/config" 
~~~

####Environment Repository 环境信息库
配置信息存储在配置服务器的哪个位置？这个策略是由 **EnvironmentRepository** 来治理的。This Environment is a shallow copy of the domain from the Spring Environment (including propertySources as the main feature). The Environment resources are parametrized by three variables:

* {application} maps to "spring.application.name" on the client side; 由"spring.application.name"指定
* {profile} maps to "spring.profiles.active" on the client (comma separated list); and
* {label} which is a server side feature labelling a "versioned" set of config files. 版本信息号，如git的master

Repository implementations generally behave just like a Spring Boot application loading configuration files from a **"spring.config.name"** equal to the **{application}** parameter, and **"spring.profiles.active"** equal to the **{profiles}** parameter. Precedence rules for profiles are also the same as in a regular Boot application: active profiles take precedence over defaults, and if there are multiple profiles the last one wins (like adding entries to a Map).

示例：一个客户端程序有这样一个boostrap.yml:

<font color=red>boostrap.yml</font>

~~~
spring:
  application:
    name: foo
  profiles:
    active: dev,mysql
~~~

如果库是基于文件的，服务器将创建一个环境application.yml（所有客户端之间共享），和foo.yml（与foo.yml优先考虑）如果YAML文件里面有文档指向Spring配置文件，这些配置会被较高优先级地应用，如果有YAML（或 Properties）文件，这些也适用具有比缺省更高的优先级。
***Higher precedence translates to a PropertySource listed earlier in the Environment.***  高优先级的会转化成 Environment 中的 PropertySource，并且它的排序比较靠前。


####Git Backend
The default implementation of EnvironmentRepository uses a Git backend, which is very convenient for managing upgrades and physical environments, and also for auditing changes. 

To change the location of the repository you can set the **"spring.cloud.config.server.git.uri"** configuration property in the Config Server (e.g. in application.yml). 

If you set it with a file: prefix it should work from a local repository so you can get started quickly and easily without a server, but in that case the server operates directly on the local repository without cloning it (it doesn’t matter if it’s not bare because the Config Server never makes changes to the "remote" repository).

To scale the Config Server up and make it highly available（高可用）, you would need to have all instances of the server pointing to the same repository, so only a shared file system would work.（所有的配置服务器应指向同一个库）Even in that case it is better to use the **ssh:** protocol for a shared filesystem repository, so that the server can clone it and use a local working copy as a cache.（使用SSH？需要在GIT上记录下所有server的public key）

This repository implementation maps the {label} parameter of the HTTP resource to a git label (commit id, branch name or tag). If the git branch or tag name contains a slash **("/")** then the label in the HTTP URL should be specified with the special string **"(_)"** instead (to avoid ambiguity with other URL paths). Be careful with the brackets in the URL if you are using a command line client like curl (e.g. escape them from the shell with quotes '').

####Placeholders in Git URI
Spring Cloud Config Server supports a git repository URL with placeholders for the {application} and {profile} (and {label} if you need it, but remember that the label is applied as a git label anyway). So you can easily support a "one repo per application" policy using (for example):

<font color=red>application.yml</font>

~~~
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/{application}
~~~

or a "one repo per profile" policy using a similar pattern but with {profile}.
 
####Pattern Matching and Multiple Repositories
正则表达式以及多个库。
There is also support for more complex requirements with pattern matching on the application and profile name. The pattern format is a comma-separated list of {application}/{profile} names with wildcards (where a pattern beginning with a wildcard may need to be quoted). Example:

~~~
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
          repos:
            simple: https://github.com/simple/config-repo
            special:
              pattern: special*/dev*,*special*/dev*
              uri: https://github.com/special/config-repo
            local:
              pattern: local*
              uri: file:/home/configsvc/config-repo

~~~

If {application}/{profile} does not match any of the patterns, it will use the default uri defined under **"spring.cloud.config.server.git.uri"**. In the above example, for the "simple" repository, the pattern is simple/* (i.e. it only matches one application named "simple" in all profiles). The "local" repository matches all application names beginning with "local" in all profiles (the /* suffix is added automatically to any pattern that doesn’t have a profile matcher).

The pattern property in the repo is actually an array, so you can use a YAML array (or [0], [1], etc. suffixes in properties files) to bind to multiple patterns. You may need to do this if you are going to run apps with multiple profiles. Example:

~~~
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
          repos:
            development:
              pattern:
                - */development
                - */staging
              uri: https://github.com/development/config-repo
            staging:
              pattern:
                - */qa
                - */production
              uri: https://github.com/staging/config-repo
~~~

**Every repository can also optionally store config files in sub-directories, and patterns to search for those directories can be specified as searchPaths. For example at the top level:**

~~~
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
          searchPaths: foo,bar*
~~~
In this example the server searches for config files in the top level and in the "foo/" sub-directory and also any sub-directory whose name begins with "bar".

By default the server clones remote repositories when configuration is first requested. The server can be configured to clone the repositories at startup. For example at the top level: **cloneOnStart: false**属性标记启动时clone配置并缓存到本地。

~~~
spring:
  cloud:
    config:
      server:
        git:
          uri: https://git/common/config-repo.git
          repos:
            team-a:
                pattern: team-a-*
                cloneOnStart: true
                uri: http://git/team-a/config-repo.git
            team-b:
                pattern: team-b-*
                cloneOnStart: false
                uri: http://git/team-b/config-repo.git
            team-c:
                pattern: team-c-*
                uri: http://git/team-a/config-repo.git
~~~

In this example the server clones team-a’s config-repo on startup before it accepts any requests. All other repositories will not be cloned until configuration from the repository is requested。克隆都会触发，只是启动时克隆、或者第一次请求的时候克隆的区别。

To use HTTP basic authentication on the remote repository add the "username" and "password" properties separately (not in the URL), e.g. 直接使用用户名密码

~~~
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
          username: trolley
          password: strongpassword
~~~

If you don’t use HTTPS and user credentials, SSH should also work out of the box when you store keys in the default directories (~/.ssh) and the uri points to an SSH location, e.g. "git@github.com:configuration/cloud-configuration". The repository is accessed using JGit, so any documentation you find on that should be applicable. HTTPS proxy settings can be set in ~/.git/config or in the same way as for any other JVM process via system properties (**-Dhttps.proxyHost and -Dhttps.proxyPort**).

####Placeholders in Git Search Paths
Spring Cloud Config Server also supports a search path with placeholders for the {application} and {profile} (and {label} if you need it). Example:

~~~
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
          searchPaths: '{application}'
~~~

searches the repository for files in the same name as the directory (as well as the top level). Wildcards are also valid in a search path with placeholders (any matching directory is included in the search).

####Version Control Backend Filesystem Use
With VCS based backends (git, svn) files are checked out or cloned to the local filesystem. By default they are put in the system temporary directory with a prefix of config-repo-. On linux, for example it could be /tmp/config-repo-<randomid>. Some operating systems routinely clean out temporary directories. This can lead to unexpected behaviour such as missing properties. To avoid this problem, change the directory Config Server uses, by setting **spring.cloud.config.server.git.basedir** or **spring.cloud.config.server.svn.basedir** to a directory that does not reside in the system temp structure.
设置git clone的路径，避免clone下来的配置文件被清除。

####File System Backend
There is also a "native" profile in the Config Server that doesn’t use Git, but just loads the config files from the local classpath or file system (any static URL you want to point to with "spring.cloud.config.server.native.searchLocations"). To use the native profile just launch the Config Server with "spring.profiles.active=native".

>Remember to use the file: prefix for file resources (the default without a prefix is usually the classpath). Just as with any Spring Boot configuration you can embed ${}-style environment placeholders, but remember that absolute paths in Windows require an extra "/", e.g. file:///${user.home}/config-repo

>The default value of the searchLocations is identical to a local Spring Boot application (so [classpath:/, classpath:/config, file:./, file:./config]). This does not expose the application.properties from the server to all clients because any property sources present in the server are removed before 
being sent to the client.

>A filesystem backend is great for getting started quickly and for testing. To use it in production you need to be sure that the file system is reliable, and shared across all instances of the Config Server.

####Sharing Configuration With All Applications
With file-based (i.e. git, svn and native) repositories, resources with file names in application* are shared between all client applications (so application.properties, application.yml, application-*.properties etc.). You can use resources with these file names to configure global defaults and have them overridden by application-specific files as necessary.

The #_property_overrides[property overrides] feature can also be used for setting global defaults, and with placeholders applications are allowed to override them locally.

####Property Overrides 属性覆盖
The Config Server has an "overrides" feature that allows the operator to provide configuration properties to all applications that cannot be accidentally changed by the application using the normal Spring Boot hooks. To declare overrides just add a map of name-value pairs to **spring.cloud.config.server.overrides**. For example

~~~
spring:
  cloud:
    config:
      server:
        overrides:
          foo: bar
~~~

>Normal, Spring environment placeholders with "${}" can be escaped (and resolved on the client) by using backslash ("\") to escape the "$" or the "{", e.g. \${app.foo:bar} resolves to "bar" unless the app provides its own "app.foo". Note that in YAML you don’t need to escape the backslash itself, but in properties files you do, when you configure the overrides on the server.

####Health Indicator
Config Server comes with a Health Indicator that checks if the configured EnvironmentRepository is working. By default it asks the EnvironmentRepository for an application named app, the default profile and the default label provided by the EnvironmentRepository implementation.

You can configure the Health Indicator to check more applications along with custom profiles and custom labels, e.g.

~~~
spring:
  cloud:
    config:
      server:
        health:
          repositories:
            myservice:
              label: mylabel
            myservice-dev:
              name: myservice
              profiles: development
~~~
You can disable the Health Indicator by setting 

**spring.cloud.config.server.health.enabled=false**.

####Security 安全性
You are free to secure your Config Server in any way that makes sense to you (from physical network security to OAuth2 bearer tokens), and Spring Security and Spring Boot make it easy to do pretty much anything.

To use the default Spring Boot configured HTTP Basic security, just include Spring Security on the classpath (e.g. through **spring-boot-starter-security**). The default is a username of "user" and a randomly generated password, which isn’t going to be very useful in practice, so we recommend you configure the password (via **security.user.password**) and encrypt it (see below for instructions on how to do that).

####Encryption and Decryption
> Prerequisites: 需安装JCE

If the remote property sources contain encrypted content (values starting with {cipher}) they will be decrypted before sending to clients over HTTP. The main advantage of this set up is that the property values don’t have to be in plain text when they are "at rest" (e.g. in a git repository). If a value cannot be decrypted it is removed from the property source and an additional property is added with the same key, but prefixed with "invalid." and a value that means "not applicable" (usually "<n/a>"). This is largely to prevent cipher text being used as a password and accidentally leaking.

If you are setting up a remote config repository for config client applications it might contain an application.yml like this, for instance:

~~~
spring:
  datasource:
    username: dbuser
    password: '{cipher}FKSAJDFGYOS8F7GLHAKERGFHLSAJ'
~~~

Encrypted values in a .properties file must not be wrapped in quotes, otherwise the value will not be decrypted:

~~~
spring.datasource.username: dbuser
spring.datasource.password: {cipher}FKSAJDFGYOS8F7GLHAKERGFHLSAJ
~~~

You can safely push this plain text to a shared git repository and the secret password is protected.


###Spring Cloud Config Client
添加依赖

<font style="color:red;">pom.xml</font>

~~~
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.3.5.RELEASE</version>
    <relativePath /> <!-- lookup parent from repository -->
</parent>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Brixton.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>

<!-- repositories also needed for snapshots and milestones -->
~~~

配置spring.cloud.config.uri：

<font style="color:red;">bootstrap.yml</font>

~~~
spring:
  application:
    name: "platform"
  cloud:
    config:
      uri: "http://localhost:9001/"
      profile: "dev"

~~~

####Config Client Fail Fast
In some cases, it may be desirable to fail startup of a service if it cannot connect to the Config Server. If this is the desired behavior, set the bootstrap configuration property spring.cloud.config.failFast=true and the client will halt with an Exception.启动失败处理。

####Config Client Retry 重试
If you expect that the config server may occasionally be unavailable when your app starts, you can ask it to keep trying after a failure. First you need to set spring.cloud.config.failFast=true, and then you need to add spring-retry and spring-boot-starter-aop to your classpath. The default behaviour is to retry 6 times with an initial backoff interval of 1000ms and an exponential multiplier of 1.1 for subsequent backoffs. You can configure these properties (and others) using spring.cloud.config.retry.* configuration properties.

<font style="color:red;">pom.xml</font>

~~~
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
    <version>1.1.3.RELEASE</version>
</dependency>
~~~

<font style="color:red;">bootstrap.xml</font>

~~~
spring:
  cloud:
    config:
      retry:
        initial-interval: 1000
        max-attempts: 10
~~~

>To take full control of the retry add a @Bean of type RetryOperationsInterceptor with id "configServerRetryInterceptor". Spring Retry has a RetryInterceptorBuilder that makes it easy to create one.

####Locating Remote Configuration Resources
The Config Service serves property sources from /{name}/{profile}/{label}, where the default bindings in the client app are

"name" = ${spring.application.name}

"profile" = ${spring.profiles.active} (actually Environment.getActiveProfiles())

"label" = "master"

All of them can be overridden by setting spring.cloud.config.* (where * is "name", "profile" or "label"). The "label" is useful for rolling back to previous versions of configuration; with the default Config Server implementation it can be a git label, branch name or commit id. Label can also be provided as a comma-separated list, in which case the items in the list are tried on-by-one until one succeeds. This can be useful when working on a feature branch, for instance, when you might want to align the config label with your branch, but make it optional (e.g. spring.cloud.config.label=myfeature,develop).

####Security
If you use HTTP Basic security on the server then clients just need to know the password (and username if it isn’t the default). You can do that via the config server URI, or via separate username and password properties, e.g.

~~~
spring:
  cloud:
    config:
     uri: https://user:secret@myconfig.mycompany.com
~~~

OR

~~~
spring:
  cloud:
    config:
     uri: https://myconfig.mycompany.com
     username: user
     password: secret
~~~

The spring.cloud.config.password and spring.cloud.config.username values override anything that is provided in the URI.

If you deploy your apps on Cloud Foundry then the best way to provide the password is through service credentials, e.g. in the URI, since then it doesn’t even need to be in a config file. An example which works locally and for a user-provided service on Cloud Foundry named "configserver":

~~~
spring:
  cloud:
    config:
     uri: ${vcap.services.configserver.credentials.uri:http://user:password@localhost:8888}
~~~

If you use another form of security you might need to provide a RestTemplate to the ConfigServicePropertySourceLocator (e.g. by grabbing it in the bootstrap context and injecting one).

##Spring Cloud Netflix
Spring Boot集成Netflix OSS，包括：

* **Eureka**：服务发现
* **Hystrix**：断路器
* **Zuul**：智能路由
* **Ribbon**：客户端负载均衡

###服务发现：Eureka
在微服务架构中，服务发现是一个关键的元素。试图手动配置每个客户端或某种形式的约定是很难做的，而且是很脆弱的。Eureka是Netflix的服务发现的服务器和客户端。该服务器可以配置和部署为高可用，在每个服务器复制服务的状态给其他节点。

####Eureka注册
When a client registers with Eureka, it provides meta-data about itself such as host and port, health indicator URL, home page etc. Eureka receives heartbeat messages from each instance belonging to a service. If the heartbeat fails over a configurable timetable, the instance is normally removed from the registry.

Example eureka client:

<font color="red">pom.xml</font>

~~~
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
~~~



~~~java
@Configuration
@ComponentScan
@EnableAutoConfiguration
@EnableEurekaClient
@RestController
public class Application {

    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
~~~

(i.e. utterly normal Spring Boot app). In this example we use @EnableEurekaClient explicitly, but with only Eureka available you could also use @EnableDiscoveryClient. Configuration is required to locate the Eureka server. Example:

~~~
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
~~~

where "defaultZone" is a magic string fallback value that provides the service URL for any client that doesn’t express a preference (i.e. it’s a useful default).

The default application name (service ID), virtual host and non-secure port, taken from the Environment, are ${spring.application.name}, ${spring.application.name} and ${server.port} respectively.

@EnableEurekaClient makes the app into both a Eureka "instance" (i.e. it registers itself) and a "client" (i.e. it can query the registry to locate other services). The instance behaviour is driven by eureka.instance.* configuration keys, but the defaults will be fine if you ensure that your application has a spring.application.name (this is the default for the Eureka service ID, or VIP).

####Authenticating with the Eureka Server
HTTP basic authentication will be automatically added to your eureka client if one of the eureka.client.serviceUrl.defaultZone URLs has credentials embedded in it (curl style, like http://user:password@localhost:8761/eureka). 

For more complex needs you can create a @Bean of type DiscoveryClientOptionalArgs and inject ClientFilter instances into it, all of which will be applied to the calls from the client to the server.

>Because of a limitation in Eureka it isn’t possible to support per-server basic auth credentials, so only the first set that are found will be used.

####Status Page and Health Indicator
The status page and health indicators for a Eureka instance default to "/info" and "/health" respectively, which are the default locations of useful endpoints in a Spring Boot Actuator application. You need to change these, even for an Actuator application if you use a non-default context path or servlet path (e.g. server.servletPath=/foo) or management endpoint path (e.g. management.contextPath=/admin). Example:

<font color="red">application.yml</font>

~~~java
eureka:
  instance:
    statusPageUrlPath: ${management.context-path}/info
    healthCheckUrlPath: ${management.context-path}/health
~~~

These links show up in the metadata that is consumed by clients, and used in some scenarios to decide whether to send requests to your application, so it’s helpful if they are accurate.

####Registering a Secure Application

If your app wants to be contacted over HTTPS you can set two flags in the EurekaInstanceConfig, viz eureka.instance.[nonSecurePortEnabled,securePortEnabled]=[false,true] respectively. This will make Eureka publish instance information showing an explicit preference for secure communication. The Spring Cloud DiscoveryClient will always return an https://…​; URI for a service configured this way, and the Eureka (native) instance information will have a secure health check URL.

Because of the way Eureka works internally, it will still publish a non-secure URL for status and home page unless you also override those explicitly. You can use placeholders to configure the eureka instance urls, e.g.

<font color="red">application.yml</font>

~~~
eureka:
  instance:
    statusPageUrl: https://${eureka.hostname}/info
    healthCheckUrl: https://${eureka.hostname}/health
    homePageUrl: https://${eureka.hostname}/
~~~

(Note that ${eureka.hostname} is a native placeholder only available in later versions of Eureka. You could achieve the same thing with Spring placeholders as well, e.g. using ${eureka.instance.hostName}.)

> your app is running behind a proxy, and the SSL termination is in the proxy (e.g. if you run in Cloud Foundry or other platforms as a service) then you will need to ensure that the proxy "forwarded" headers are intercepted and handled by the application. An embedded Tomcat container in a Spring Boot app does this automatically if it has explicit configuration for the 'X-Forwarded-\*` headers. A sign that you got this wrong will be that the links rendered by your app to itself will be wrong (the wrong host, port or protocol).

####Eureka’s Health Checks
By default, Eureka uses the client heartbeat to determine if a client is up. Unless specified otherwise the Discovery Client will not propagate the current health check status of the application per the Spring Boot Actuator. Which means that after successful registration Eureka will always announce that the application is in 'UP' state. This behaviour can be altered by enabling Eureka health checks, which results in propagating application status to Eureka. As a consequence every other application won’t be sending traffic to application in state other then 'UP'.

<font color="red">application.yml</font>

~~~
eureka:
  client:
    healthcheck:
      enabled: true
~~~

If you require more control over the health checks, you may consider implementing your own com.netflix.appinfo.HealthCheckHandler.

####Eureka Metadata for Instances and Clients
It’s worth spending a bit of time understanding how the Eureka metadata works, so you can use it in a way that makes sense in your platform. There is standard metadata for things like hostname, IP address, port numbers, status page and health check. These are published in the service registry and used by clients to contact the services in a straightforward way. Additional metadata can be added to the instance registration in the **eureka.instance.metadataMap**, and this will be accessible in the remote clients, but in general will not change the behaviour of the client, unless it is made aware of the meaning of the metadata. There are a couple of special cases described below where Spring Cloud already assigns meaning to the metadata map.

#####Using Eureka on Cloudfoundry
Cloudfoundry has a global router so that all instances of the same app have the same hostname (it’s the same in other PaaS solutions with a similar architecture). This isn’t necessarily a barrier to using Eureka, but if you use the router (recommended, or even mandatory depending on the way your platform was set up), you need to explicitly set the hostname and port numbers (secure or non-secure) so that they use the router. You might also want to use instance metadata so you can distinguish between the instances on the client (e.g. in a custom load balancer). By default, the eureka.instance.instanceId is vcap.application.instance_id. For example:

<font color="red">application.yml</font>

~~~
eureka:
  instance:
    hostname: ${vcap.application.uris[0]}
    nonSecurePort: 80
~~~

Depending on the way the security rules are set up in your Cloudfoundry instance, you might be able to register and use the IP address of the host VM for direct service-to-service calls. This feature is not (yet) available on Pivotal Web Services (PWS).

#####Using Eureka on AWS
If the application is planned to be deployed to an AWS cloud, then the Eureka instance will have to be configured to be Amazon aware and this can be done by customizing the EurekaInstanceConfigBean the following way:

~~~
@Bean
@Profile("!default")
public EurekaInstanceConfigBean eurekaInstanceConfig() {
  EurekaInstanceConfigBean b = new EurekaInstanceConfigBean();
  AmazonInfo info = AmazonInfo.Builder.newBuilder().autoBuild("eureka");
  b.setDataCenterInfo(info);
  return b;
}
~~~

#####Changing the Eureka Instance ID
A vanilla Netflix Eureka instance is registered with an ID that is equal to its host name (i.e. only one service per host). Spring Cloud Eureka provides a sensible default that looks like this: ${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id:${server.port}}}. For example myhost:myappname:8080.

Using Spring Cloud you can override this by providing a unique identifier in eureka.instance.instanceId. For example:

<font color="red">application.yml</font>

~~~
eureka:
  instance:
    instanceId: ${spring.application.name}:${spring.application.instance_id:${random.value}}
~~~

With this metadata, and multiple service instances deployed on localhost, the random value will kick in there to make the instance unique. In Cloudfoundry the spring.application.instance_id will be populated automatically in a Spring Boot Actuator application, so the random value will not be needed.


####Using the EurekaClient
Once you have an app that is @EnableDiscoveryClient (or @EnableEurekaClient) you can use it to discover service instances from the Eureka Server. One way to do that is to use the native com.netflix.discovery.EurekaClient (as opposed to the Spring Cloud DiscoveryClient), e.g.

~~~java
@Autowired
private EurekaClient discoveryClient;

public String serviceUrl() {
    InstanceInfo instance = discoveryClient.getNextServerFromEureka("STORES", false);
    return instance.getHomePageUrl();
}
~~~

>Don’t use the EurekaClient in @PostConstruct method or in a @Scheduled method (or anywhere where the ApplicationContext might not be started yet). It is initialized in a SmartLifecycle (with phase=0) so the earliest you can rely on it being available is in another SmartLifecycle with higher phase.

####Alternatives to the native Netflix EurekaClient
You don’t have to use the raw Netflix EurekaClient and usually it is more convenient to use it behind a wrapper of some sort. Spring Cloud has support for Feign (a REST client builder) and also Spring RestTemplate using the logical Eureka service identifiers (VIPs) instead of physical URLs. To configure Ribbon with a fixed list of physical servers you can simply set <client>.ribbon.listOfServers to a comma-separated list of physical addresses (or hostnames), where <client> is the ID of the client.

You can also use the org.springframework.cloud.client.discovery.DiscoveryClient which provides a simple API for discovery clients that is not specific to Netflix, e.g.

~~~
@Autowired
private DiscoveryClient discoveryClient;

public String serviceUrl() {
    List<ServiceInstance> list = discoveryClient.getInstances("STORES");
    if (list != null && list.size() > 0 ) {
        return list.get(0).getUri();
    }
    return null;
}
~~~

####Why is it so Slow to Register a Service?
Being an instance also involves a periodic heartbeat to the registry (via the client’s serviceUrl) with default duration 30 seconds. A service is not available for discovery by clients until the instance, the server and the client all have the same metadata in their local cache (so it could take 3 heartbeats). You can change the period using **eureka.instance.leaseRenewalIntervalInSeconds** and this will speed up the process of getting clients connected to other services. In production it’s probably better to stick with the default because there are some computations internally in the server that make assumptions about the lease renewal period.

###Service Discovery: Eureka Server
Example eureka server (e.g. using spring-cloud-starter-eureka-server to set up the classpath):

<font color="red">pom.xml</font>

~~~
<dependency>
    <groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
~~~

~~~java
@SpringBootApplication
@EnableEurekaServer
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
~~~

The server has a home page with a UI, and HTTP API endpoints per the normal Eureka functionality under /eureka/*.

Eureka background reading: see flux capacitor and google group discussion.

####High Availability, Zones and Regions
The Eureka server does not have a backend store, but the service instances in the registry all have to send heartbeats to keep their registrations up to date (so this can be done in memory). Clients also have an in-memory cache of eureka registrations (so they don’t have to go to the registry for every single request to a service).

By default every Eureka server is also a Eureka client and requires (at least one) service URL to locate a peer. If you don’t provide it the service will run and work, but it will shower your logs with a lot of noise about not being able to register with the peer.

See also below for details of Ribbon support on the client side for Zones and Regions.

####Standalone Mode
The combination of the two caches (client and server) and the heartbeats make a standalone Eureka server fairly resilient to failure, as long as there is some sort of monitor or elastic runtime keeping it alive (e.g. Cloud Foundry). In standalone mode, you might prefer to switch off the client side behaviour, so it doesn’t keep trying and failing to reach its peers. Example:


<font color="red">application.yml</font>

~~~
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
~~~

####Peer Awareness
Eureka can be made even more resilient and available by running multiple instances and asking them to register with each other. In fact, this is the default behaviour, so all you need to do to make it work is add a valid serviceUrl to a peer, e.g.

<font color="red">application.yml</font>

~~~
---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2/eureka/

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1/eureka/
~~~
In this example we have a YAML file that can be used to run the same server on 2 hosts (peer1 and peer2), by running it in different Spring profiles. You could use this configuration to test the peer awareness on a single host (there’s not much value in doing that in production) by manipulating /etc/hosts to resolve the host names. In fact, the eureka.instance.hostname is not needed if you are running on a machine that knows its own hostname (it is looked up using java.net.InetAddress by default).

You can add multiple peers to a system, and as long as they are all connected to each other by at least one edge, they will synchronize the registrations amongst themselves. If the peers are physically separated (inside a data centre or between multiple data centres) then the system can in principle survive split-brain type failures.

####Prefer IP Address
In some cases, it is preferable for Eureka to advertise the IP Adresses of services rather than the hostname. Set eureka.instance.preferIpAddress to true and when the application registers with eureka, it will use its IP Address rather than its hostname.

####参考
> [Spring Cloud构建微服务架构（一）服务注册与发现](http://blog.didispace.com/springcloud1/)

***

###Circuit Breaker: Hystrix Clients 断路器

>[Spring Cloud构建微服务架构（三）断路器](http://blog.didispace.com/springcloud3/)

Netflix has created a library called Hystrix that implements the circuit breaker pattern. In a microservice architecture it is common to have multiple layers of service calls. Netflix公司创建了一个库称实现断路器模式。在微服架构通常有多个服务层调用。

![Microservice Graph](http://cloud.spring.io/spring-cloud-static/images/HystrixGraph.png)


A service failure in the lower level of services can cause cascading failure all the way up to the user. When calls to a particular service reach a certain threshold (20 failures in 5 seconds is the default in Hystrix), the circuit opens and the call is not made. In cases of error and an open circuit a fallback can be provided by the developer.
![ Hystrix fallback prevents cascading failures](http://cloud.spring.io/spring-cloud-static/images/HystrixFallback.png)

Having an open circuit stops cascading failures and allows overwhelmed or failing services time to heal. The fallback can be another Hystrix protected call, static data or a sane empty value. Fallbacks may be chained so the first fallback makes some other business call which in turn falls back to static data.

Example boot app:

~~~java
@SpringBootApplication
@EnableCircuitBreaker
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}

@Component
public class StoreIntegration {

    @HystrixCommand(fallbackMethod = "defaultStores")
    public Object getStores(Map<String, Object> parameters) {
        //do stuff that might fail
    }

    public Object defaultStores(Map<String, Object> parameters) {
        return /* something useful */;
    }
}
~~~

The @HystrixCommand is provided by a Netflix contrib library called "javanica". Spring Cloud automatically wraps Spring beans with that annotation in a proxy that is connected to the Hystrix circuit breaker. The circuit breaker calculates when to open and close the circuit, and what to do in case of a failure.

To configure the @HystrixCommand you can use the commandProperties attribute with a list of @HystrixProperty annotations. See here for more details. See the Hystrix wiki for details on the properties available.

####Propagating the Security Context or using Spring Scopes 上下文传播 or Spring Scopes
If you want some thread local context to propagate into a @HystrixCommand the default declaration will not work because it executes the command in a thread pool (in case of timeouts). You can switch Hystrix to use the same thread as the caller using some configuration, or directly in the annotation, by asking it to use a different "Isolation Strategy". For example:

~~~
@HystrixCommand(fallbackMethod = "stubMyService",
    commandProperties = {
      @HystrixProperty(name="execution.isolation.strategy", value="SEMAPHORE")
    }
)
...
~~~

The same thing applies if you are using @SessionScope or @RequestScope. You will know when you need to do this because of a runtime exception that says it can’t find the scoped context.

####Health Indicator
The state of the connected circuit breakers are also exposed in the /health endpoint of the calling application.

~~~
{
    "hystrix": {
        "openCircuitBreakers": [
            "StoreIntegration::getStoresByLocationLink"
        ],
        "status": "CIRCUIT_OPEN"
    },
    "status": "UP"
}
~~~

####Hystrix Metrics Stream ???
To enable the Hystrix metrics stream include a dependency on spring-boot-starter-actuator. This will expose the /hystrix.stream as a management endpoint.

~~~
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
~~~

###Circuit Breaker: Hystrix Dashboard
One of the main benefits of Hystrix is the set of metrics it gathers about each HystrixCommand. The Hystrix Dashboard displays the health of each circuit breaker in an efficient manner.

![Hystrix Dashboard](http://cloud.spring.io/spring-cloud-static/images/Hystrix.png)

To run the Hystrix Dashboard annotate your Spring Boot main class with @EnableHystrixDashboard. You then visit /hystrix and point the dashboard to an individual instances /hystrix.stream endpoint in a Hystrix client application.

~~~
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
~~~

~~~java
@SpringBootApplication
@EnableEurekaServer
@EnableCircuitBreaker
@EnableHystrix
@EnableHystrixDashboard
public class PlatformApplication {

	public static void main(String[] args) {
		SpringApplication.run(PlatformApplication.class, args);
	}
}

~~~

####Turbine
Looking at an individual instances Hystrix data is not very useful in terms of the overall health of the system. **Turbine** is an application that aggregates all of the relevant /hystrix.stream endpoints into a combined /turbine.stream for use in the Hystrix Dashboard. Individual instances are located via Eureka. Running Turbine is as simple as annotating your main class with the @EnableTurbine annotation (e.g. using spring-cloud-starter-turbine to set up the classpath). All of the documented configuration properties from the Turbine 1 wiki apply. The only difference is that the turbine.instanceUrlSuffix does not need the port prepended as this is handled automatically unless turbine.instanceInsertPort=false
看一个单独的实例数据在检测系统的整体健康方面并没有多大的作用。Turbine程序收集所有有关的/hystrix.stream终端，合并成一个能 Hystrix Dashboard使用的turbine.stream。通过Eureka来定位一个实例。使用方法：

~~~
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-cloud-starter-turbine</artifactId>
</dependency>
~~~



