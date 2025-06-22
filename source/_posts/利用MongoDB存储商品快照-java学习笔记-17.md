---
title: 利用MongoDB存储商品快照-java学习笔记(17)
date: 2025-05-06 20:45:40
tags:
  - java
---

## 为什么用MongoDB存储快照?

其实商品快照就是订单商品的当前信息，我们把该商品信息从 tb_goods 表中读取出来，然后存储到快照表中。你用MSQL中的数据表存储快照也是可以的，但是日积月累快照记录越来越多(2000万以上)，MySQL数据表就显得很吃力了。因此我们可以选择适合存储海量数据(TB级别)的MongoDB数据库，而且它的读写速度比MySQL快多了。当然了，还有比MongoDB存储更多数据(PB级别)的HBase数据库，但是用在我们的项目上就显得大材小用了。

## MongoDB的集合与文档

由于MongoDB没有数据表结构，也不支持SQL语句。它里面的数据是JSON格式的，被称作文档，然后保存在集合中。也就是说，集合相当于MySQL的数据表，文档相当于数据记录。

<!-- more -->

### 创建逻辑库

用Navicat打开MongoDB连接，然后点击鼠标右键，选择新建数据库。创建的罗辑库名称为his，这个逻辑库是给我们项目使用的。

## 编写持久层代码

我们先创建一个商品快照的实体类，用来封装商品快照的数据。

### 创建Pojo映射类

既然我们要向MongoDB中保存商品快照信息，肯定需要用到Pojo类。在`com.example.his.api.db.pojo` 包中，创建 `GoodsSnapshotEntity.java` 类。这个映射类的结构就是 `goods_snapshot` 集合的结构。

```java
package com.jiang.his.api.db.pojo;

import java.io.Serializable;
import java.math.BigDecimal;
import java.util.List;
import java.util.Map;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import lombok.Data;

@Data
@Document(collection = "goods_snapshot")
public class GoodsSnapshotEntity implements Serializable {
    @Id
    private String _id;

    @Indexed
    private Integer id;

    private String code;

    private String title;

    private String description;

    private List<Map<String, Object>> checkup_1;

    private List<Map<String, Object>> checkup_2;

    private List<Map<String, Object>> checkup_3;

    private List<Map<String, Object>> checkup_4;

    private String image;

    private BigDecimal initialPrice;

    private BigDecimal currentPrice;

    private String type;

    private List<String> tag;

    private String ruleName;

    private String rule;

    private List<Map<String, Object>> checkup;

    @Indexed
    private String md5;
}

```

### 编写Dao方法

因为MongoDB不支持SQL语句，所以MyBatis也没办法用。我们就直接编写DAO方法，通过Java代码来管理MongoDB中的数据。在`com.example.his.api.db.dao` 包中，创建 `GoodSsnapshotDao.java` 类

```java
package com.jiang.his.api.db.dao;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.stereotype.Repository;

import com.jiang.his.api.db.pojo.GoodsSnapshotEntity;

@Repository
public class GoodsSnapshotDao {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    /**
     * 判断MongoDB中是否存在指定md5的商品快照数据
     * @param md5 商品快照的md5值
     * @return 存在返回true，不存在返回false
     */
    public String existsByMd5(String md5) {
        Query query = new Query(Criteria.where("md5").is(md5));
        query.skip(0);
        query.limit(1);
        GoodsSnapshotEntity entity = mongoTemplate.findOne(query, GoodsSnapshotEntity.class);
        return entity != null ? entity.get_id() : null;
    }
    
    /**
     * 插入一条商品快照数据（如果已存在则更新）
     * @param entity 商品快照实体
     * @return 插入或更新后的商品快照实体
     */
    public String insert(GoodsSnapshotEntity entity) {
        String _id = mongoTemplate.save(entity).get_id();
        return _id;
    }
}

```

## 限定特殊客户下单购买体检套餐

某个客户当天有10笔未付款的订单，或者有5笔以上的退款订单，该用户当天就无法下单购买任何体检套餐。

### 编写持久层代码

在 `OrderDao.xml` 文件中，声明SQL语句，查询该用户当天未付款订单数量和退款订单数量。

