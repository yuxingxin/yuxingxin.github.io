---
layout: post
title:  iOS 应用内购用户退款Apple处理流程
tags: 应用开发
categories: iOS
date: 2026-06-25
---
苹果处理 iOS App / IAP 用户退款，大致可以按这个流程理解：

### 1. 用户向 Apple 发起退款申请

用户通常通过 Apple 的问题报告入口申请退款，选择退款原因并提交。退款审批主体是 Apple，不是开发者。常见原因包括误购、未授权消费、订阅扣费争议、商品/服务不满意、账号被盗用等。

### 2. Apple 判断是否需要开发者提供消耗信息

如果退款涉及 **消耗型内购** 或 **自动续期订阅**，并且 Apple 认为需要了解 App 侧交付/消耗情况，可能会向开发者配置的 App Store Server Notifications V2 URL 发送：

```
CONSUMPTION_REQUEST
```

它表示：用户已发起退款申请，Apple 请求开发者提供 consumption data。

注意：这不是每个退款申请都会发，也不代表退款已成功。

### 3. 开发者上报消费/交付情况

收到  `CONSUMPTION_REQUEST`  后，服务端应调用  `Send Consumption Information` ，向 Apple 上报，例如：

- 是否已成功交付商品或服务

- 用户消费了多少比例

- 是否提供过免费试用、样例内容或功能说明

- 开发者是否倾向同意退款

- 用户是否同意提供消费数据

 `CONSUMPTION_REQUEST`  里可能包含  `consumptionRequestReason` ，这是用户申请退款时选择的原因分类，例如误购、交付问题、不满意、法律原因、其他等。它不是用户填写的完整说明。

### 4. Apple 做退款裁决

Apple 会结合用户申请原因、交易类型、购买时间、账号/支付风险、开发者上报的消费信息、地区政策等因素判断是否批准退款。

Apple 没有公开一张“退款原因 -> 是否发送  `CONSUMPTION_REQUEST`  -> 是否退款成功”的固定规则表。所以未成年人未经授权消费、误触购买、账号被盗等场景，不能简单判断为一定发送或一定不发送  `CONSUMPTION_REQUEST` 。

### 5. 退款请求被拒绝时

如果 Apple 拒绝退款，可能发送：

```
REFUND_DECLINED
```

这表示退款申请没有通过，开发者通常不需要撤销权益。

### 6. 退款成功时

如果 Apple 批准并完成退款，会向服务端发送：

```
REFUND
```

这才是正式退款成功信号。服务端应根据交易信息撤销权益，例如会员、金币、虚拟道具、内容访问权限等。

 `REFUND`  通知中的交易 payload 通常关注：

-  `transactionId` 

-  `originalTransactionId` 

-  `productId` 

-  `revocationDate` 

-  `revocationReason` 

### 7. revocationReason的含义

 `revocationReason`  是退款/撤销后的粗粒度原因码，不是用户填写的完整退款说明：

```
0 = 其他原因退款，例如误购
1 = 用户认为 App 或服务存在实际或感知问题
```

无论是  `0`  还是  `1` ，只要收到  `REFUND` ，都说明交易已被退款/撤销，服务端都应撤销对应权益。 `1`  可以额外用于产品质量、交付问题分析； `0`  可以用于误购、账号、安全、客服类分析。

### 8. 退款被反转时

如果之前已经批准的退款后来因争议等原因被反转，Apple 可能发送：

```
REFUND_REVERSED
```

如果服务端之前已经撤销了权益，这时需要恢复对应权益。

### 9.服务端不能只依赖通知

Apple 通知不是业务上绝对可靠的队列。可能存在未配置 URL、服务端不可用、证书/TLS 问题、超时、返回非成功状态、沙盒行为差异等情况。

所以生产设计建议：

```
CONSUMPTION_REQUEST:
  记录退款申请阶段信息，按需上报 consumption data；
  不撤销权益。

REFUND_DECLINED:
  记录退款被拒；
  不撤销权益。

REFUND:
  幂等撤销权益；
  记录 revocationDate / revocationReason。

REFUND_REVERSED:
  幂等恢复权益。

Get Refund History:
  定时或关键场景补偿查询，防止漏通知。
```

### 10. 总结

核心结论： `CONSUMPTION_REQUEST`  **是可选的退款申请阶段信号，** `REFUND`  **才是退款成功后的权益处理信号。服务端应以**  `REFUND  **和退款历史查询作为最终依据。**
