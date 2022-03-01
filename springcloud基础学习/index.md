# SpringCloud基础学习


# SpringCould学习

## 前言

### 版本选择

```yaml
SpringCloud: Hoxtom.SR1
SpringBoot: 2.2.2RELEASE
CloudAibaba: 2.1.0RELEASe
Java: 1.8
Maven: 3.5及以上
Mysql: 5.7及以上
```

## 模块构建

> 新建项目

1. 选择maven创建

2. 选择骨架

   ![image-20211002163136341](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002163136341.png)

3. 修改字符编码

   **都改成utf-8**

   ![image-20211002163349833](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002163349833.png)

4. 注解生效激活

   ![image-20211002163559775](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002163559775.png)

5. 设置java编译版本

6. File Type过滤(将项目下显示的一些杂七杂八的文件过滤掉)

7. 将src删掉

8. 将pom.xml中加入如下

   ![image-20211002165209310](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002165209310.png)

9. 统一管理jar包版本

   ```xml
   <!--统一管理Jar包版本-->
     <properties>
       <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
       <maven.compiler.source>12</maven.compiler.source>
       <maven.compiler.target>12</maven.compiler.target>
       <junit.version>4.12</junit.version>
       <lombok.version>1.18.10</lombok.version>
       <log4j.version>1.2.17</log4j.version>
       <mysql.version>5.1.47</mysql.version>
       <druid.version>1.1.16</druid.version>
       <mybatis.spring.boot.version>1.3.0</mybatis.spring.boot.version>
     </properties>
   ```

10. 编写dependencyManagement

    **作用**:

    - 统一声明版本号
    - 升级时只需该父工程的pom.xml中的版本号就可以处处升级
    - 子模块不用写版本号
    - 如果子模块要使用别的版本，只用自己声明version

    ```xml
    <!--子模块继承之后，提供作用：锁定版本+子module不用写groupId和version-->
      <dependencyManagement><!--定义规范，但不导入-->
        <dependencies>
          <dependency>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-project-info-reports-plugin</artifactId>
            <version>3.0.0</version>
          </dependency>
          <!--spring boot 2.2.2-->
          <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.2.2.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
          </dependency>
          <!--spring cloud Hoxton.SR1-->
          <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR1</version>
            <type>pom</type>
            <scope>import</scope>
          </dependency>
          <!--spring cloud 阿里巴巴-->
          <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2.1.0.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
          </dependency>
          <!--mysql-->
          <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
            <scope>runtime</scope>
          </dependency>
          <!-- druid-->
          <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>${druid.version}</version>
          </dependency>
          <!--mybatis-->
          <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>${mybatis.spring.boot.version}</version>
          </dependency>
          <!--junit-->
          <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>${junit.version}</version>
          </dependency>
          <!--log4j-->
          <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>${log4j.version}</version>
          </dependency>
        </dependencies>
      </dependencyManagement>
    ```

11. 增加热启动插件

    ```xml
    <!--热启动插件-->
      <build>
        <plugins>
          <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
              <fork>true</fork>
              <addResources>true</addResources>
            </configuration>
          </plugin>
        </plugins>
      </build>
    ```

12. maven中跳过单元测试

    ![image-20211002170708015](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002170708015.png)

13. 父工程创建完成后执行 mvn:install将父工程发布到仓库方便子工程继承

---

### 支付模块构建

![image-20211002171500936](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002171500936.png)

#### 构建微服务模块统一步骤

1. 建module
2. 改pom
3. 写yml
4. 主启动
5. 业务类

---

#### 创建支付模块

1. 创建module，继承自父亲模块(maven无骨架创建)

2. 改pom文件

   把需要使用的pom文件导入

   ```xml
   <dependencies>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-actuator</artifactId>
           </dependency>
           <dependency>
               <groupId>org.mybatis.spring.boot</groupId>
               <artifactId>mybatis-spring-boot-starter</artifactId>
           </dependency>
           <dependency>
               <groupId>com.alibaba</groupId>
               <artifactId>druid-spring-boot-starter</artifactId>
               <version>1.1.10</version>
           </dependency>
           <!--mysql-connector-java-->
           <dependency>
               <groupId>mysql</groupId>
               <artifactId>mysql-connector-java</artifactId>
           </dependency>
           <!--jdbc-->
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-jdbc</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-devtools</artifactId>
               <scope>runtime</scope>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
           </dependency>
       </dependencies>
   ```

