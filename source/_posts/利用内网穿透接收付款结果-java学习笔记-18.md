---
title: 利用内网穿透接收付款结果-java学习笔记(18)
date: 2025-05-17 09:37:28
tags:
  - java
---

## 支付通知消息内容

在微信支付的官方文档中([https://pay.weixin.qq.com/doc/v3/merchant/4012791861](https://pay.weixin.qq.com/doc/v3/merchant/4012791861))，介绍了微信服务器发送的支付结果通知消息包含了哪些内容。其中比较重要的几个参数，我列到下面的表格中了。

{% asset_img image1.png 内网穿透 %}

除了上述这几个我们能用上的参数之外，通知消息中还包含了数字签名。我们接收到通知消息之后，为了检验是否为微信服务器发送的，而不是其他人伪造的消息。我们可以用本地的微信支付密钥和数字证书，对接收到的消息重新计算数字签名，如果这个数字签名与消息附带的数字签名一致，说明该消息是微信服务器发送的。

## 编写持久层代码

在 `orderDao.xml` 文件中，声明SQL语句，更新订单为已付款状态。

```xml

```

## 微信支付回调

### 如何使用平台证书验签名

文档地址：[https://pay.weixin.qq.com/doc/v3/merchant/4013053420](https://pay.weixin.qq.com/doc/v3/merchant/4013053420)


## 编写web服务

在 `OrderController` 中，编写接收微信支付通知的接口。

```java
/**
    * 接收微信支付通知
    * 注意：此接口需要允许匿名访问，因为是微信服务器回调
    * @return 处理结果
    */
    @SneakyThrows
   @PostMapping("/wx-notify")
   public Map<String, Object> wxPayNotify(@RequestBody String notifyData, 
                                         @RequestHeader("Wechatpay-Serial") String serialNo,
                                         @RequestHeader("Wechatpay-Signature") String signature,
                                         @RequestHeader("Wechatpay-Timestamp") String timestamp,
                                         @RequestHeader("Wechatpay-Nonce") String nonce,
                                         HttpServletRequest request) throws IOException {
       
       Map<String, Object> result = new HashMap<>();

       String requestBody = request.getReader().lines().collect(Collectors.joining(System.lineSeparator()));
       ResponseSignVerifyParams params = new ResponseSignVerifyParams();
       params.setWechatpaySerial(serialNo);
       params.setWechatpaySignature(signature);
       params.setWechatpayTimestamp(timestamp);
       params.setWechatpayNonce(nonce);
       params.setBody(requestBody);

       wechatApiProvider.callback("his-vue").transactionCallback(params, data -> {
           String outTradeNo = data.getOutTradeNo();
           String transactionId = data.getTransactionId();
           // 更新订单状态为已支付
           orderService.updateOrderPaid(outTradeNo, transactionId);
           
       });
       return result;
       
       
   }
   
```


## 配置内网穿透

量子互联 (https://www.uulap.com/)[https://www.uulap.com/]