```xml

<!-- 判断用户当天是否合法 -->
    <select id="checkCustomerLegality" resultType="Boolean">
        SELECT 
            CASE
                WHEN (
                    (SELECT COUNT(*) FROM tb_order WHERE customer_id = #{customerId} AND status &lt; 3 AND create_date = CURDATE()) >= 10
                    OR
                    (SELECT COUNT(*) FROM tb_order WHERE customer_id = #{customerId} AND status = 4 AND refund_date = CURDATE()) >= 5
                ) THEN FALSE
                ELSE TRUE
            END as isLegal
    </select>
```

在 `com.example.his.api.db.dao` 包 `orderDao.java` 接囗中，声明DAO方法

```java
/**
 * 判断用户当天是否合法
 * 规则：某个客户当天有10笔未付款的订单，或者有5笔以上的退款订单，该用户当天就不合法
 * @param customerId 客户ID
 * @return true 表示合法，false 表示不合法
 */
Boolean checkCustomerLegality(Integer customerId);
```

### 关闭超时未支付的订单

```xml
<!-- 关闭超时未支付的订单 -->
    <update id="closeTimeoutOrders">
        UPDATE tb_order
        SET status = 2
        WHERE status = 1
          AND create_date &gt;= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
          <!-- 订单创建时间超过7天，并且创建时间小于当前时间减去30分钟的订单，则关闭，注意时区 -->
          AND create_time &lt; CONVERT_TZ(NOW(), '+00:00', '+08:00') - INTERVAL 30 MINUTE
        ORDER BY create_time
        LIMIT 100
    </update>
```

### 创建微信支付订单

