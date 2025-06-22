---
title: QLExpress规则引擎-java学习笔记(16)
date: 2025-05-05 22:23:29
tags:
  - java
---

## 为什么要使用规则引擎?

往简单了说，规则引擎是用来执行条件判断的，符合什么规则就执行什么代码。既然是这样，为什么我们不直接写if-else语句，而要用规则引擎呢?答案很简单:可维护性。如果你在Java代码中写了一堆条件判断，将来判断依据变了，你要重新修改代码编译项目，还要重新发布项目，整个流程太繁琐了。如果采用规则引擎，我们可以把条件判断写成文本保存到配置文件或者数据库，日后维护也更加方便，修改配置文件或者数据记录就可以了，不需要修改程序代码。

<!-- more -->
## QLExpress规则引擎

现在业界有很多规则引擎产品，比如说老牌的Drools、Easy-Rules、RuleBook等等，这里我选择是[阿里开源的QLExpress规则引擎](https://gitee.com/cuibo119/QLExpress)。它轻量级，启动速度快，占用内存小，可以很容易与SpringBoot项目融合。更重要的一点，它的语法与Java非常贴近，可以实现复杂的条件判断和代码语句。像是Drools这种规则引擎，只能做简单的条件判断，不支持复杂的表达式和语法，所以从功能上来说还是QLExpress最佳。

## 整合QLExpress规则引擎

在Java项目中的 `pom.xml` 文件中，添加下面的标签，把规则引擎整合到Java项目

```xml
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
```

## QLExpress规则引擎使用

```java
package com.jiang.his;

import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;

public class TestRule {
    public static void main(String[] args) throws Exception {
        ExpressRunner runner = new ExpressRunner();
        String rule = """
                temp=salary-3500;
                tax=0;
                if(temp<=0){
                    tax=0;
                } else if(temp<=1500){
                    tax=temp*0.03-0;
                } else if(temp<=4500){
                tax=temp*0.10-105;
                } else if(temp<=9000){
                    tax=temp*0.20-555;
                } else if(temp<=35000){
                 tax=temp*0.25-1005;
                }
                else if(temp<=55000){
                    tax=temp*0.30-2755;
                }else if(temp<=80000){
                tax=temp*0.35-5505;
                }
                else{
                tax=temp*0.45-13505;
                }
                return tax;
                """;
        DefaultContext<String, Object> context = new DefaultContext<String, Object>();
        context.put("salary", 50500);
        Object r = runner.execute(rule, context, null, true, false);
        System.out.println(r.toString());
    }
}
```

## 第二件半价怎么计算?

假设某个体检套餐价格是1000元，它促销规则是第二件半价。客户一笔订单中，购买了几件该体检套餐决定了不同的折扣金额。
1.如果只购买一件该体检套餐，则不享受促销优惠，订单金额是1000元。
2.如果购买两件该体检套餐，则第二件享受半价，订单金额是1500元
3.如果购买三件该体检套餐，则最后一件享受半价，订单金额是2500元
4.如果购买四件该体检套餐，则前两件原价，后两件享受半价，订单金额是3000元

## 数据表中的计算规则

我们打开 tb_rule 数据表，里面保存了第二件半价的计算规则。将来我们做到促销规则模块的时候，可以在MIS系统中，添加新的促销规则。现在我们就暂时看一下已有的促销规则。

数据表 rule 字段中的完整内容如下。计算的公式为: 总金额 - 折扣金额 。很多同学看不懂下面表达式中的number/2 代表什么意思。其实很简单，就是计算有几个商品享受折扣。如果是3/2，结果是1;如果是4/2，结果是2;整数相除，结果还是整数，现在能看明白是什么意思了吧。

```java
import java.math.BigDecimal;   

p = new BigDecimal(price);
n = new BigDecimal(number);

result = n.multiply(p).subtract(new BigDecimal(number / 2).multiply(new BigDecimal("0.5")).multiply(p)).toString();
```
