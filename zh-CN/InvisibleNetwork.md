# Invisible Network for DIMP

[![license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/moky/DIMP/blob/master/LICENSE)
[![Version](https://img.shields.io/badge/alpha-0.1.0-red.svg)](https://github.com/moky/DIMP/wiki)

## 无痕通讯时序图

DIMP 无痕通讯过程分为4个阶段：

1. 预设阶段：包括创建用户ID、创建隐身衣ID，以及在 MTA 之间建立恒流管道
2. 发送阶段：发送方创建消息，并以隐身衣包装
3. 传输阶段：消息从 sender 客户端经 MTA 转发至 receiver 客户端
4. 接收阶段：拆除隐身衣，还原消息

```
    User 1       Client 1      MTA 1        MTA 2       Client 2       User 2
  =========    ===========    =======      =======    ===========    =========
      |             |            |            |            |             |
      |----\        |            |            |            |             |----\
      |<---/        |            |<---------->|            |             |<---/
      | S0.1 Create user         | S0.2 Build pipelines    |             | S0.1'
      |             |----\       |            |            |----\        |
      |             |<---/       |            |            |<---/        |
      |             | S0.3 New cloak          |            | S0.3'       |
      |             |----------->|            |<-----------|             |
      |             | S0.4       |            | S0.4' Register cloak for user
      |             |            |            |            |             |
      |----\        |            |            |            |             |
      |<---/        |            |            |            |             |
      | S1 Create a message      |            |            |             |
      |------------>|            |            |            |             |
      | S2 Wrap the message with cloak-1 by client         |             |
      |             |            |            |            |             |
      |             |----------->|            |            |             |
      |             | S3.1 Send the wrapped message to MTA |             |
      |             |            |----------->|            |             |
      |             |            | S3.2 Unwrap and relay the message     |
      |             |            |      through the pipelines            |
      |             |            |            |----------->|             |
      |             |            |            | S3.3 Wrap with cloak-2 and
      |             |            |            |      relay to receiver's client
      |             |            |            |            |             |
      |             |            |            |            |------------>|
      |             |            |            |            | S4 Unwrap message
      |             |            |            |            |             |
```

预设（初始化）阶段说明：

S0.1: 开始使用 DIM 之前，必须首先在客户端创建一个**账号**，包括 User ID 以及与之对应的公钥、私钥等；

S0.2: 网络服务提供商需在 MTA 之间提前建立**恒流管道**，管道流量根据实际所需带宽设置；

S0.3: 客户端运行时，应自动生成一个（临时的）新的**隐身衣**，包括 Cloak ID 以及与之对应的公钥、私钥；

S0.4: 发送方和接收方将各自的新隐身衣信息发送给 MTA 以便注册登记。

## 名词解析

### Invisible Cloak 隐身衣
由客户端生成的临时账号（每次开始通讯前自动生成/定时自动更换），用于隐藏客户端到 MTA 之间的通讯，防御来自本地网络的**身份嗅探**攻击。用法如下：

1. 当发送方客户端需要发送消息时，先正常加密原始的**明文信息**得到加密消息包 msg，然后用自己的隐身衣 cloak-1 进行打包之后得到 msg-1，再将 msg-1 发送给附近的 MTA；
2. 当 MTA 收到一个客户端发来的消息包 msg-1 时，先解开隐身衣得到真实的加密消息包 msg；
3. 当 MTA 与接收方客户端成功连接时，先用接收方的隐身衣 cloak-2 进行打包之后得到 msg-2，再将 msg-2 发送给接收方客户端；
4. 当接收方客户端收到 MTA 转交的消息包 msg-2 时，先解开隐身衣得到真实的加密消息包 msg，再用自己的私钥解密 msg 得到原始的**明文信息**。

通过对隐身衣的进一步利用，还可以实现在 MTA 之间实现隐藏**收发双方身份**和**传输路径**的需求，具体实现方法较为复杂，此处暂且略去。

### Pacific Pipeline 恒流管道
简称 PPL，由网络服务提供商（SP）根据实际需要提前设定，用于消除 MTA 之间的网络流量波动，防御来自公网的**流量特征分析**攻击。

1. 当网络中没有有效负载信息（payload）时，MTA 之间通过生成**随机噪音信息**来维持一个恒定的流量带宽 W；
2. 当网络中存在一定量的 payload 时，假设其所需流量带宽为 W1，则 MTA 生成所需带宽为 W2 = W - W1 的随机噪音信息 noise，再将 payload 和 noise 混杂在一起发送；
3. 当某一时刻 payload 所需带宽 W1 大于 PPL 的预设带宽 W 时，超出部分的消息会被延迟发送（削峰）；
4. 当 W1 > W 的情况频繁发生时，SP 需要考虑增加 W 的值以避免网络拥堵。


Copyright &copy;2020 Albert Moky
