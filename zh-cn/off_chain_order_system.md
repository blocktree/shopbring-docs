# 链下订单系统设计

## 概述

用户在平台不断发生代购交易，系统会记录相当庞大的订单数据。如果把订单数据存储在链上，将会产生非常大的存储成本。因此我们会开发一个链下的订单管理系统，用于存储详细的购物订单信息，发货信息，发票信息，退货信息，这些信息会进行序列化并哈希后才存储到链上。该系统是部署在我们管理的中心化服务器，会采用像polkassembly.io那样可以使用Polkadot钱包签名授权登录，普通账户只能访问自己的订单数据。验票人可以访问待验证的商品发票数据和其关联的商品明细。

## 业务流程

### Polkadot钱包授权登录

## 数据模型

### OrderDetail `订单明细`

| Parameter          | Type        | Description    |
|--------------------|-------------|----------------|
| id                 | uint64      | 订单ID         |
| consumer           | string      | 消费者账户地址 |
| shopping_agent     | string      | 代购者账户地址 |
| payment_amount     | big.Int     | 支付金额       |
| tip                | big.Int     | 小费           |
| currency           | uint32      | 支付币种       |
| status             | uint32      | 订单状态       |
| create_time        | uint64      | 提交时间       |
| required_deposit   | big.Int     | 保证金要求     |
| required_credit    | uint64      | 信用值要求     |
| logistics_company  | string      | 物流公司       |
| shipping_num       | string      | 发货运单号     |
| receiver           | string      | 收货人         |
| receiver_phone     | string      | 收货人电话     |
| shipping_address   | string      | 收货地址       |
| is_return          | Bool        | 是否有申请退货 |
| version            | uint32      | 交易单版本     |
| commodities        | []Commodity | 订购商品数组   |
| platform_id        | string      | 电商平台ID     |
| platform_order_num | string      | 电商平台订单号 |
| merchant           | string      | 商家账户       |
| note               | string      | 备注           |
| hash               | string      | 订单明细哈希   |
| fare               | u64         | 商家法币运费   |
| total              | u64         | 法币商品合计   |

### CommodityDetail `商品明细`

| Parameter | Type   | Description |
|-----------|--------|-------------|
| id        | uint64 | 商品ID      |
| o_id      | uint64 | 订单ID      |
| hash      | string | 商品哈希    |
| name      | string | 商品名字    |
| url       | string | 商品链接    |
| options   | string | 附加选项    |
| amount    | uint64 | 数量        |
| price     | uint64 | 法币单价    |
| total     | uint64 | 法币小计    |
| note      | string | 备注        |

### ReturnOrderDetail `退货订单明细`

| Parameter         | Type         | Description              |
|-------------------|--------------|--------------------------|
| id                | uint64       | 退货订单ID               |
| c_id              | uint64       | 商品ID                   |
| o_id              | uint64       | 订单ID                   |
| return_amount     | Balance      | 退货金额                 |
| status            | ReturnStatus | 退货状态                 |
| create_time       | Timestamp    | 提交时间                 |
| logistics_company | string       | 物流公司                 |
| shipping_num      | string       | 发货运单号               |
| receiver          | string       | 收货人                   |
| receiver_phone    | string       | 收货人电话               |
| shipping_address  | string       | 收货地址                 |
| amount            | u64          | 退货数量                 |
| price             | u64          | 商家法币单价             |
| total             | u64          | 商家法币小计             |
| note              | string       | 备注                     |
| return_type       | string       | 类型：0：退货退款，1：只退款 |