3. 改yml

   - 在模块下的resources下新建application.yaml

     ```yaml
     server:
       port: 8001 #服务端口号
     
     spring:
       application:
         name: cloud-payment-service #服务名称
       datasource:
         type: com.alibaba.druid.pool.DruidDataSource #当前数据源操作类型
         driver-class-name: org.gjt.mm.mysql.Driver #mysql驱动包
         url: jdbc:mysql://${person.mysql.host}:${person.mysql.port}/springcloud?useUnicode-true&charcterEncoding=utf-8&useSSL=false
         username: root
         password: lcy021030
     
     person:
       mysql:
         host: 49.234.111.177
         port: 3306
     ```

4. 主启动类

   ![image-20211002173808036](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002173808036.png)

   创建主启动类

   ![image-20211002173930824](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211002173930824.png)

5. 业务类

   >数据库建表

   ```shell
    CREATE TABLE `payment` (  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',  `serial` varchar(200) DEFAULT '',  PRIMARY KEY (`id`)) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8
   ```

   >创建实体类

   ```java
   @NoArgsConstructor
   @AllArgsConstructor
   @Data
   public class Payment implements Serializable {
   
       private Long id;
   
       private String serial;
   }
   ```

   

   ```java
   @Data
   @AllArgsConstructor
   @NoArgsConstructor
   public class CommonResult<T> {
   
       public static final Integer SUCCESS = 20000;
   
       public static final Integer ERROR = 40000;
   
       private Integer code;
   
       private String message;
   
       private T data;
   
       public static CommonResult ok(){
           return new CommonResult(SUCCESS,"success",null);
       }
   
       public static CommonResult error(){
           return new CommonResult(ERROR,"error",null);
       }
   
       public CommonResult msg(String message){
           this.message = message;
           return this;
       }
   
       public CommonResult code(int code){
           this.code = code;
           return this;
       }
   
       public CommonResult<T> data(T data){
           this.data = data;
           return this;
       }
   }
   ```

   > 编写PaymentDao层

   > 编写PaymentMapper

   **文件头**

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
   
   <mapper namespace="pro.risingsun.springcloud.dao.PaymentDao">
       
   </mapper>
   ```

   > 写Service

   >写controller

   > postman自测

#### 创建消费者模块

>创建模块

> 写实体类(Payment和CommonResult)

> 使用RestTemplate

##### RestTemplate

1. 创建config.ApplicationContexConfig

   ![image-20211003113322154](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003113322154.png)

2. 将RestTemplate注册成bean

> 完善消费者的OrderConsumer

​	![image-20211003114013088](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003114013088.png)

> 测试

---

### 工程重构

将部分相似的代码进行重构整合

> 创建新模块cloud-api-commons

> 写pom

![image-20211003122500391](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003122500391.png)

> 将entites移入

> 放入本地库

`maven:clean`

`maven:install`

>改造原本的两个模块的公共内容

- 删掉entities

- 在各自的pom文件中引入

  ![image-20211003123314000](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003123314000.png)

---

## Eureka

一种服务注册中心，使用Eureka的客户端连接到到Eureka Server并维持心跳连接。这样就可以通过Eureka Server来监控系统的各个微服务是否正常运行

**EurekaServer**

提供注册服务，各个微服务节点通过配置启动后，会从EurekaServer中心进行注册。这样RurekaSever中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观看到

**EurekaClient**

一个Java客户端，用于简化EurekaServer的交互，客户端同时也具备一个内置的、使用轮询负载算法的负载均衡器。在应用启动后，将会用EurekaServer发送心跳(默认30秒为一个周期)。如果EurekaServer在多个心跳周期内没有接收到某个节点的心跳，EurekaServer就会从服务注册表中把这个服务节点移除(默认90秒)

### IDEA创建EurekaServer

> 创建模块

> 改pom

```xml
<dependencies>
    <!--自定义的api通用包-->
    <dependency>
        <groupId>pro.risingsun.springcloud</groupId>
        <artifactId>cloud-api-commons</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
    <!--eureka-server-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

> 写yaml

```yaml
server:
  port: 7001

eureka:
  instance:
    hostname: localhost #eureka服务端的实例名称
  client:
    register-with-eureka: false #不注册自己
    fetch-registry: false #false表示自己就是注册中心,不需要去检索自己的服务
    service-url: 
      #设置与EurekaServer交互的地址查询服务和注册服务都需要依赖这个地址
      defaultzone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

> 写启动类

![image-20211003140501015](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003140501015.png)

​		**打上注解@EnbaleEurekaServer表示自己是EurekaServer**

> 测试

输入localhost:7001

---

### 将payment注册进Eureka中

> 改pom

增加EurekaClient的依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

> 改配置

```yaml
eureka:
  client:
    #表示自己注册进EurekaServer
    register-with-eureka: true
    #是否从EurekaServer抓取已有的配置信息,默认为true(单节点无所谓,集群必须设置为true才可以配置ribbon使用负载均衡)
    fetch-registry: true
    service-url: 
      defaultZone: http://localhost:7001/eureka #EurekeServer的地址
