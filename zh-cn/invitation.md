# 邀请人模块设计

## 概述

Shopbring网络刚上线时，普通用户在平台都没有`SBG`（原生代币），无法进行链上发起交易。为了解决此问题，我们设计了邀请人机制。
普通用户持有原生币都可以注册成为邀请人，给受邀人赠予小量`SBG`，而当受邀人完成代购交易得到信用分增值，他的上级邀请人也能得到小量信用分增值。

## 业务流程

### 角色

- **邀请人**。根据注册表单，需要冻结一定`SBG`成为邀请人，并搭建一个链下的邀请人服务，让受邀人访问服务获得邀请授权。
- **受邀人**。获得邀请人的链下邀请授权后，可在链上发起接受邀请获得邀请人发放的赠金。

1. 邀请人离线下随机生成一个ed25519密钥对，我们定义私钥为$p_i$，公钥为$PK_i$，调用`registerInviter`方法注册成为邀请人，参数有：发送者地址，$PK_i$，单笔赠金`SBG`，最大可邀请人数，邀请期限（可选几个时期）。$系统冻结邀请人_{SBG} = 单笔赠金_{SBG} * 最大可邀请人数$。
1. 邀请人可搭建自己的服务器，可给满足特定条件的用户才授权邀请签名。获得授权的用户把他的账户公钥$PK_a$作为签名参数，签名算法为$Sign_{ed25519}(Blake2b(PK_a), p_i)$，得到受邀签名$SIG_i$。
1. 受邀人调用acceptInvitation方法接受邀请，参数有：发送者公钥，$SIG_i$，邀请人账户地址。
1. 系统查找邀请人是否存在，使用期限是否到期，并$Verify_{ed25519}(PK_i, Blake2b(PK_a), SIG_i)$。验证通过后，单笔赠金数`SBG`转账给受邀人，这笔交易单的手续费也由邀请人支出。
1. 受邀人每完成一笔代购交易，自己的信用分会增加，同时他的上级邀请人的信用分也会小量增加。
1. 邀请人可以调用`endInvitationPeriod`提前结束邀请期，来回收冻结中的赠金。

## 数据模型

### 常量

#### MinUnitBonus `最低单位赠金数`

单位赠金数可由邀请人自由设置，但不能低于系统要求的最低值。

### 成员变量

#### TotalInviters `全部邀请人`

当前全网络注册的邀请人

#### TotalDistributedBonus `已发放的赠金`

全网络累积邀请人发放的赠金总数

### 存储表

#### InviterRegistration `邀请人登记表`

邀请人绑定其邀请人登记信息。

```rust

InviterRegistration map hasher(twox_64_concat) T::AccountId => InvitationInfo<T::AccountId, T::Balance>;

```

#### InviterRelationship `邀请人关系表`

被邀请绑定其上级邀请人的索引表

```rust

InviterRelationship map hasher(twox_64_concat) T::AccountId => T::AccountId;

```

### 枚举

无定义

### 结构体

### InvitationInfo `邀请人信息`

存储已登记的邀请人信息

| Parameter    | Type    | Description      |
|--------------|---------|------------------|
| invitationPK | byte32  | 邀请函校验公钥。  |
| unitBonus    | Balance | 单笔赠金数`BPG`。 |
| maxInvitees  | u64     | 最大邀请人数。    |
| frozenAmount | Balance | 冻结的金额。      |
| numOfInvited | u32     | 已邀请人数。      |

## 公共方法

### registerInviter `注册邀请人`

#### 函数定义

首先邀请人离线生成ed25519密钥对，把其中的公钥作为`invitationPK`参数提交。

```javascript

/// registerInviter
/// - inviter address 邀请人账户地址
/// - invitationPK byte32 邀请函校验公钥
/// - unitBonus Balance 单笔赠金数
/// - maxInvitees u64 最大邀请人数
registerInviter(inviter, invitationPK, unitBonus, maxInvitees);

```

#### Event

```javascript

/// registerInviter
/// - inviter address 邀请人账户地址
/// - invitationPK byte32 邀请函校验公钥
/// - unitBonus Balance 单笔赠金数
/// - maxInvitees u64 最大邀请人数
/// - frozenAmount Balance 冻结的金额
RegisterInviter(inviter, invitationPK, unitBonus, maxInvitees, frozenAmount)

```

### acceptInvitation `接受邀请`

`invitationSIG`需要邀请人的邀请函私钥签名受邀人账户公钥Blake2b获得。
`acceptInvitation`会查询`inviter`的邀请人信息是否存在，最后验证`invitationSIG`是否正确。

#### 函数定义

```javascript

/// acceptInvitation
/// - invitee address 受邀人账户地址
/// - invitationSIG byte64 邀请函签名
/// - inviter address 邀请人账户地址
acceptInvitation(invitee, invitationSIG, inviter);

```

#### Event

```javascript

/// AcceptInvitation
/// - invitee address 受邀人账户地址
/// - invitationSIG byte64 邀请函签名
/// - inviter address 邀请人账户地址
AcceptInvitation(invitee, invitationSIG, inviter);

```

### endInvitationPeriod `结束邀请期`

邀请人结束邀请期，可以回收未使用的赠金。

#### 函数定义

```javascript

/// endInvitationPeriod
/// - inviter address 邀请人账户地址
endInvitationPeriod(inviter);

```

#### Event

邀请人提前结束邀请期将记录该事件，最后一个受邀人接受邀请时也会纪录该事件。纪录事件后，系统将删除邀请人信息。

```javascript

/// EndInvitationPeriod
/// - inviter address 邀请人账户地址
/// - reclaimedBonus Balance 已回收的赠金
/// - numOfInvited Balance 已邀请人数
/// - end BlockNumber 准确的结束时间
EndInvitationPeriod(inviter, reclaimedBonus, numOfInvited, end);

```

## 错误码

- `InsufficientBalance`。账户余额不足。
- `InvitationInfoIsExisted`。邀请人信息已存在，不可重复提交。
- `InvitationInfoIsNotExisted`。邀请人信息不存在。
