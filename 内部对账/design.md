# 内部对账设计方案

## 需求背景

目前的支付系统内部系统，各模块

## 实现目标

- 可动配置化对账主体SQL（如提现订单与提现流水核对）

- 日终自动触发（加入日终处理作业）

- 支持重新核对
    > 可通过sendmsg + 对账名称 + 指定日期来重新进行核对

- 核对结果与差异存档
    > 创建对账结果表

- 核对结果异常自动触发监控
    > 监控系统维护规则，每日抓取核对结果，通知相关人员

- 以系统时间为鉴，不考虑时间差导致的存疑
    > 为了从简，暂不考因系统时间差导致的差异

## 设计思路

```mermaid

```

### 数据匹配规则情况

1. 两边完全对等 (提现订单与通道流水)
2. 左边包含右边
3. 右边包含左边（卡包消费订单与卡明细）
4. 联合多表集合（正交易与冲正的情况）

## 系统设计

### 数据结构

#### 对账规则配置表

### 模块间表

#### 远程消费模块 [RPM]

| Schema | Table        | Module | Desc           |
| :----: | ------------ | ------ | -------        |
| ipay   | T_RPM_ORDR   | RPM    | 远程消费订单表 |
| ipay   | T_RPM_RTUL   | RPM    | 远程消费退款表 |


#### 充值模块

| Schema | Table          | Module | Desc                       |
| :----: | ------------   | ------ | -------                    |
| ipay   | T_PPD_ORDR     | PPD    | 充值订单表                 |
| ipay   | T_PPD_JNQPSPDB | PPD    | 通道流水表（浦发快捷支付） |
| ipay   | T_PPD_JNQPUPOP | PPD    | 通道流水表（银联快捷支付） |
| ipay   | T_PPD_JNUPOP   | PPD    | 通道流水表（银联B2C支付）  |
| ipay   | T_PPD_RFD      | PPD    | 充值退款订单表             |
| ipay   | T_PPD_JFQPUPOP | PPD    | 通道流水表（银联快捷退款） |
| ipay   | T_PPD_JFQPSPDB | PPD    | 通道流水表（浦发快捷退款） |
| ipay   | T_PPD_JFUPOP   | PPD    | 通道流水表（银联B2C退款）  |

#### 提现模块 [PWM]

| Schema | Table        | Module | Desc                         |
| :----: | ------------ | ------ | -------                      |
| ipay   | T_WDC_ORDR   | PWM    | 提现订单表                   |
| ipay   | T_WDC_JNCORG | PWM    | 提现通道流水表（含所有通道） |

#### OK卡模块 [CRD]

| Schema | Table          | Module | Desc           |
| :----: | ------------   | ------ | -------        |
| icon   | T_CRD_ORD      | CRD    | OK卡支付订单表 |
| icon   | T_CRD_ORD_DTL  | CRD    | OK卡支付明细表 |
| icon   | T_CRD_REFD_DTL | CRD    | OK卡退款明细表 |


#### 生活缴费模块 [PES]

| Schema | Table        | Module | Desc                             |
| :----: | ------------ | ------ | -------                          |
| icon   | T_PES_ORDR   | PES    | 生活缴费订单表                   |
| icon   | T_PES_JRN    | PES    | 生活缴费流水表（含光大与全渠道） |


#### 账务会计 [ACM & ACT]

| Schema | Table        | Module | Desc    |
| :----: | ------------ | ------ | ------- |
| ipay   |              |        |         |


### 3 模块间关系