```

> 改主启动类

加上注解`@EnableEurekaClient`

![image-20211003141734790](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003141734790.png)

>启动测试 

![image-20211003142127016](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003142127016.png)

**这里的应用名称是我们在yaml文件中配置的,如下图:**

![image-20211003142202567](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003142202567.png)

---

### 将order注册进Eureka中

步骤如上，基本一致



---

### 集群Eureka搭建

**互相注册，相互守望**

![image-20211003144311133](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003144311133.png)

> 创建cloud-eureka-server7002

参考7001进行创建

> 修改映射文件

![image-20211003145905291](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003145905291.png)

> 写yaml

互相注册，相互守望

1. 修改7001的配置文件

   ![image-20211003150049850](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003150049850.png)

2. 修改7002的配置文件

   ![image-20211003150203275](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003150203275.png)

> 主启动类

> 测试访问`eureka7001:7001`

![image-20211003150653411](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003150653411.png)

> 测试访问`eureka7002:7002`

![image-20211003150741619](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003150741619.png)

---

### 将服务注册进集群

> 改配置文件

![image-20211003151030412](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003151030412.png)

---

### PaymentProvider集群搭建

参考8001搭建8002和8003

> 修改consumer的服务调用URL

![image-20211003153614901](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003153614901.png)

**但是这样消费者还是找不到是哪个生产者，因为三个生产者的服务名称都是cloud-payment-service，因此需要开启负载均衡**

> 开启负载均衡

使用`@LoadBalances注解赋予RestTemplate负载均衡能力`

![image-20211003160759256](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003160759256.png)

---

#### 集群信息完善

> 增加实例名称

在payment8001和8002的配置文件中增加

![image-20211003164120964](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003164120964.png)

> 增加ip显示

![image-20211003164345463](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003164345463.png)

---

### 服务发现

对于注册进eureke里面的微服务，可以通过服务发现来获得该服务的信息

> 修改cloud-provider-payment8001的Controller

![image-20211003165853936](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003165853936.png)

> 主启动类增加注解

![image-20211003170023899](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003170023899.png)

> 测试

![image-20211003170104407](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003170104407.png)

---

### Eureka自我保护

某时刻某一个微服务不可用了，Eureka不会立刻清理，仍旧会对该微服务的消息进行保存

**属于CAP中的AP分支**

---

## ZooKeeper

### 创建payment

> 服务器上启动Zookeeper

>新建模块cloud-provider-payment8004

> 改pom文件

将Eureka的依赖改成zookeeper的

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
</dependency>
```

> 改配置文件(将Eureka的改成Zookeeper)

![image-20211003174617468](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003174617468.png)

> 主启动类

![image-20211003174818725](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003174818725.png)

> 写Controller

> 测试localhost:8004/payment/zk

​	![image-20211003180251839](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003180251839.png)

---

### 创建order

...省略

---

## Consul

...

---

## Ribbon

Ribbon是一套实现负载均衡的工具

`spring-cloud-starter-netflix-eureka-client`已经集成了Ribbon

> 如何替换负载均衡策略

该配置类不能放在@CompomentScan可以扫到的位置

![image-20211003202754227](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003202754227.png)

主启动类增加注解

![image-20211003202904121](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211003202904121.png)

---

## OpenFeign

用在消费端

> 创建模块cloud-consumer-feign-order9000

> 改pom

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>pro.risingsun.springcloud</groupId>
        <artifactId>cloud-api-commons</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
    <!--eureka-client-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <!--openfeign客户端-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

> 改配置文件

```yaml
server:
  port: 9000
spring:
  application:
    name: cloud-order-service
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001:7001/eureka,http://eureka7002:7002/eureka #EurekeServer的地址(集群版)
```

**注册不注册都可以，消费端本来就不用注册**

> 写主启动类

![image-20211004110205347](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004110205347.png)

> 写服务调用接口

![image-20211004111251813](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004111251813.png)

**要点：**