```java
/**
     * 创建微信支付订单
     */
    @Override
    @Transactional
    public Map<String, Object> createWxPayOrder(Integer customerId, Integer goodsId, Integer number) {
        // 检查用户是否合法
        Boolean isLegal = orderDao.checkCustomerLegality(customerId);
        if (isLegal == null || !isLegal) {
            log.warn("用户[{}]创建订单受限：当天未付款订单过多或退款订单过多", customerId);
            return null;
        }
        
        // 1. 查询商品信息
        HashMap<String, Object> goods = goodsDao.selectGoodsWithRule(goodsId);
        if (goods == null) {
            throw new HisException("商品不存在");
        }

        // 2. 计算订单金额
        BigDecimal currentPrice = new BigDecimal(goods.get("currentPrice").toString());
        
        // 应用折扣规则
        BigDecimal finalPrice = currentPrice;
        BigDecimal totalAmount = currentPrice.multiply(new BigDecimal(number));
        
        Object ruleContent = goods.get("rule");
        if (ruleContent != null && StringUtils.hasText(ruleContent.toString())) {
            try {
                // 初始化ExpressRunner
                ExpressRunner runner = new ExpressRunner();
                DefaultContext<String, Object> context = new DefaultContext<>();
                
                // 设置上下文变量
                context.put("number", number);                                // 购买数量
                context.put("price", currentPrice.doubleValue());             // 商品原价
                context.put("totalPrice", totalAmount.doubleValue());         // 总价
                
                // 执行规则表达式计算折扣
                Object result = runner.execute(ruleContent.toString(), context, null, true, false);
                
                if (result != null) {
                    // 转换为BigDecimal，保留2位小数（这是折扣后的总价）
                    double discountedTotalPrice = Double.parseDouble(result.toString());
                    totalAmount = BigDecimal.valueOf(discountedTotalPrice).setScale(2, RoundingMode.HALF_UP);
                    
                    // 计算折扣后的单价
                    finalPrice = totalAmount.divide(new BigDecimal(number), 2, RoundingMode.HALF_UP);
                    
                    log.info("商品[{}]应用折扣规则，原单价{}，折后单价{}，原总价{}，折后总价{}", 
                            goodsId, currentPrice, finalPrice, 
                            currentPrice.multiply(new BigDecimal(number)), totalAmount);
                }
            } catch (Exception e) {
                log.error("计算折扣规则失败：{}，将使用原价", e.getMessage(), e);
                // 规则执行失败，使用原价
                totalAmount = currentPrice.multiply(new BigDecimal(number));
            }
        }
        
        // 3. 生成订单号
        String outTradeNo = generateOrderNo();
        
        // 检查商品快照是否存在，如果不存在则创建
        String snapshotId;
        String md5 = (String) goods.get("md5");
        if (md5 != null && !md5.isEmpty()) {
            // 查询MongoDB中是否已存在此商品快照
            String existingSnapshotId = goodsSnapshotDao.existsByMd5(md5);
            
            if (existingSnapshotId == null) {
                // 如果不存在，则创建新的商品快照
                GoodsSnapshotEntity snapshotEntity = new GoodsSnapshotEntity();
                snapshotEntity.setId(goodsId);
                snapshotEntity.setCode((String) goods.get("code"));
                snapshotEntity.setTitle((String) goods.get("title"));
                snapshotEntity.setDescription((String) goods.get("description"));
                
                // 设置检查项数据，需要先检查是否存在
                if (goods.containsKey("checkup_1")) {
                    snapshotEntity.setCheckup_1((String) goods.get("checkup_1"));
                }
                if (goods.containsKey("checkup_2")) {
                    snapshotEntity.setCheckup_2((String) goods.get("checkup_2"));
                }
                if (goods.containsKey("checkup_3")) {
                    snapshotEntity.setCheckup_3((String) goods.get("checkup_3"));
                }
                if (goods.containsKey("checkup_4")) {
                    snapshotEntity.setCheckup_4((String) goods.get("checkup_4"));
                }
                if (goods.containsKey("checkup")) {
                    snapshotEntity.setCheckup((String) goods.get("checkup_5"));
                }
                
                snapshotEntity.setImage((String) goods.get("image"));
                snapshotEntity.setInitialPrice(new BigDecimal(goods.get("initialPrice").toString()));
                snapshotEntity.setCurrentPrice(currentPrice);
                
                // 设置类型、标签和规则信息
                snapshotEntity.setType((String) goods.get("type"));
                if (goods.containsKey("tag")) {
                    snapshotEntity.setTag((String) goods.get("tag"));
                }
                snapshotEntity.setRuleName((String) goods.get("ruleName"));
                snapshotEntity.setRule((String) goods.get("rule"));
                
                snapshotEntity.setMd5(md5);
                
                // 保存到MongoDB
                snapshotId = goodsSnapshotDao.insert(snapshotEntity);
                log.info("创建商品快照成功，商品ID={}，MD5={}，快照ID={}", goodsId, md5, snapshotId);
            } else {
                // 已存在，直接使用
                snapshotId = existingSnapshotId;
                log.info("使用已存在的商品快照，商品ID={}，MD5={}，快照ID={}", goodsId, md5, snapshotId);
            }
        } else {
            // 没有MD5，使用随机UUID作为快照ID
            snapshotId = UUID.randomUUID().toString();
            log.warn("商品没有MD5值，使用随机UUID作为快照ID：{}", snapshotId);
        }
        
        // 4. 创建本地订单记录
        OrderEntity order = new OrderEntity();
        order.setCustomerId(customerId);
        order.setGoodsId(goodsId);
        order.setSnapshotId(snapshotId); // 使用获取到的快照ID
        order.setGoodsTitle((String) goods.get("title"));
        order.setGoodsPrice(finalPrice);  // 使用折扣后价格
        order.setNumber(number);
        order.setAmount(totalAmount);     // 折扣后总金额
        order.setGoodsImage((String) goods.get("image"));
        order.setGoodsDescription((String) goods.get("description"));
        order.setOutTradeNo(outTradeNo);
        order.setStatus(1); // 1-待支付
        
        // 设置创建时间
        LocalDateTime now = LocalDateTime.now();
        order.setCreateDate(now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd")));
        order.setCreateTime(now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        
        // 保存订单
        orderDao.insert(order);
        
        // 记录支付订单创建日志
        Map<String, Object> logInfo = new HashMap<>();
        logInfo.put("customerId", customerId);
        logInfo.put("goodsId", goodsId);
        logInfo.put("goodsTitle", goods.get("title"));
        logInfo.put("number", number);
        logInfo.put("amount", totalAmount.toString());
        
        orderLogger.logOperation("CREATE_ORDER", outTradeNo, "wechat", logInfo);
        
        // 5. 调用PaymentService创建微信支付订单
        try {
            Map<String, Object> payResult = paymentService.unifiedOrder(order, now);
            QrConfig qrConfig = new QrConfig();
            qrConfig.setWidth(230);
            qrConfig.setHeight(230);
            qrConfig.setMargin(2);
            String codeUrl = (String) payResult.get("codeUrl");
            String qrCodeBase64 = QrCodeUtil.generateAsBase64(codeUrl, qrConfig, "jpg");
            payResult.put(codeUrl, qrCodeBase64);
            return payResult;
        } catch (PayException e) {
            // 支付订单创建失败，更新本地订单状态
            orderDao.updateOrderStatus(outTradeNo, 0, null); // 0-订单创建失败
            log.error("支付创建失败，订单已标记为失败状态，订单号：{}", outTradeNo);
            
            // 重新抛出异常
            throw e;
        }
    }

```

