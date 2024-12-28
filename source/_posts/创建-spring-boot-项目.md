---
title: 创建 spring boot 项目 4
date: 2024-12-21 19:40:13
tags:
---

## 思维导图

{% asset_img image1.png 思维导图 %}

### 利用Maven创建SpringBoot项目

在IDEA工具上面新建SpringBoot项目，项目的名称为 `his-api` 需要注意Java语法版本没有15，所以我们选择低版本的JDK语法，后续在 pom.xml文件中改成JDK15语法即可。

<!-- more -->

{% asset_img image2.png 创建SpringBoot项目 %}

### 配置Jetty

在pom.xml文件中配置Jetty，Jetty是SpringBoot默认的Web容器，但是Jetty的性能不如Tomcat，所以我们可以配置Jetty为Web容器。

{% asset_img image3.png 配置Jetty %}

```yaml
server:
  # 端口号
  port: 7700
  # jetty配置
  jetty:
    threads:
      # 接受连接的线程数
      acceptors: 4
      # 选择连接的线程数
      selectors: 8
      # 最小线程数
      min: 8
      # 最大线程数
      max: 200
  servlet:
    context-path: /his-api
```

### 配置Druid连接池和MyBatis

#### 配置Druid连接池

既然要用JDBC连接数据库，那么必然少不了数据库连接池。虽然Java领域有很多知名的数据库连接池产品，但是其中功能最丰富、稳定性最好的产品当属阿里巴巴出品的Druid连接池。该连接池不仅功能齐全，而且还内置了监控功能，我们可以利用Web页面查看数据库连接池的负载情况。

在 application.yml 文件中，添加Druid连接池和MyCat连接信息。如果你用的是云主机，要把 localhost 写成云主机的IP地址

```yaml
spring:
  servlet:
    multipart:
      # 是否启用多部分文件上传
      enabled: true
      # 单个文件最大大小
      max-file-size: 20MB
      # 请求最大大小
      max-request-size: 20MB
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      # 这里需要注意，MySQL8.0以上版本使用com.mysql.cj.jdbc.Driver，MySQL8.0以下版本使用com.mysql.jdbc.Driver， mycat 使用 com.mysql.jdbc.Driver，且mysql-connector-java 版本需要是5.1.47
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://localhost:8066/his?useUnicode=true&characterEncoding=utf-8&serverTimezone=UTC&useSSL=false
      username: root
      password: abc123456
      initial-size: 8
      min-idle: 8
      max-active: 16
      max-wait: 60000
      test-while-idle: true
      test-on-borrow: true
      test-on-return: false
      validation-query: SELECT 1
```

#### 配置MyBatis

在 application.yml 文件中，添加MyBatis配置信息。

```yaml
mybatis:
  mapper-locations: classpath*:mapper/*.xml
  type-aliases-package: com.jiang.his.db.pojo
  configuration:
    map-underscore-to-camel-case: true
    # 打印sql
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
logging:
  level:
    root: info
    com.jiang.his.db.dao: warn
  pattern:
    console: '%d{yyyy-MM-dd HH:mm:ss} %-5p [%c{1}] - %msg%n'
```

#### 生成MyBatis代码

接下来我们创建 com.example.his.api.db.dao 和 com.example.his.api.db.pojo 两个package，用来存放MyBatis用到的DAO接口和POJO映射类。

##### 连接MySQL

打开IDEA工具，给IDEA内置的DataGrip程序配置数据库连接。DataDrip是一种相当于Navicat的数据库客户端程序，但是功能性和兼容性不如Navicat。我们配置DataGrip的目的是让MyBatisX插件能根据数据表生成POJO类和DAO接口。

连接MySQL数据库，在IDEA工具中，点击左上角的DataGrip图标，然后点击+号，选择MySQL，输入数据库连接信息，点击测试连接，如果连接成功，点击确定。

##### 生成代码

选中所有的数据表然后点击右键，选择菜单中第一个选项，调用MyBatisX插件生成代码。