- 这里相当于是直接根据服务名和路径去服务端找相应的controller的方法，所以这里的服务名、路径、请求类型要一致才能在服务端找到方法
- 参数也要保证一致，而且需要打上相应的注解，没有SpringMVC帮忙封装了

> 负载均衡

自带负载均衡，使用Ribbon

### OpenFeign超时控制

OpenFeign默认等待一秒钟

如果超过1秒种，Feign客户端直接报错

> 修改超时时间(OpenFeign自带Ribbon)

![image-20211004114857219](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004114857219.png)

---

### OpenFeign日志

日志级别:

- NONE：默认的，不显示任何日志
- BASIC：仅记录请求方法、URL、相应状态码及执行时间
- HEADERS：除了记录BASIC中定义的信息外，还有请求和相应的头信息
- FULL：除了HEADERS中定义的信息外，还有请求和相应的正文及元数据

#### 开启日志

> 创建日志配置类

![image-20211004115533042](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004115533042.png)

> 配置文件中配置日志

![image-20211004115757543](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004115757543.png)

---

## Hystix

### 重要概念

> 服务 降级

**fallback**

出现情况

- 程序运行异常
- 超时
- 服务熔断出发服务降级
- 线程池/常量池打满

> 服务熔断

当达到最大服务访问时，直接拒绝访问，然后调用服务降级

> 服务限流

限制服务访问

### 实践

#### 创建生产者

> 新建模块cloud-provider-hystrix-payment8005

> pom文件

```xml
<!--hystrix依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

> 写配置文件

> 写主启动类

![image-20211004140300990](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004140300990.png)

> 写Service

![image-20211004141602440](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004141602440.png)

> 写Controller

![image-20211004141621307](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004141621307.png)

> 测试

---

#### 创建消费者

...

> service，远程调用

![image-20211004150654220](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004150654220.png)

> controller

![image-20211004150718627](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004150718627.png)

---

#### 服务降级

##### 服务提供方降级

> 激活服务降级功能

在主启动类上加上注解`@EnbaleCircuitBeaker`

![image-20211004155305943](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004155305943.png)

使用注解`@HystrixComman`在业务方法上标记

![image-20211004155318910](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004155318910.png)

---

##### 服务消费方降级

> 改yml

![image-20211004164102888](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004164102888.png)

> 改启动类

![image-20211004170031979](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004170031979.png)

> 业务上处理

![image-20211004170313796](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004170313796.png)

---

### 全局服务降级

现在的降级方式和业务代码耦合严重，而且重复冗余。那么需要进行全局的一个服务降级配置

> @DefaultProerties(defaultFallback = "xxx")设置默认的服务降级配置

**如果某个方法有自己的配置，就用自己的，没有就用默认的**

![image-20211004171425533](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004171425533.png)

> 在Feign接口进行统一处理

> 写一个类实现PaymentFeignService接口，对方法进行重写

![image-20211004175255974](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004175255974.png)

> 配置fallback

![image-20211004175318614](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004175318614.png)

---

### 服务熔断

> 生产方Service代码如下

![image-20211004184946223](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004184946223.png)

> 在错误了一定次数内失败率达到了设定值，那么进入熔断状态，然后在一定时间后(默认是5秒)会放出几个请求进去，如果还是错误降级了，那么就保持熔断状态，刷新休眠窗口期。如果请求正确了，那么熔断取消，正常运行。

![image-20211004185531341](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004185531341.png)

> 参考链接

- https://blog.csdn.net/loushuiyifan/article/details/82702522

> 参数说明

![image-20211004190613982](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004190613982.png)

1. 请求总数阈值：在快照时间窗内，必须满足请求总数阈值才有资格熔断。
2. 快照时间窗：断路器通过在统计一定时间内的请求和错误等数据决定是否开启熔断，而统计的时间范围就是快照时间窗，默认就是最近10秒
3. 错误百分比阈值，当请求总数在快照时间窗内超过了阈值，而且错误请求的次数占的百分比超过了该错误百分比阈值，那么就会将断路器打开

---

### 监控页面

> 建模块

> 改pom

增加：

```xml
<!--hystrix仪表盘图形化页面-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

> 改yml

> 写主启动类

![image-20211004194235802](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004194235802.png)

> 启动，访问localhost:9001/hystrix

> 修改8005

![image-20211004200922650](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004200922650.png)

> 监控8005

> 访问监控网址，测试

![image-20211004201520979](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004201520979.png)

---

## GateWay

### 入门配置

> 创建模块

![image-20211004205413699](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004205413699.png)

