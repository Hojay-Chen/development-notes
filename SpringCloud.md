# 一、Nacos

## 1. 介绍



## 2. 快速入门

### 2.1 服务器部署

#### 2.1.1 下载服务器代码

Spring Cloud Alibaba Nacos下载网址：[Releases · alibaba/nacos](https://github.com/alibaba/nacos/releases)  [https://github.com/alibaba/nacos/releases]

#### 2.1.2 启动服务器

##### 2.1.2.1 单机启动

**Windows系统**

找到文件夹下的包含startup.cmd的bin目录，在该目录下执行命令：

```
.\startup.cmd -m standalone
```

**Linux/Unix/Mac系统**

找到文件夹下包含startup.sh的bin目录，在该目录下执行命令：

```
sh startup.sh -m standalone
```

看到类似如下信息即表示启动成功：

![spring_cloud_nacos_nocsbajo](E:\各种资料\Java开发笔记\我的笔记\images\spring_cloud_nacos_nocsbajo.png)



### 2.2 注册中心

#### 2.2.1 添加依赖

在maven配置文件中添加Nacos依赖：

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>${spring-cloud-nacos.version}</version>
</dependency>
```

#### 2.2.2 编写配置文件

在应用项目的application.yml文件中添加如下配置：

```
spring:
  cloud:
    # nacos的配置
    nacos:
      discovery:
        server-addr: 169.254.8.5:8848
    # 在注册中心中的名字
  application:
    name: gulimall-coupon
```

其中，server-addr是Nacos服务器的ip:port，根据自己的Nacos服务器信息进行填写，而application.name是应用程序在注册中心中的名称，用来区别不用应用程序的，根据自己的业务类进行填写。

#### 2.2.3 添加服务注册与发现注解

有些版本的springboot要求要在项目入口类添加@EnableDiscoveryClient注解，使得应用程序能够被注册中心发现和注册，代码展示如下：

```java
@SpringBootApplication
@EnableDiscoveryClient
@MapperScan("com.hojay.gulimall.coupon.dao")
public class GulimallCouponApplication {

    public static void main(String[] args) {
        SpringApplication.run(GulimallCouponApplication.class, args);
    }

}

```

但比较新的SpringBoot版本则只需要在配置文件中写相关配置即可，无需添加注解，若想关闭服务自动发现，则可以在配置文件中添加如下配置：

```
spring:
  cloud:
    discovery:
      enabled: false
```

#### 2.2.4 启动项目

在启动项目之前，需要确保Nacos服务器已经启动，然后启动SpringBoot项目即可进行客户端服务注册。

#### 2.2.5 查看注册情况

可以进入到Nacos提供的管理页面来查看和管理客户端服务的注册情况，默认网址为：http://ip:8848/nacos/index.html，可以在Nacos服务器启动的命令行中找到，如下图即为注册成功示例：

![spring_cloud_nacos_xnaoib](E:\各种资料\Java开发笔记\我的笔记\images\spring_cloud_nacos_xnaoib.png)

### 2.3 配置中心

#### 2.3.1 添加依赖

添加如下两个依赖，分别为nacos配置中心的启动器和从远程配置中心加载配置i西南西的必要依赖：

```
<!-- spring cloud alibaba nacos 配置中心 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>${spring-cloud-nacos.version}</version>
</dependency>
<!-- 远程获取配置中心的配置文件的基础依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
    <version>4.0.0</version>
</dependency>
```

#### 2.3.2 编写配置文件

在bootstrap.yml / bootstrap.properties配置文件中添加如下配置信息来配置nacos配置中心，以yml文件为例：

```
spring:
  application:
    name: myname
  cloud:
    # nacos的配置
    nacos:
      # 配置中心的配置
      config:
        server-addr: 127.0.0.1:8848
```

其中，spring.application.name填写应用名称，spring.cloud.nacos.config中配置配置中心的各个参数信息。

#### 2.3.3 添加配置文件到配置中心

在Nacos管理页面（上面所说的默认为http://ip:8848/nacos/index.html的网站）点击“配置管理 -> 配置列表 -> 创建配置”即可进入配置文件创建界面，Data ID填写“应用名.配置文件后缀”（如“gulimall-coupon.properties”），命名空间和Group根据业务自行选择，完成配置文件类型选择和内容填写即可创建成功。页面展示如下：

![spring_cloud_nacos_gbaoa](E:\各种资料\Java开发笔记\我的笔记\images\spring_cloud_nacos_gbaoa.png)

#### 2.3.4 配置信息调用

完成如上配置，即可用@Value("...")注解来获取配置中心的配置文件的配置信息了，这里的调用方法和本地配置文件配置信息的调用方法一致。

如果希望Nacos配置中心的配置文件修改后，不经过项目重启直接将新的配置项更新到应用中，则需要在配置项调用的类中添加@RefreshScope注解。如下代码示例：

```java
@RefreshScope  # 动态刷新配置项注解
@RestController
@RequestMapping("...")
public class CouponController {
    @Autowired
    private CouponService couponService;

    @Value("${...}")
    private String name;
}
```



## 3. 服务器部署



## 4. 注册中心



## 5. 配置中心详解

### 5.1 使用详解

- 配置中心的配置项优先于项目本地的配置文件。

### 5.2 引入依赖详解



> ### spring-cloud-starter-bootstrap介绍
>
> `spring-cloud-starter-bootstrap` 依赖引入了 Spring Cloud Bootstrap 的核心功能，它允许应用程序在启动时从远程配置中心（如 Nacos）获取配置项。这是 Spring Cloud 应用程序与配置中心交互的基础。
>
> **配置加载机制**
>
> Spring Cloud Bootstrap 提供了一个配置加载器 (`ConfigDataLocationResolver`)，它能够在应用程序启动时解析和加载远程配置。这个加载器会查找并加载配置文件，如 `bootstrap.yml` 或 `bootstrap.properties`，这些文件通常包含配置中心的地址和其他引导配置。
>
> **配置优先级**
>
> Bootstrap 配置具有最高的优先级，它会在应用程序加载任何其他配置之前被加载。这意味着在 `bootstrap.yml` 或 `bootstrap.properties` 中定义的配置项会覆盖 `application.yml` 或 `application.properties` 中的相同配置项。
>
> **配置管理**
>
> 通过使用 `spring-cloud-starter-bootstrap`，你可以实现配置管理，包括配置的集中管理、动态更新等。这对于微服务架构和云原生应用尤为重要，因为它们需要灵活地适应不断变化的环境和需求。

### 5.3 配置文件详解

Nacos配置中心的配置信息在配置文件boostrap.yml / boostrap.properties的spring.cloud.nacos.config下进行配置，包含了如下配置内容：

1. **`server-addr`**：
   - 指定 Nacos 配置中心的地址。可以是单个地址或地址列表，用于服务发现。
   - 示例：`server-addr=127.0.0.1:8848`

2. **`namespace`**（命名空间）：
   - 默认值为public。
   - 用于隔离不同环境的配置。不同的命名空间可以包含相同的配置项，但是值不同。当需要在多套配置项来回切换，则只需要为每一套配置项定义在不同配置空间，通过修改namespace的值来切换配置项。例如可以创建三个命名空间分别为开发、测试、生产，并且每一个命名空间对应一套配置项，分别在开发、测试、生产时使用，从而隔离了不同环境下的配置项的使用。
   - 示例：`namespace=public`

3. **`group`**（分组）：

   - 默认值为DEFAULT_GROUP。

   - 指定配置项所属的分组。分组可以帮助组织和隔离不同应用或模块的配置。
   - 示例：`group=DEFAULT_GROUP`

4. **`file-extension`**：

   - 指定从 Nacos 配置中心加载的配置文件的文件扩展名。常见的扩展名有 `.properties` 和 `.yml`。默认值为properties。
   - 示例：`file-extension=yaml`

5. **`enabled`**：
   - 指定是否启用 Nacos 配置管理功能。默认值为true。
   - 示例：`enabled=true` 或 `enabled=false`

6. **`refresh-enabled`**：
   - 指定是否允许配置的动态刷新功能。如果为true，那么使用了@RefreshScope注解的类中的配置项的调用可以在 Nacos 配置中心的配置发生变化时，应用程序会自动刷新配置。如果为false，那么无论是否添加@RefreshScope注解，都无法进行自动刷新配置。默认值为true。
   - 示例：`refresh-enabled=true`

7. ~~**`config-import`**：~~
   - ~~指定从 Nacos 配置中心加载的配置项。可以指定配置文件的 `dataId` 或者使用通配符 `*` 来加载所有配置。~~
   - ~~示例：`config-import=nacos:gulimall-coupon.yml`~~

8. ~~**`context-path`**：~~
   - ~~指定配置项的上下文路径，用于组织配置项。~~
   - ~~示例：`context-path=application`~~

9. ~~**`timeout`**：~~
   - ~~指定从 Nacos 配置中心获取配置的超时时间（以毫秒为单位）。~~
   - ~~示例：`timeout=5000`~~

10. ~~**`max-retries`**：~~
    - ~~指定从 Nacos 配置中心获取配置失败时的最大重试次数。~~
    - ~~示例：`max-retries=3`~~

> ### 笔者经验总结
>
> （经验总结自版本：SpringBoot 2.6.13 / SpringCloud 2021.0.5 / Nacos-starter 2021.0.5.0 / Nacos-server 2.5.1，此经验大概率适配此版本以后的版本）
>
> - 在SpringBoot 2.4之后的版本中，要从远程获取配置中心中的配置文件都需要导入“spring-cloud-starter-bootstrap”这个依赖，没导入这个依赖在SpringBoot启动时会报错。
> - 建议将nacos配置中心的所有配置信息都写在bootstrap.xml / bootstrap.properties，并且在该配置文件中重新填写应用名配置信息（即spring.application.name），若不按照如此配置可能出现如下问题：
>   - nacos.config即nacos配置中心的配置信息既可以写在application.xml / application.properties，也可以写在bootstrap.xml / bootstrap.properties，目前写在这两个位置未发现区别，但是建议写在bootstrap.xml / bootstrap.properties，为了和下面说到的配置信息一同构成nacos配置中心全套配置信息。
>   - 若没有在bootstrap.xml / bootstrap.properties中添加应用名配置信息，则动态刷新无法使用，无论其他配置中心信息写在哪个配置文件都无法使用动态刷新。
>
> （如上是笔者用大量时间调试得出的血的经验...）



# 二、OpenFeign

## 1. 介绍



## 2. 快速入门

### 2.1 引入依赖

`使用远程方法的应用`的maven配置文件中添加OpenFeign依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### 2.2 准备被远程调用的方法

在`提供远程方法的应用`中添加一个实现接口方法，需要记录该接口的URL，代码如下：

```java
@RestController  // 声明是一个Controller层的Bean
@RequestMapping("/abc")  // 声明Controller内的接口的统一URL前缀
public class CouponController {
    @RequestMapping("/def")  // 声明接口的URL
    public R remoteMethod(){
		// ...
    }
}
```

同时假设`提供远程方法的应用`包含如下配置信息：

```
spring:
  application:
    name: myname
```

### 2.3 创建远程调用客户端接口

在`调用远程方法的应用`中创建一个接口，用`@FeignClient("...")`将接口标识为远程调用客户端接口，并在注解内填写`提供远程方法的应用`的应用名称（application.name，在应用的application.yml中的配置信息进行声明）。

在接口内声明与被调用的远程方法相同（方法名相同、参数相同、返回值相同）的抽象方法，并为该抽象方法添加`@RequestMapping("...")`注解，注解内填写远程方法的URL。

```java
@FeignClient("myname")
public interface FeignService {

    // 远程服务的url
    @RequestMapping("/abc/def")//注意写全优惠券类上还有映射
    public R remoteMethod();//得到一个R对象

}
```

### 2.4 开启远程调用功能

在`调用远程方法的应用`的主启动类添加`@EnableFeignClients(basePackages="...")`，并在注解内的`basePackages`参数填写所有远程调用客户端接口所在的全路径名，这一步会使得启动SpringBoot后OpenFeign自动创建一个远程调用客户端接口FeignService（接口名称可根据业务自定义）的代理类放入`调用远程方法的应用`的IoC容器中。

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages="com.xmh.gulimall.member.feign")//扫描接口方法注解
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(GulimallMemberApplication.class, args);
    }
}
```

### 2.5 远程方法调用

我们只需要在需要调用到远程方法的类中创建属性添加@Autowired注解进行自动注入，然后调用该代理类的方法即可调用到对应的远程方法。



# 三、Gateway

## 1. 介绍



## 2. 核心概念

* **路由（Route）**：这是网关的基石。路由由一个ID、一个目标地址（URI）、一组断言（Predicates）和一组过滤器（Filters）组成。当所有断言组合起来的结果为“真”的时候，这条路由就会被匹配到。
* **断言（Predicate）**：这是基于Java 8的函数式判断。它的输入是Spring框架中的ServerWebExchange对象。你可以用它来检查HTTP请求中的任何内容，比如请求头或者请求参数，看看它们是否符合你设定的条件。
* **过滤器（Filter）**：这些是通过特定工厂创建的GatewayFilter实例。你可以在把请求发送到下游服务之前或者收到下游服务的响应之后，用过滤器来修改请求或者响应的内容。



## 3. 运行原理

![spring_cloud_gateway_cbskja](E:\各种资料\Java开发笔记\我的笔记\images\spring_cloud_gateway_cbskja.png)