> IDEA的MyBatisX插件不能用，在这里提供一个[MyBatisX插件](https://github.com/forlornMWS/MybatisX/releases/tag/plugin)，下载后本地安装。

在MyBatis插件的面板中配置参数。比如说生成的POJO类没有 `tb` 前缀，每个POJO的结尾都要加上 `Entity` 字样。

{% asset_img image3.png 配置MyBatisX %}

点击next按钮。

{% asset_img image4.png 点击next按钮 %}

点击Generate按钮，MyBatisX插件会根据数据表生成POJO类和DAO接口。

##### 修改数据映射类型

打开生成的POJO映射类，如果属性变量是 `Date` 或者 `Object` 类型的，我们都改成 `string` 类型。首先我来解答为什么映射类中不使用Date数据类型，这是因为后端向前端返回JSON数据的时候，Date对象转化成字符串的时候结果不可控，很可能把毫秒时间和时区都给带上了，为了避免这种情况发生，所以映射类中不要有Date数据类型。至于说Object类型是怎么产生的呢?如果数据表字段是Enum类型或者JSON类型，MvBatisX插件生成代码的时候就会出现Object类型，于是需要我们自己把映射类中的Object修改成String类型。
新建一个 `db.pojo` 包，将修改后的entity类复制到 `db.pojo` 包中。
将mapper文件夹中的文件改成dao后缀文件。新建一个 `db.dao` 包，将生成的DAO接口复制到 `db.dao` 包中。

### 配置pom.xml文件

在pom.xml文件中，配置SpringBoot的版本，以及SpringBoot的依赖包。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.jiang</groupId>
    <artifactId>his</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>his</name>
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>15</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>2.6.13</spring-boot.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jetty</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <!-- mail -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
        </dependency>
        <!-- mysql -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>
        <!-- mysql连接池 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.2.15</version>
        </dependency>
        <!-- mongodb -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
        <!-- 工具类 -->
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>5.8.12</version>
        </dependency>
        <!-- http状态码 -->
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpcore</artifactId>
            <version>4.4.16</version>
        </dependency>
        <!-- 后端验证 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <!-- satoken -->
        <dependency>
            <groupId>cn.dev33</groupId>
            <artifactId>sa-token-spring-boot-starter</artifactId>
            <version>1.34.0</version>
        </dependency>
        <dependency>
            <groupId>cn.dev33</groupId>
            <artifactId>sa-token-dao-redis</artifactId>
            <version>1.34.0</version>
        </dependency>
        <dependency>
            <groupId>cn.dev33</groupId>
            <artifactId>sa-token-spring-aop</artifactId>
            <version>1.34.0</version>
        </dependency>
        <dependency>
            <groupId>cn.dev33</groupId>
            <artifactId>sa-token-alone-redis</artifactId>
            <version>1.34.0</version>
        </dependency>

        <!-- 对象池 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
            <version>2.11.1</version>
        </dependency>
        <!-- 生成二维码图片 -->
        <dependency>
            <groupId>com.google.zxing</groupId>
            <artifactId>core</artifactId>
            <version>3.3.3</version>
        </dependency>
        <!-- 微信支付 -->
        <dependency>
            <groupId>cn.felord</groupId>
            <artifactId>payment-spring-boot-starter</artifactId>
            <version>1.0.17.RELEASE</version>
        </dependency>
        <!-- 腾讯云 -->
        <dependency>
            <groupId>com.tencentcloudapi</groupId>
            <artifactId>tencentcloud-sdk-java</artifactId>
            <version>3.1.416</version>
        </dependency>
        <!-- minio -->
        <dependency>
            <groupId>io.minio</groupId>
            <artifactId>minio</artifactId>
            <version>8.2.1</version>
        </dependency>
        <!-- 文件上传 -->
        <dependency>
            <groupId>commons-fileupload</groupId>
            <artifactId>commons-fileupload</artifactId>
            <version>1.2.2</version>
        </dependency>
        <!-- okio -->
        <dependency>
            <groupId>com.squareup.okio</groupId>
            <artifactId>okio</artifactId>
            <version>2.8.0</version>
        </dependency>
        <!-- amqp-client -->
        <dependency>
            <groupId>com.rabbitmq</groupId>
            <artifactId>amqp-client</artifactId>
            <version>5.9.0</version>
        </dependency>
        <!-- amqp -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <!-- websocket -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
        <!-- 规则引擎 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>QLExpress</artifactId>
            <version>3.2.0</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-core</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-to-slf4j</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!-- poi -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>5.2.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>5.2.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-scratchpad</artifactId>
            <version>5.2.3</version>
        </dependency>
        <!-- 华为云 -->
        <!-- <dependency>
            <groupId>com.huawei.apigateway</groupId>
            <artifactId>java-sdk-core</artifactId>
            <version>3.0.2</version>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-simple</artifactId>
                </exclusion>
            </exclusions>
        </dependency> -->
        <!-- 腾讯云 -->
        <dependency>
            <groupId>com.github.tencentyun</groupId>
            <artifactId>tls-sig-api-v2</artifactId>
            <version>2.0</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.huaweicloud.sdk</groupId>
            <artifactId>huaweicloud-sdk-core</artifactId>
            <version>3.1.87</version>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>15</source>
                    <target>15</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot.version}</version>
                <configuration>
                    <mainClass>com.jiang.his.HisApplication</mainClass>
                    <skip>true</skip>
                </configuration>
                <executions>
                    <execution>
                        <id>repackage</id>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    

</project>
```
