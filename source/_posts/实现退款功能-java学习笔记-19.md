---
title: 实现退款功能-java学习笔记(19)
date: 2025-09-02 20:35:24
tags:
  - java
---

## 时序图


{% asset_img image1.png 实现退款功能 %}

<!-- more -->

## 退款API接口说明

微信支付3.0的官方文档讲了调用退款API接口需要提供哪些参数，返回的响应中有什么参数。具体的参数，我列举到下面的表格。

### 请求参数
 
| 序号 | 参数 | 类型 | 描述 |
| --- | --- | --- | --- |
| 1 | transaction_id | string | 微信支付单编号 |
| 2 | out_trade_no | string | 订单流水号 |
| 3 | out_refund_no | string | 退款流水号（由我们自己生成） |
| 4 | amount | Object | 金额对象 |
| 4.1 | amount.refund | int | 退款金额 |
| 4.2 | amount.total | int | 订单金额 |

### 响应参数


| 序号 | 参数 | 类型 | 描述 |
| --- | --- | --- | --- |
| 1 | refund_id | string | 退款单编号 |
| 2 | transaction_id | string | 付款单编号 |
| 3 | out_trade_no | string | 订单流水号 |
| 4 | status | string | 退款状态：SUCCESS(退款成功)、CLOSED(退款关闭)、PROCESSING(退款处理中)、ABNORMAL(退款异常) |


## 编写持久层代码

在 `orderdao.xml` 文件中，声明SQL语句。

```xml
<mapper namespace="orderdao">
  <select id="searchAlreadyRefund" parameterType="int" resultType="String">
    SELECT out_refund_no AS outRefundNo
    FROM tb_order
    WHERE id = #{id}
  </select>

  <select id="searchRefundNeeded" parameterType="Map" resultType="HashMap">
    SELECT transaction_id AS transactionId,
           amount AS amount
    FROM tb_order
    WHERE id = #{id}
      AND status = 3
      AND customer_id = #{customerId}
  </select>

  <update id="updateOutRefundNo" parameterType="Map">
    UPDATE tb_order
    SET out_refund_no = #{outRefundNo},
        refund_date = CURRENT_DATE(),
        refund_time = NOW()
    WHERE id = #{id}
      AND status = 3
  </update>
</mapper>
```

### 编写 DAO 接口

在 `com.example.his.api.db.dao.OrderDao.java` 接口中，声明抽象方法：

```java
package com.example.his.api.db.dao;

import java.util.HashMap;
import java.util.Map;

public interface OrderDao {
  public String searchAlreadyRefund(int id);
  public HashMap searchRefundNeeded(Map param);
  int updateOutRefundNo(Map param);
}
```

## 编写Web层代码

在 `com.example.his.api.front.controller.form` 包中，创建 `RefundForm.java` 类：

```java
package com.example.his.api.front.controller.form;

import lombok.Data;
import javax.validation.constraints.Min;
import javax.validation.constraints.NotNull;

@Data
public class RefundForm {
  private Integer customerId;

  @NotNull
  @Min(value = 1, message = "id不能小于1")
  private Integer id;
}
```

在 `com.example.his.api.front.controller` 包中，创建 `OrderController.java` 类，声明 Web 方法：

```java
package com.example.his.api.front.controller;

import com.example.his.api.front.controller.form.RefundForm;
import cn.dev33.satoken.annotation.SaCheckLogin;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;
import javax.validation.Valid;
import java.util.Map;
import cn.hutool.core.bean.BeanUtil;

@RestController("FrontOrderController")
@RequestMapping("/front/order")
@Slf4j
public class OrderController {

  @PostMapping("/refund")
  @SaCheckLogin(type = StpCustomerUtil.TYPE)
  public R refund(@RequestBody @Valid RefundForm form) {
    int customerId = StpCustomerUtil.getLoginIdAsInt();
    form.setCustomerId(customerId);
    Map param = BeanUtil.beanToMap(form);
    boolean bool = orderService.refund(param);
    return R.ok().put("result", bool);
  }
}

```

## 编写业务层代码

在 `com.example.his.api.front.service` 包 `PaymentService.java` 接口中，声明抽象方法：

```java
package com.example.his.api.front.service;

public interface PaymentService {
  String refund(String transactionId, Integer refund, Integer total, String notifyUrl);
}
```

在 `com.example.his.api.front.service.impl` 包 `PaymentServiceImpl.java` 类中，实现抽象方法：

```java
package com.example.his.api.front.service.impl;

import com.example.his.api.front.service.PaymentService;

public class PaymentServiceImpl implements PaymentService {
  @Override
  public String refund(String transactionId, Integer refund, Integer total, String notifyUrl) {
    RefundParams params = new RefundParams();
    params.setTransactionId(transactionId);
    String outRefundNo = IdUtil.simpleUUID(); // 生成退款流水号
    params.setOutRefundNo(outRefundNo);
    params.setNotifyUrl(notifyUrl);

    RefundParams.RefundAmount amount = new RefundParams.RefundAmount();
    amount.setRefund(refund); // 退款金额
    amount.setTotal(total);   // 订单金额
    amount.setCurrency("CNY");
    params.setAmount(amount);
    
    WechatResponseEntity<ObjectNode> entity = wechatApiProvider.directPayApi("his-vue", params);
    if (!entity.is2xxSuccessful()) {
      log.error("退款失败", entity.getBody());
      return null;
    }

    ObjectNode body = entity.getBody();
    // 判断退款中状态
    if ("PROCESSING".equals(body.get("status").textValue())) {
      return outRefundNo;
    }

    return null;
  }
}
```

### 编写订单业务实现

在 `com.example.his.api.front.service.impl` 包 `OrderServiceImpl.java` 类中，实现抽象方法：

```java
package com.example.his.api.front.service.impl;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.Map;

@Service("FrontOrderServiceImpl")
@Slf4j
public class OrderServiceImpl implements OrderService {
  private String refundNotifyUrl = "/front/order/refundCallback";

  @Override
  @Transactional
  public boolean refund(Map param) {
    // 先查询订单是否存在退款流水号，避免用户重复申请退款
    int id = MapUtil.getInt(param, "id");
    String outRefundNo = orderDao.searchAlreadyRefund(id);
    // 判断该订单是否申请退款了
    if (outRefundNo != null) {
      return false;
    }
    // 查询退款所需参数
    HashMap map = orderDao.searchRefundNeeded(param);
    String transactionId = MapUtil.getStr(map, "transactionId");
    String amount = MapUtil.getStr(map, "amount");

    int total = NumberUtil.mul(amount, "100").intValue(); // 总金额
    // int refund = total;  // 退款金额
    int refund = 1; // 退款1分钱

    if (transactionId == null) {
      log.error("transactionId不能为空");
      return false;
    }

    // 执行退款
    outRefundNo = paymentService.refund(transactionId, refund, total, refundNotifyUrl);
    param.put("outRefundNo", outRefundNo);
    if (outRefundNo != null) {
      // 更新退款流水号和退款日期时间
      int rows = orderDao.updateOutRefundNo(param);
      if (rows == 1) {
        return true;
      }
    }

    return false;
  }
}
```
