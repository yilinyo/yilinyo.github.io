---
title: go项目中的DDD四层架构
tag: go项目
date: 2026-05-12 00:00:00
---
![本文蓝图](https://files.seeusercontent.com/2026/05/11/yZ1e/_cgi-bin_mmwebwx-bin_webwxgetmsg.jpg)
### DDD GO项目结构
```
internal/
├── order/
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── interfaces/
│
├── payment/
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── interfaces/
│
└── inventory/
    ├── domain/
    ├── application/
    ├── infrastructure/
    └── interfaces/
```

按领域优先划分，再在每个领域内部做分层
```
internal/
├── domain/
│   ├── order/
│   ├── payment/
│   └── inventory/
│
├── application/
│   ├── order/
│   ├── payment/
│   └── inventory/
│
├── infrastructure/
│   ├── order/
│   ├── payment/
│   └── inventory/
│
└── interfaces/
    ├── order/
    ├── payment/
    └── inventory/
```
按技术层优先划分，再在每层里面放不同业务
小型项目 可以先按技术层划分，大型项目建议按上面的更清晰
![DDD项目结构](https://files.seeusercontent.com/2026/05/11/Gi8l/ChatGPT-Image-2026511-23_54_58-2.png)

### 各层概念和职责
interface 里一般有路由 还有handler，handler http请求数据，调用，返回。主要教研请求格式，比如json数据是否合法，字段类型是否一致。然后转换请求对象到dto

application 层组织编排业务调用，不会关心业务的边界，他只负责编排

domain 领域层（DDD架构核心）放业务模型 **业务模型和业务规则**以及repo接口，主要校验业务数据是否满足业务规则，保证领域对象不进入非法状态。**Repository 接口放在 domain，是为了表达“领域需要什么能力”**

infrastructure 负责实现有domain 领域层定义的repo接口实现依赖反转
![四层模型职责与依赖关系](https://files.seeusercontent.com/2026/05/11/Oau8/ChatGPT-Image-2026511-23_52_29-2.png)

### 各层的依赖关系
interface 依赖 application
application 依赖 domain
infrastructure 依赖 domain
domain 不依赖 infrastructure 

### 充血概念
充血模型指领域对象不仅保存实体结构还保存的业务行为和业务规则。
这样就不会全部把一些规则放到application 上去了
Go 充血Order模型例子
```go
type OrderStatus string

const (
	OrderStatusPending  OrderStatus = "PENDING"
	OrderStatusPaid     OrderStatus = "PAID"
	OrderStatusShipped  OrderStatus = "SHIPPED"
	OrderStatusCanceled OrderStatus = "CANCELED"
)

type Order struct {
	ID     int64
	UserID int64
	Status OrderStatus
	Amount int64
}

func (o *Order) Cancel() error {
	switch o.Status {
	case OrderStatusPending:
		o.Status = OrderStatusCanceled
		return nil

	case OrderStatusPaid:
		return errors.New("已支付订单不能直接取消，需要走退款流程")

	case OrderStatusShipped:
		return errors.New("已发货订单不能取消")

	case OrderStatusCanceled:
		return errors.New("订单已经取消，不能重复取消")

	default:
		return errors.New("未知订单状态")
	}
}
```
![充血模型](https://files.seeusercontent.com/2026/05/11/1Abi/ChatGPT-Image-2026511-23_54_54-2.png)


### 领域拆分


核心目标是把复杂业务系统拆成多个边界清晰、职责明确、语言一致的业务模块。

最重要的概念是：

> **按业务语义划分，而不是按技术层划分。**

也就是说，不是简单分成 Controller、Service、DAO，而是按照业务能力划分，例如：用户、订单、支付、库存、物流、营销等。

不要按照表拆分，太细了**一个领域通常会包含多张表**，因为领域不是按表划分，而是按一组完整的业务规则划分。

例如电商里的 **订单领域**，可能包含这些表：
```
orders                 订单主表order_items            订单明细表order_status_logs      订单状态变更日志order_price_details    订单价格明细order_cancel_records   订单取消记录order_invoice_info     订单发票信息
```
这些表虽然是多张表，但它们共同服务于一个核心业务：
管理订单生命周期。
所以它们可以都属于 **订单领域**。

### 与MVC的对比分析
 |对比点|MVC 三层架构|DDD 四层架构|
|---|---|---|
|划分方式|按技术职责划分|按业务职责 + 技术职责划分|
|核心层|Service 层|Domain 层|
|业务规则位置|通常在 Service|Domain 层|
|Service 职责|业务逻辑 + 流程编排 + 数据处理|Application 只负责编排|
|模型特点|常用数据库 Entity / POJO|聚合、实体、值对象|
|数据访问|Service 调 DAO|Domain 定义仓储接口，Infrastructure 实现|
|适用场景|CRUD、简单系统|复杂业务、规则多、长期演进系统|
|优点|简单直接|业务表达清晰，复杂度可控|
|缺点|Service 容易膨胀|设计成本更高|

![DDD VS MVC](https://files.seeusercontent.com/2026/05/11/uY8y/ChatGPT-Image-2026511-23_52_29-4.png)

