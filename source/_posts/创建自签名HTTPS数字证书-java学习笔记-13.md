---
title: 创建自签名HTTPS数字证书-java学习笔记(13)
date: 2024-12-29 19:50:06
tags:
---

也许有的同学觉得前后端项目都部署在同一台服务器主机上面，前后端项目之间不需要使用HTTPS协议，其实不然。如果前后端项目都部署在同一台主机上面，该主机的负载会偏大，所以通常的做法是把前后端项目部署在不同主机上面。这种情况下，局域网之内的数据传输也应该加密，所以给后端项目配置HTTPS协议是必要的。

<!-- more -->

开通HTTPS协议需要使用数字证书，然而申请数字证书需要有公网域名。目前大健康项目处在开发阶段，我们可以自己创建一个数字证书，开通HTTPS协议。将来我们的项目上线的时候，再去申请正式的数字证书就行了。

在命令行界面执行下面的指令，生成一个365天有效期的数字证书文件。

```bash
keytool -genkey -alias jetty -dname CN=demo,OU=demo,O=demo,L=SHH,ST=SHH,C=CN -storetype PKCS12 -keyalg RSA -keysize 2048 -validity 365 -keystore jetty.p12
```

将生成的数字证书文件复制到项目的resources目录中，然后在application.yml文件中，配置数字证书文件的路径。

```yaml
server:
  ssl:
    enabled: true
    key-store-type: PKCS12
    key-store: classpath:jetty.p12
    key-store-password: abc123456
```

打开浏览器访问Web方法，访问 <https://localhost:7700/his-api/test/xss> 地址。能看到如下的画面。
因为是自签名的数字证书，不被浏览器认可，需要我们手动允许浏览器信任该数字证书。
