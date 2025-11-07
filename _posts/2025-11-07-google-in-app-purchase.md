---
layout: post
title: Google Play 应用内购最佳实践
tags: 应用开发
categories: Android
date: 2025-11-07
---

在出海应用开发过程中，通常会接入 Google Play支付，而常见的三个覆盖场景：即新购、恢复和异常处理，是构建一个稳健支付系统所必须的。下面我将基于Google Play Billing Library，对这三个场景的实现细节、关键API和最佳实践进行详细拆解。

## 场景一：用户新购买流程 (New Purchase Flow)
这是最基础的“快乐路径”，流程的每一步都至关重要。

### 详细步骤：

#### 初始化BillingClient

1. 在onCreate中创建并连接BillingClient。这是所有操作的起点。

#### 查询商品信息 (ProductDetails)

2. 在展示购买按钮前，使用 billingClient.queryProductDetailsAsync() 从Google Play获取商品信息（价格、标题、描述等）。不要在客户端硬编码价格。

#### 发起购买 (Launch Billing Flow)

1. 用户点击购买按钮后，构建BillingFlowParams，并调用 billingClient.launchBillingFlow()。
2. 最佳实践：在BillingFlowParams中，使用 setObfuscatedAccountId() 传递你的用户系统中的唯一ID。这有助于Google检测欺诈行为，并将购买与你的用户关联起来。

#### 处理购买结果 (onPurchasesUpdated)

1. 这是PurchasesUpdatedListener中的核心回调方法。
2. 当responseCode == BillingResponseCode.OK 且 purchases 列表不为空时，表示用户已成功支付（或进入PENDING状态）。
3. 关键操作：从Purchase对象中获取purchaseToken(originalJson和signature等信息)。立即将此其发送到你的服务端进行验证。
4. 注意：此时不要在客户端直接发放权益，因为客户端可能被篡改。

#### 服务端验单入账 (Server-Side Verification)

1. 你的服务端收到purchaseToken后，必须使用 Google Play Developer API 进行验证。
* 对于订阅，调用 purchases.subscriptions.get。
* 对于一次性商品，调用 purchases.products.get。
2. 验证关键字段：
* purchaseState: 必须为 0 (Purchased)。如果是 1 (Pending)，则不应发货，等待后续状态更新。
* acknowledgementState: 确认交易是否已被确认。
3. 入账逻辑：
* 验证通过后，将订单信息（orderId, purchaseToken, 用户ID等）存入你的数据库。
* 必须做好幂等性处理：使用orderId或purchaseToken作为唯一键，防止同一笔交易被重复入账。
* 在你的系统中为用户发放相应的权益（如VIP、金币等）。
* 向客户端返回一个成功或失败的响应。

#### 完成交易 (Acknowledge/Consume)

1. 客户端在收到你的服务端的成功响应后，必须完成Google Play的交易。
2. 对于订阅或非消耗型商品（如去广告），调用 billingClient.acknowledgePurchase()。
3. 对于消耗型商品（如金币、宝石），调用 billingClient.consumeAsync()。
4. 为什么必须做？ 如果一个PURCHASED状态的交易在3天内没有被acknowledge或consume，Google Play会自动退款，并可能认为你的应用存在问题。

## 场景二：恢复购买 (Restore Purchases)
这个场景主要针对已订阅用户，确保他们在更换设备或重装应用后，权益能自动恢复。

### 详细步骤：

#### 触发时机

1. 自动恢复：在应用启动时（如Activity的onResume）调用。这是最佳实践，用户无需任何操作。
2. 手动恢复：提供一个“恢复购买”按钮，用户点击时调用。

#### 查询历史交易 (Query Purchases)

1. 调用 billingClient.queryPurchasesAsync()。这个API会返回当前Google账户在该应用下所有有效的购买记录（即，有效的订阅和未消耗的一次性商品）。
3. 注意：它不会返回已过期、已取消或已消耗的交易。


#### 处理查询结果

1. 遍历返回的Purchase列表。
2. 对于列表中的每一个Purchase对象：
* 执行与“新购”类似的服务器验证流程：将purchaseToken发送到你的服务器。
* 服务器逻辑：服务器查询数据库。
如果这笔交易已经存在于你的数据库中，说明是老用户，直接告诉客户端“验证成功”。
如果这笔交易不存在（可能是掉单或新设备首次同步），则执行完整的验单入账流程。
* 客户端逻辑：根据服务器的响应，在UI上为用户解锁相应权益。
* 补确认：在遍历时，检查 purchase.isAcknowledged()。如果返回false，说明上次购买流程中客户端确认步骤失败了。在服务器验证成功后，客户端应立即调用 acknowledgePurchase() 进行“补确认”，防止该笔交易被退款。


## 场景三：处理未完成的交易 (Handling Pending/Unfinished Transactions)
这是保证系统稳健性的关键，国内称之为“补单”逻辑。它处理两种情况：1. 支付后应用崩溃导致未上报；2. 用户使用了需要延迟确认的支付方式（如便利店现金支付）。

### 详细步骤

#### 在应用启动时处理

1. 这个逻辑和“恢复购买”逻辑完全重合，都依赖于在onResume中调用 queryPurchasesAsync()。
2. 当你从 queryPurchasesAsync() 的结果中找到一个purchaseState为 PURCHASED 但 isAcknowledged() 为 false 的交易时，就意味着这是一笔“未完成”的交易。
3. 处理流程：严格按照 场景二（恢复购买） 的第3步进行处理：发到服务端验证 -> 服务端入账 -> 客户端确认(Acknowledge)。

#### 处理待处理交易 (Pending Purchases)

1. 当用户选择某些支付方式时，onPurchasesUpdated回调会返回一个purchaseState为 PENDING 的Purchase对象。
2. 客户端操作：
* 不要发货！不要授予用户任何权益。
* 可以向用户显示一个提示，例如：“您的订单正在处理中，支付成功后将自动到账。”
* 必须确认这笔“待处理”交易：对于PENDING状态的交易，同样需要调用 acknowledgePurchase()。Google官方文档指出，这也需要确认，否则可能被取消。
3. 后续状态变化：
* 当用户完成支付后，onPurchasesUpdated会再次被调用，此时同一个purchaseToken对应的Purchase对象，其purchaseState会变为 PURCHASED。
* 这时，你就按照**场景一（新购）**的流程处理这笔交易：发到服务端验单 -> 入账 -> 发放权益。
4. 服务端最佳实践：使用 Real-Time Developer Notifications (RTDN)。当一笔交易从PENDING变为PURCHASED时，Google会主动向你的服务器发送一个通知。这比依赖客户端上报更及时、更可靠。