> 改pom

```xml
<!--gateway网关-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

> 写yml

```yml
server:
  port: 9527
spring:
  application:
    name: cloud-gateway
eureka:
  instance:
    hostname: cloud-gateway-service
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url: 
      defaultZone: http://eureka7001:7001/eureka,http://eureka7002:7002/eureka #EurekeServer的地址(集群版)
```

> 写主启动类

![image-20211004210219955](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004210219955.png)

> 写配置

---

### 动态路由

> 配置文件配置

```yml
server:
  port: 9527
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true #开启从服务注册中心中动态创建路由的功能,利用微服务名进行路由
      routes:
        - id: payment_routh #配置路由的id,没有固定规则,但是要求唯一
          uri: lb://cloud-payment-service #匹配后提供的服务地址
          predicates:
            - Path=/payment/** #断言,根据路径进行路由
          ##################
        - id: payment_routh2
          uri: lb://cloud-payment-service #匹配后提供服务的地址
          predicates:
            - Path=/service #断言,根据路径进行路由
eureka:
  instance:
    hostname: cloud-gateway-service
  client:
    #表示自己注册进EurekaServer
    register-with-eureka: true
    #是否从EurekaServer抓取已有的配置信息,默认为true(单节点无所谓,集群必须设置为true才可以配置ribbon使用负载均衡)
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001:7001/eureka,http://eureka7002:7002/eureka #EurekeServer的地址(集群版)
```

> 访问localhost:9527/payment/1测试

---

### 常用Predicate

也就是常用的断言配置

![image-20211004215344211](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004215344211.png)

#### After Route Predicate

请求时间在该时间之后可以匹配

> 在配置文件中配置

![image-20211004220110267](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004220110267.png)

**可以通过如下Api进行获得该格式的日期:**

![image-20211004215711880](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004215711880.png)

> 在该日期之前无法访问

![image-20211004215837978](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004215837978.png)

> 该日期之后可以访问

![image-20211004220120328](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004220120328.png)

---

#### Before Route Predicate

请求的时间在这个时间之前可以匹配

#### Between Route Predicate

请求时间在两个时间内，可以匹配

![image-20211004220344077](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211004220344077.png)

#### Cookie Route Predicate

匹配Cookie断言

需要两个参数，第一个是Cookie name，第二个是政策表达式

Gateway会去检验对应的Cookie的value是否符合正则表达式

> 配置Predicate

![image-20211005112725758](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005112725758.png)

![image-20211005112702695](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005112702695.png)

---

#### Header Route Predicate

请求头中进行匹配

两个参数，一个是请求头名，一个是值

![image-20211005120128556](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005120128556.png)

> 测试不匹配

![image-20211005120212339](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005120212339.png)

> 测试匹配

![image-20211005120232923](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005120232923.png)

---

#### Method Route Predicate

#### Path Route Predicate

#### Query Route Predicate

---

### Filter

#### 自定义全局日志过滤器

![image-20211005122258670](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005122258670.png)

---

## Config

### 服务端(连接github)

微服务统一配置

> 建模块cloud-config-center-3344

> 写pom

增加

```xml
<!--配置中心-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

> 写配置文件

![image-20211005130747400](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005130747400.png)

> 主启动类

![image-20211005130442556](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005130442556.png)

> 测试localhost:3344/master/config-dev.yml

![image-20211005134056447](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005134056447.png)

> 如果连接失败，使用ssh方式连接，使用旧方法创建私匙，在github中添加，然后配置文件中改成ssh的地址

---

### 客户端(连接服务端)

> 创建模块cloud-config-client-3355

> 改pom

```xml
<!--配置中心客户端-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
```

> 写bootstrap.yml

- application.yml是用户级的资源配置
- bootstrap.yml是系统级，**优先级更高**

![image-20211005141218298](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005141218298.png)

> 写主启动类

![image-20211005135833611](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005135833611.png)

---

### 客户端动态更新

服务端可以实时更新，但是客户端不能实时更新

> 引入监控依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

> 配置文件暴露监控端点

![image-20211005141604377](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005141604377.png)

> 在controller添加注解`@RefreshScope`

![image-20211005141656387](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005141656387.png)

---

## Bus

> 什么是总线？

在微服务架构的系统总，通常用使用轻量级的消息代理来构建一个公用的消息主题，并让系统中所有微服务的实例都连接上来，由于该主题中产生的消息会被所有实例监听和消费，所以称它为消费总线。在总线上的各个实例，都可以方便的广播一些需要其他连接到该主题上的实例都知道的消息

> 基本原理

ConfigClient实例都监听MQ中同一个topic(默认是SpringCloudBus)。当一个服务刷新数据的时候，它会把这个消息放入到Topic中，这样其他监听同一Topic的服务就能得到通知，然后去更新自身的配置

### 实践

再创建一个cloud-config-client-3366，步骤略

> 设计

- 利用消息总线触发一个客户端/bus/refresh，而刷新所有客户端配置
- 利用消息总线触发一个服务端ConfigServer的/bus/refresh端点，而刷新所有客户端的配置

**方法二更适合**

> 服务端添加Rabbitmq

```xml
<!--整合rabbitmq的bus消息总线-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
        </dependency>
```

> 增加rabbitmq的配置

![image-20211005152750355](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005152750355.png)

> 增加暴露刷新配置的端点

![image-20211005152849009](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005152849009.png)

> 3344和3355添加消息总线的支持

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```yml
rabbitmq:
  host: 49.234.111.177
  port: 5672
  username: lcy
  password: lcy021030
```

> 发送请求POST: localhost:3344/actuator/bus-refresh完成自动更新

---

## Stream

驱动，屏蔽底层消息中间件的差异

应用程序通过inputs或者outputs来与SpringCloudStream中的binder对象交互

通过配置来绑定(binding)，而SpringCloudStream的binder对象负责与消息中间件交互

### 实践

#### 生产者

> 创建模块cloud-stream-rabbitmq-provider8801

> 写pom

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

> 写yml

```yml
server:
  port: 8801

spring:
  application:
    name: cloud-stream-provider
  cloud:
    stream:
      bindings: # 服务的整合处理
        output: # 这个名字是一个通道的名称
          destination: studyExchange # 表示要使用的Exchange名称定义
          content-type: application/json # 设置消息类型，本次为json，文本则设置“text/plain”
          binder: rabbit1 # 设置要绑定的消息服务的具体设置
      #可以配置多个底层MQ的配置
      binders: # 在此处配置要绑定的rabbitmq的服务信息；
        rabbit1: # 表示定义的名称，用于于binding整合
          type: rabbit # 消息组件类型
          environment: # 设置rabbitmq的相关的环境配置
            spring:
              rabbitmq:
                host: 49.234.111.177
                port: 5672
                username: lcy
                password: lcy021030
  rabbitmq:
    host: 49.234.111.177
    port: 5672
    username: lcy
    password: lcy021030
eureka:
  client:
    service-url:
      defaultZone: http://eureka7001:7001/eureka,http://eureka7002:7002/eureka #EurekeServer的地址(集群版)
  instance:
    instance-id: send-8801.com #在信息列表显示的主机名称
    lease-renewal-interval-in-seconds: 2 #设置心跳的时间(默认是30秒)
    lease-expiration-duration-in-seconds: 5
    prefer-ip-address: true
```

> 写主启动类

> 写业务接口

![image-20211005173105499](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005173105499.png)

![image-20211005173115598](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005173115598.png)

---

#### 消费者

> 新建模块cloud-stream-rabbitmq-consumer8802

> 写pom

> 写yml

![image-20211005173534691](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211005173534691.png)

> 写业务

---

### 重复消费

此时两个消费者是不同组的，都监听一个交换机，所以都会收到消息，因此我们需要将两个消费者放到同一个组中

> 自定义分组

如果我们不自定义分组，那么默认使用的是不同的分组，因此可以自定义选择分组

![image-20211006154102316](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211006154102316.png)

在8802和8803中加入分组，那么收到的就是将两个消费者监听在同一个组中，消息则只能被消费一次

此时两个消费者就轮询接受消息。

> 原理

其实就是rabbitmq中的fandout和direct，默认情况两个消费者是不同的队列，然后绑定到同一个交换机，那么我们生产者生产一个消息到交换机，就会被两个队列都拿到。

当我们配置了组名时，其实就是变成了同一个队列，那么生产的消息发送到交换机时，交换机只发了一条消息给队列，那么监听同一个队列的消费者，只有一个能消费到消息，默认是轮询消费

---

## Sleuth

**SpringCloudSleuth提供了分布式追踪的解决方案**

### 搭建zipkin

> 下载jar包后直接启动

![image-20211006160619640](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211006160619640.png)

> 访问`localhost:9411`查看面板



### 实践

> cloud-provider8001添加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

> 改配置

![image-20211006161043960](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20211006161043960.png)

> 在consumer做一样的依赖和配置