调用PaymentService创建微信支付订单

```java
 /**
     * 创建微信支付订单
     */
    @Override
    public Map<String, Object> unifiedOrder(OrderEntity order, LocalDateTime now) {
        try {
            // 创建支付参数
            // 创建金额对象
            Amount amount = new Amount();
            // FIXME: 上线时需要改为真实的金额
            // amount.setTotal(order.getAmount().multiply(new BigDecimal(100)).intValue()); // 微信支付金额单位是分
            amount.setTotal(1);
            
            // 创建场景信息
            SceneInfo sceneInfo = new SceneInfo();
            sceneInfo.setPayerClientIp("127.0.0.1"); // 用户终端IP

            // 创建支付参数对象
            PayParams payParams = new PayParams();
            payParams.setDescription(order.getGoodsTitle());
            payParams.setOutTradeNo(order.getOutTradeNo());
            payParams.setAmount(amount);
            payParams.setNotifyUrl(payNotifyUrl);
            
            // 设置订单失效时间，默认为20分钟后
            LocalDateTime expireTime = now.plusMinutes(20);
            String timeExpire = expireTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss'Z'"));
            payParams.setTimeExpire(OffsetDateTime.parse(timeExpire));
            
            // 设置场景信息
            payParams.setSceneInfo(sceneInfo);
            
            // 调用微信支付接口
            WechatResponseEntity<ObjectNode> payResult = wechatApiProvider.directPayApi("his-vue").nativePay(payParams);
            
            // 获取支付二维码URL
            String codeUrl = payResult.getBody().get("code_url").asText();
            
            // 将支付二维码URL缓存到Redis中，有效期与订单失效时间一致（20分钟）
            cachePayCodeUrl(order.getOutTradeNo(), codeUrl, 20);
            
            // 构建返回结果
            Map<String, Object> result = new HashMap<>();
            result.put("orderId", order.getId());
            result.put("outTradeNo", order.getOutTradeNo());
            result.put("amount", order.getAmount());
            result.put("goodsTitle", order.getGoodsTitle());
            result.put("codeUrl", codeUrl); // 微信支付二维码链接
            result.put("timeExpire", expireTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))); // 失效时间
            
            return result;
        } catch (Exception e) {
            // 记录支付失败日志
            log.error("支付订单创建失败 - 订单号: {}, 渠道: {}, 金额: {}, 原因: {}", 
                order.getOutTradeNo(), "wechat", order.getAmount().toString(), e.getMessage());
            
            // 抛出支付异常，由调用者处理订单状态更新
            PayException payException = new PayException("PAY_CREATE_FAIL", "创建支付订单失败: " + e.getMessage(), 
                order.getOutTradeNo(), "wechat", e);
            log.error("支付异常 - 错误码: {}, 订单号: {}, 渠道: {}, 异常信息: {}", 
                payException.getErrorCode(), 
                payException.getOutTradeNo(), 
                payException.getChannel(), 
                payException.getMessage(), 
                payException);
            throw payException;
        }
    }
```
