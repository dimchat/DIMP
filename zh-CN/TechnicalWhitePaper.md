# DIMP（去中心化即时通讯协议）技术白皮书 (V0.2)

草案：**2018年11月11日** (@moky)

[![license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/moky/DIMP/blob/master/LICENSE)
[![Version](https://img.shields.io/badge/alpha-0.2.0-red.svg)](https://github.com/moky/DIMP/wiki)

**摘要:** 本文档引入一种新的为分布式即时通讯而设计的协议，以及一个为开发分布式即时通讯应用而设计的技术架构。该软件提供了一个账号系统（用户身份认证）以及在各账号之间通过端对端加密实现的安全通讯服务。

Copyright &copy; 2018 Albert Moky

- [背景](#background)
- [去中心化 IM 工具的需求](#requirements)
    - [去中心化的用户身份认证技术](#identity)
    - [基于端对端加密的安全通讯技术](#end-to-end)
    - [高并发（支持亿级用户）](#concurrence)
    - [低延时](#low-latency)
    - [免费使用](#free)
    - [可定制、可升级](#scalable)
- [共识](#consensus)
    - [身份确认](#identification)
    - [关系维护](#relationship)
    - [消息加密与校验](#encrypt-and-verify)
- [ID（账号）](#account)
    - [Name（账号名）](#name)
    - [Address（账号地址）](#address)
- [Group（群组）](#group)
    - [Polylogue - 多人会话（临时讨论组）](#polylogue)
    - [Chatroom - 聊天室（固定群/大规模群聊天）](#chatroom)
- [消息](#message)
    - [消息头（Envelope）](#envelope)
    - [消息内容（Content）](#content)
    - [消息包数据结构](#message-structure)
        - [原始消息包（Instant Message）](#instant-message)
        - [加密消息包（Secure Message）](#secure-message)
        - [网络传输包（Reliable Message）](#reliable-message)
- [密码](#cryptography)
    - [身份密钥 (Identity Key)](#identity-key)
    - [通讯密钥 (Communication Key)](#communication-key)
        - [对称密钥](#symmetric-key)
        - [非对称密钥](#asymmetric-key)
- [消息处理流程](#message-process)
    - [二人私聊](#personal-message)
    - [多人群聊](#group-message)
- [扩展协议](#extensions)
    - [握手协议（身份验证）](#handshake)
    - [广播消息通讯协议](#broadcast)
    - [用户资料广播协议 (Visa Protocol)](#visa)
    - [登录点信息广播协议](#login)
    - [消息包确认签收协议](#receipt)
    - [好友列表](#contacts)
    - [屏蔽列表](#blocked)
    - [蚂蚁搬运工](#ant-porter)
- [结论](#conclusion)

## 0. <span id="background">背景<span>

即时通讯（IM）是目前 Internet 上最为流行的通讯方式，它允许网络上的两个人或多人使用网络实时的传递文字消息、文件、语音与视频交流。
从用户使用场景角度，目前市场上的IM工具可划分为**企业级IM工具**（如钉钉、企业微信、RTX 等）和**个人IM工具**（如 Facebook、Whatsapp、WeChat、QQ、Telegram 等）两大类。

而当前所有最流行的 IM 工具其平台架构设计都是中心化的，每个工具背后都是一套自有的账号认证系统以及消息转发系统。这将造成至少以下3个难以解决或避免的关键性问题：

1. 每个用户的账号密码以及关系链等重要资料都被绑定在特定的平台上，**用户无法自由选择**，出于商业目的其他平台也难以共享；
2. 每个平台需要中心化的数据库来存储用户账号密码等资料，这些中心点很容易成为**黑客攻击**的目标，如果被突破，将会引发大面积的社会性事件；
3. 用户**完全依赖**于其所在的 IM 平台处理能力，一旦出现中心故障将可能导致无法登录与通讯。

问题1的主要症结在于这种中心化机制**不利于持续创新**，容易形成**寡头垄断市场**。而一旦垄断形成，极有可能会出现行业寡头凭借用户基数优势打压竞争对手的“霸权主义”（**挟用户以令天下**），阻碍行业发展，最终伤害的是广大用户的利益；即使有一天寡头丧失创新能力，其用户也会由于迁移成本过高而必须继续忍受其落后的通讯方式与不良服务体验，难以享受其他开发者提供的更优质的产品与服务。

其次，由于各家平台账号不能互通，即使寡头尚未完全形成，几家头部企业的用户相互之间也无法直接通讯。从而导致每个用户需要同时安装多个客户端、注册多个账号并频繁切换的不良体验。同时本来应该属于**用户的关系网络资源**却被平台夺取占有（成为平台最有价值的核心资源），用户自己反而**难以自由迁移**。

更进一步地，由于用户需求的多样性和快速变化等特点，很难在一款 IM 工具上承载所有的用户需求特性（这将导致一款应用过于庞大而无法使用），而同时开发和维护多款 IM 工具显然超出了一家企业的能力范围（无论这家企业多么庞大），而这正是众多具有旺盛创新能力的小团队的优势所在。所以一个有利于创新的市场，必须要有一套彻底开放的机制，以确保这种创新能力不会被巨头绞杀在摇篮里。

问题2的主要症结在于**信息安全**。由于中心化的账号系统设计，给黑客创造了一次性大批量窃取用户资料的有利条件，而去中心化的账号系统可以有效的消除这种攻击。

问题3的重点是**单点故障**风险，平台需要投入巨大资源甚至牺牲部分利益以确保其整体网络正常运行。而一旦由于某些不可描述的原因导致该平台（尤其是中心节点）停止服务，则整个 IM 网络都会立刻瘫痪，而用户除了等待平台自行修复，在有限的时间内几乎可以说是毫无选择主动权。

在中心化 IM 技术已发展完备的今天，以上问题隐患一直未能彻底消除，所以市场需要一种全新的去中心化的 IM 技术来满足更安全更丰富的通讯需求。

## 1. <span id="requirements">去中心化 IM 工具的需求</span>
IM 工具的竞争，是当今互联网世界最激烈的竞争领域之一，且已基本形成寡头垄断态势。为了在这个已被深耕多年的领域生存和发展，基于去中心化设计的 IM 工具必须要满足以下的要求，才能解决中心化 IM 所不能解决的那些问题，进而赢得广泛的应用。

### <span id="identity">去中心化的用户身份认证技术</span>
去中心化 IM，首先要解决的就是去中心化用户身份认证技术。现有的用户身份认证系统，依靠的是简单的 ID+Password 配对技术，也即需要一个可靠的数据中心去保存配对信息，因此中心化不可避免。

去中心化的用户身份认证系统，需要从算法上保证无需中心化数据库来保存配对信息，仅仅基于**共识算法**就可以实现身份认证。

### <span id="end-to-end">基于端对端加密的安全通讯技术</span>
中心化 IM 由于其服务提供者（SP）的唯一性，所有的信息安全问题都必须且只能由 SP 负责解决，用户只能选择信任 SP。而去中心化的 IM 系统中，任何人都可以成为 SP 为他人提供服务，因此我们不能完全信任任何一个 SP，从而必须在算法上保证在所有网络节点都不足信的前提之下，依然能够放心的进行通讯。

因此，基于端对端加密的通讯技术成为了必选。

### <span id="concurrence">高并发（支持亿级用户）</span>
目前所有头部的 IM 服务提供商都拥有**亿级**以上的庞大用户群体，如国内的微信、QQ 月活跃用户已近10亿，而国外的 Facebook 更是高达20亿。因此一个可以处理极其庞大用户的高并发系统设计对于去中心化 IM 技术是至关重要的。

### <span id="low-latency">低延时</span>
一个好的 IM 软件，需要有秒级的送达能力，才能保证**实时通讯**的良好体验。高延时的系统设计甚至不能称之为 IM 技术。

### <span id="free">免费使用</span>
一个可以免费供普通用户使用的 IM 软件或许将会在现有的 IM 领域中得到更为广泛的应用。

### <span id="scalable">可定制、可升级</span>
一个可根据不同人群通过定制、升级来扩展高级功能的 IM 平台，将会得到更多的开发者支持，从而衍生出更丰富的产品特性，满足更多的用户需求。

## 2. <span id="consensus">共识</span>
DIMP 引入了3类信息的共识机制：身份确认算法、关系维护机制、消息格式（含加密与校验算法），以保证整个系统稳定可靠地运行。

### <span id="identification">身份确认</span>
首先，用户账号 ID 通过一个我称之为“元”（Meta）的数据结构来生成，该生成算法确保了用户的 ID 与其非对称密钥对（PK+SK）之间的关联关系。

**ID** 是一个格式为 ```name@address[/terminal]``` 的字符串，主要包含 name 和 address 两个字段，另外一个可选字段 terminal 用于标明该 ID 当前登录设备，以区分多终端登录的情形：

1. name - 账号名，非唯一；
2. address - 由 Meta 算法计算得到的地址，在当前算力下可保证全网唯一性；
3. terminal - 登录点名称（可选项），仅用于表示同一个 ID 在不同地方登录。

**Meta** 信息是一个数据结构，包含以下 4 个字段：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| type | integer | “元”算法版本（默认为 0x01），对应于不同格式的 ID 生成算法 |
| key | PublicKey | 用户的公钥（PK）信息 |
| seed | string | 信息种子（可选），用于生成指纹信息，同时用作 ID.name |
| fingerprint | binary | 指纹信息（可选），由用户私钥（SK）对信息种子签名而得，用于生成 ID.address（区块链地址直接由 key.data 生成） |

**Meta 信息实例**，对应 ID 实例为 "hulk@4YeVEN3aUnvC1DNUufCq1bs9zoBSJTzVEj"

```javascript
{
    type        : 0x01,
    key         : {
        // 公钥算法名称
        algorithm  : "RSA",
        // 公钥数据（支持直接导入 PEM 等格式文件）
        data       : "-----BEGIN PUBLIC KEY-----\nMIGJAoGBALB+vbUK48UU9rjlgnohQowME+3JtTb2hLPqtatVOW364/EKFq0/PSdnZVE9V2Zq+pbX7dj3nCS4pWnYf40ELH8wuDm0Tc4jQ70v4LgAcdy3JGTnWUGiCsY+0Z8kNzRkm3FJid592FL7ryzfvIzB9bjg8U2JqlyCVAyUYEnKv4lDAgMBAAE=\n-----END PUBLIC KEY-----",
        // 其他参数（默认值）
        keySize    : 1024,
        encryption : "PKCS1",
        signature  : "PKCS1v15SHA256"
    },
    seed        : "hulk",
    fingerprint : "jIPGWpWSbR/DQH6ol3t9DSFkYroVHQDvtbJErmFztMUP2DgRrRSNWuoKY5Y26qL38wfXJQXjYiWqNWKQmQe/gK8M8NkU7lRwm+2nh9wSBYV6Q4WXsCboKbnM0+HVn9Vdfp21hMMGrxTX1pBPRbi0567ZjNQC8ffdW2WvQSoec2I="
}
```

当一个节点或客户端从其他节点获得一个宣称与某 ID 对应的 Meta 信息时，可以根据共识算法自行校验。如果校验通过，则确认 Meta 信息中包含的公钥（PK）信息合法，并加入本地的数据库中。由于 Meta 算法的确定性，任何一个节点或客户端都可以判断 ID & PK 的对应关系是否合法，无需第三方机构证明。

DIMP 默认的 ID.address 生成算法在 BitCoin 地址生成算法基础之上做了一点微小的升级（参见 [Address](#address) 章节），使其能包含一个人类可读的 **name** 信息。

DIMP 扩展 meta 算法可以支持使用任意一个区块链项目的地址作为 ID（此时 seed 和 fingerprint 字段为空， ID.address 直接由 key.data 生成）。
如 ```meta.type = 0x02``` 时生成的 ID 为 BTC 地址， ```meta.type = 0x04``` 时生成的 ID 为 ETH 地址。

### <span id="relationship">关系维护</span>
对于“群组”（Group）等包含关系网络的信息，我们可以采用类似于区块链的共识机制，通过签名+投票的形式实现关系维护和演变。

### <span id="encrypt-and-verify">消息加密与校验</span>
DIMP 制定了统一的消息格式与加解密/签名校验机制。每一条消息在发到网络中之前都必须事先进行加密和签名，从而确保不会被任何中间节点窃听或篡改。

## 3. <span id="account">ID（账号）</span>

DIMP 的通用账号（ID）是一个格式为 ```name@address``` 的字符串（terminal 字段以后再扩展），其中 name 为由用户指定的字符串，命名规则约定如下：

### <span id="name">Name（账号名）</span>

命名规则：

1. 长度大于 1，且不超过 32 字节；
2. 应由英文字母（区分大小写）、数字0-9、下横线“_”、横线“-”、小数点“.”等组成，不包括空格以及其他特殊字符；
3. 不能包含保留字符“@”与“/”。

其中第3条是由算法本身决定的，第1条和第2条则是出于通用性的考虑所做的建议性限制，各客户端也可以酌情考虑允许用户输入其他语言文字（不过本人认为没有这个必要，建议客户端需要显示的包含其他语言的用户名字信息以 [Visa Document](#visa) 方式提供）。

```javascript
// 举例
name = "Albert.Moky";
```

### <span id="address">Address（账号地址）</span>

DIMP 的账号地址由 Meta 算法生成，过程如下：

```javascript
// 1. 以用户私钥 SK 对种子 seed 进行签名生成指纹信息
meta.type        = 0x01;
meta.key         = PK;
meta.seed        = "moky";
meta.fingerprint = SK.sign(meta.seed); // 具体算法细节见 meta.key 参数

// 2. 对指纹信息进行组合哈希运算（sha256 + ripemd160）得到指纹摘要 digest
// 3. 将 Network ID 与指纹摘要 digest 拼接后再双哈希（sha256 + sha256）
//    然后取其结果前 4 个字节作为校验码
// 4. 将 Network ID、指纹摘要 digest、校验码 check_code 三者拼接后
//    再 base58 编码即得到账号地址
function btcBuildAddress(fingerprint, network) {
    digest     = ripemd160(sha256(fingerprint));
    check_code = sha256(sha256(network + digest)).prefix(4);
    address    = base58_encode(network + digest + check_code);
    return address;
}

// 将 name 与 address 组合，即得到 ID （字符串格式：“name@address”）
ID.name    = meta.seed;
ID.address = btcBuildAddress(meta.fingerprint, network);
```

校验过程如下：

```javascript
// Meta algorithm
function isMatch(ID, meta) {
    // 1. 首先检查 Meta 信息中的 seed、key、fingerprint 与 ID.name 是否对应
    if (meta.seed != ID.name) {
        return false;
    }
    if (meta.key.verify(meta.seed, meta.fingerprint) == false) {
        return false;
    }
    
    // 2. 再由 Meta 算法生成其对应的地址，检查是否与 ID.address 相同
    address = btcBuildAddress(meta.fingerprint, ID.address.network);
    if (address != ID.address) {
        return false;
    }
    
    // 3. 以上全部通过，则表示匹配成功，可以接受此 meta.key 作为该账号的身份公钥
    return true;
}
```

上面提到的 **Network ID** 为 1个字节长度的字符数据，表示该 ID 的类型和用途。在 DIMP 规范中，个人账号统一取值为 ```0x08```，群组为 ```0x10```，详见 [dimc-objc:/mkm/entity/address](https://github.com/moky/dimc-objc/blob/master/Classes/mkm/entity/MKMAddress.h) 的定义（其他类型暂时用不到，以后需要再由组委会讨论扩展）。

## 4. <span id="group"> Group（群组）</span>
超过2人参与的聊天谓之“群”（Group）。根据人数多少和存续时间长短可以采用不同的实现方式。

确认群指令的有效性，采用 **POP（Proof of Permission，权限证明）** 机制，即依据共识相互约定每个角色的权限，然后每一项操作必须由拥有此操作权限的角色成员签名方为有效。

### 4.1. <span id="polylogue">Polylogue - 多人会话（临时讨论组）</span>
对于少数人参与的群（例如少于 100 人），可以采用“临时讨论组”方式实现。

Polylogue 是一个由客户端维护的**虚拟群**，任何一个群成员邀请其他用户时，遵循向当前所有成员群发 **invite** 指令的规则；当某位成员希望退出该群，则群发 **quit** 指令；仅群创建者（群主）可以驱逐成员，通过群发 **expel** 指令实现。

每个用户需要用自己的私钥对指令进行**签名**，每个客户端接收到一条群指令后，需验证其身份与签名，验证通过后，如实记录在本地数据库中。

#### 建群操作
首先通过 ID 生成算法得到群 ID：

```javascript
// 生成元信息
meta.type        = 0x01;
meta.key         = founder.PK;
meta.seed        = "Polylogue-1234567890"; // 随机字符串
meta.fingerprint = founder.SK.sign(meta.seed);

// 生成 ID
ID.name    = meta.seed;
ID.address = btcBuildAddress(meta.fingerprint, MKMNetwork_Polylogue);
```

然后将群 ID 发给初始成员。每位成员收到后，在本地保存为一个**临时会话**记录。
后续所有通讯都必须在 message.content.group 中夹带群 ID，以便接收方知道这是一个群消息。

#### 添加群成员
生成 **invite** 指令：

```javascript
{
    type    : 0x88,      // DIMMessageType_Command
    sn      : 794594362,
    group   : "Polylogue-1234567890@7XrUr4staRFC5xu7iCqXRMpbawAGMyASUR",
    
    command : "invite",
    member  : "hulk@4YeVEN3aUnvC1DNUufCq1bs9zoBSJTzVEj"
}
```

然后加密、签名，得到 Reliable Message（参照 [消息](#message) 章节）。再发送给全部已有成员即可。

#### 移除成员
如上，**command** 字段为 ```expel```，且仅限群主操作。

#### 请求入群
如上，**command** 字段为 ```join```，没有 **member** 字段。

#### 退群
如上，**command** 字段为 ```quit```。

#### 查询群信息
如上，**command** 字段为 ```query```，仅限现有群成员查询。

#### 群发消息
详见[多人群聊](#group-message)章节。

### 4.2. <span id="chatroom">Chatroom - 聊天室（固定群/大规模群聊天）</span>
对于参与人数较多的群（例如 100 人以上），由于需要添加**管理员(Administrator)**角色以协助管理，并且需要允许转让**群主(Owner)**身份，相对比较复杂，所以我这里建议使用**区块链**技术来记录**群历史**信息。

具体实现方式是每个群一条**区块链**，打包区块的权限认定采用 **POP 机制**，即通过共识定义群角色的权限，然后由该角色成员对数据进行签名并打包写入区块。

_（注：DIM 项目第一阶段由于使用人数较少，暂时无需实现这部分功能，建议所有群聊天都采用 **Polylogue** 方式）。_

## 5. <span id="message">消息</span>

本协议首先定义了点对点的聊天（私聊）消息格式，然后在此基础上扩展出多对多的聊天（群聊）消息格式。

以下是点对点消息格式规范说明：

### 5.1. <span id="envelope">消息头（Envelope）</span>
每一个消息都包含3个字段作为消息头：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| sender | ID | 发送方 ID |
| receiver | ID | 接收方 ID (or 群 ID) |
| time | Date | 发送时间 |

### 5.2. <span id="content">消息内容（Content）</span>
每一份消息内容都包含 2 个公共字段 ```type, sn``` 以区分不同的消息：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| type | integer | 消息类型（对应 text、file、image、audio、video、command 等） |
| sn | integer | 消息序列号（正整数，由发送方生成，以唯一标识具体某个消息） |
| ... | ... | （其他字段） |

这里列举几个[常规消息内容类型](DIMP-Contents.md)的格式规范。

### 5.3. <span id="message-structure">消息包数据结构</span>
以下是 DIMP 定义的 3 级消息格式，其中 **Instant Message** 为收发双方客户端保存的明文消息格式；**Secure Message** 是打包/解包过程中产生的中间格式；只有 **Reliable Message** 会发到网络上进行传送：

```javascript
Message transformations:
    
        Instant Message <-> Secure Message <-> Reliable Message
        +-------------+     +------------+     +--------------+
        |  sender     |     |  sender    |     |  sender      |
        |  receiver   |     |  receiver  |     |  receiver    |
        |  time       |     |  time      |     |  time        |
        |             |     |            |     |              |
        |  content    |     |  data      |     |  data        |
        +-------------+     |  key/keys  |     |  key/keys    |
                            +------------+     |  signature   |
                                               +--------------+
        Algorithm:
            data      = PW.encrypt(content)
            key       = receiver.PK.encrypt(password)
            signature = sender.SK.sign(data)
```

- <span id="instant-message">**原始消息包（Instant Message）**</span>

发送方发出的消息（打包处理前）、接收方收到后的消息（解包处理后），格式如下：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| sender | ID | 发送方 ID |
| receiver | ID | 接收方 ID (or 群 ID) |
| time | Date | 发送时间 |
| content | Content | 消息内容（明文） |

- <span id="secure-message">**加密消息包（Secure Message）**</span>

客户端发出消息前，先对原始消息进行加密，所得到的中间消息包（content 字段替换成了 data），格式如下：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| sender | string | 发送方 ID |
| receiver | string | 接收方 ID (or 群 ID) |
| time | integer | 发送时间戳 |
| data | binary | 加密消息内容（用一个随机生成的对称密码 PW 对 content 进行加密） |
| key | binary | 对称密码信息（用接收方公钥对 PW 进行加密） |

加密算法如下：

```javascript
string = json(content);            // 1. 先将 content 序列化为一个字符串
PW     = random();                 // 2. 生成随机密码（对称加密密钥）
data   = PW.encrypt(string);       // 3. 对序列化字符串进行加密并替换
key    = receiver.PK.encrypt(PW);  // 4. 对随机密码进行非对称加密
```

- <span id="reliable-message">**网络传输包（Reliable Message）**</span>

用发送方私钥对中间消息包（Secure Message）进行签名，最终得到的网络消息包，格式如下：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| sender | string | 发送方 ID |
| receiver | string | 接收方 ID (or 群 ID) |
| time | integer | 发送时间戳 |
| data | binary | 加密消息内容（用一个随机生成的对称密码 PW 对 content 进行加密） |
| key | binary | 对称密码信息（用接收方公钥对 PW 进行加密） |
| signature | binary | 签名信息（用发送方私钥对 data 进行签名 |

签名算法如下：

```javascript
// 具体算法细节由 meta.key 定义
signature = sender.SK.sign(data);
```

最终发送到 DIM 网络中的只有 Reliable Message 消息包，DIM 网络仅负责快速正确投递消息包，无法解密其内容、也无法篡改冒充，在当前世界算力水平下可确保足够安全。以下是一个测试实例：

```javascript
/**
 *  网络消息包实例
 *  
 *  Algorithm:
 *      // 1. 加密，并将明文的 content 字段替换为加密的 data
 *      string = json(content);
 *      PW     = random();
 *      data   = PW.encrpyt(string);       // Symmetric
 *      key    = receiver.PK.encrypt(PW);  // Asymmetric
 *      // 2. 签名
 *      signature = sender.SK.sign(data);  // Asymmetric
 */
{
    //-------- head (envelope) --------
    sender   : "moki@4WDfe3zZ4T7opFSi3iDAKiuTnUHjxmXekk",
    receiver : "hulk@4YeVEN3aUnvC1DNUufCq1bs9zoBSJTzVEj",
    time     : 1544106533,
    
    //-------- body (content) ---------
    data      : "1e8OshcP8Z1XBf49ABJkTGNbIVWS8HjD2DCVEv7HmzMv4LqMKdZBSr4wvf4lXrAk",
    key       : "MnaepvMge7eSSKGeYr2YYblvQr3DPVb3xe3HBC4u5BScusHydQ4/lx0Vl3rvzC3uLMdGVN+BG4qmjYYt53hLSCJhwfwwLynuw/ldSeABQG9t0ObKHnpgAwkvchfDINs2ssz6QgD9bDuV1WzwH49ycNTr84Wa12vXzjERJYalpvw=",
    signature : "oKcdCzYfysL5CJNCkgRUfeiRGG5AfEWc6GPerLafUbFWW+sij1codi3kZCiHiBlC4Ya4D/I+2xST78A0GME8P1b//LRP+/4Lh8tOE4qRPjj/G8eWSXvjsLkRbjiLeNmNHiD74BR84/Q0d18T0dlP5hQ30DzBKzauZzrhMas89kc="
}
```

## 6. <span id="cryptography">密码</span>

### 6.1. <span id="identity-key">身份密钥 (Identity Key)</span>

用户身份由非对称密码学验证确认。

用户在客户端生成个人私钥，然后用私钥构建 meta 信息以及生成 ID，此后所有需要身份验证的地方都由 meta.key 进行验证。

### 6.2. <span id="communication-key">通讯密钥 (Communication Key)</span>
端对端加密通讯消息需要用到对称与非对称两层密钥。

- <span id="symmetric-key">**对称密钥**</span>

1. 对称密钥为消息发送方随机生成的密钥，用于对消息主体（content）进行加/解密；
2. 对称密钥有方向属性，即每一个**&lt;FROM, TO&gt;向量**对应一个密钥，其中 **FROM** 即发送方（sender）地址，**TO** 为接收方（receiver or group）地址；
3. 在一个由 N 个人组成的群聊天中，每一个成员分别维护一把 **&lt;MemberID, GroupID&gt;** 的密钥，即一共存在 N 个对称密钥；
4. 对称密钥可以重用（比如每4小时定期更换一次），由消息发送方负责维护，建议重用时以 6 字节摘要信息填充 key/keys 字段，以便接收方判断密钥是否已更改；
6. 默认算法：“**AES/CBC/PKCS7Padding**”。

- <span id="asymmetric-key">**非对称密钥**</span>

1. 非对称密钥为每个用户自己生成的密钥对，用于对非对称密钥进行加/解密，默认是用户注册时生成的 meta.key 及其对应的 private key；
2. 每个消息发送方必须先取得接收方的公钥（默认是 meta.key，当存在 visa document 时，则为 visa.key），然后用它为**对称密钥(&lt;SenderID, ReceiverID&gt;)**加密，并将加密结果附在 ReliableMessage 的 key 中发送给接收方；
3. 在一个由 N 个人组成的群聊天中，发送方需要用每一个群成员的公钥为**对称密钥(&lt;SenderID, GroupID&gt;)**进行加密，并将加密结果附在 ReliableMessage 的 keys 中发送给每一个接收方；
4. 由于对称密钥支持重用，所以在对称密钥不变的情况下，以上两步加密操作可以省略，接收方在消息中没有找到 key/keys 字段时，应从之前的记录中获取相应的密码进行解密；
5. 默认算法："**RSA/ECB/PKCS1Padding**"；
6. 非对称通讯密钥可以更换，具体逻辑参见 [扩展协议::用户资料广播协议 (Visa Protocol)](#visa)。

## 7. <span id="message-process">消息处理流程</span>

### 7.1. <span id="personal-message">二人私聊</span>

当一个用户需要发送消息给另一个用户时，客户端需要先执行以下3个步骤，再将计算结果打包发送到 DIM 网络中：

1. 随机生成一个对称加密密码 PW，用此密码对消息内容 content 进行**对称加密**，得到加密数据 data；
2. 用接收方的公钥 PK-2 对 PW 进行**非对称加密**，得到密钥信息 key；
3. 用发送方的私钥 SK-1 对 data 进行**签名**（先求取密文的摘要，再对摘要进行签名），得到签名信息 signature。

相应地，消息的接收也需要3个对应步骤：

1. 用发送方的公钥 PK-1 对密文 data 和签名 signature 进行**校验**（先求得密文的摘要，再对摘要进行校验）；
2. 用接收方的私钥 SK-2 对 key 进行**非对称解密**，得到对称加密密码 PW；
3. 用密码 PW 对密文 data 进行**对称解密**，还原出消息内容 content。

通过以上算法，既能确保消息不被任何中间节点监听窃取，也能防止第三方冒充篡改，从而实现安全可靠的去中心化通讯。

### 7.2. <span id="group-message">多人群聊</span>

在多人群聊场景中，可以基于上述的“二人私聊”逻辑来实现，具体根据实现复杂度可以有多种实现方式。其中最简单的是由客户端负责分包，这种方式实现逻辑最简单，私密性最好，对网络依赖性最低，但同时对带宽消耗也最大：

#### 方式一：客户端自行分包

消息发送的过程跟二人私聊的发送过程大致相似：

1. 首先，消息打包的时候需要在消息内容 content 中增加 **group** 字段，填入对应的 Group ID，然后再开始进行打包发送操作；
2. 打包过程第一步跟私聊相同，由消息发送者生成一个随机的对称密码 PW，并用 PW 将待发送消息内容加密为密文 data；
3. 然后遍历**所有群成员**，用每一位成员的公钥对 PW 分别进行加密，得到各自对应的 key；
4. 接着用发送者私钥对密文 data 进行签名，将签名信息 signature 与前一步得到的 key 一起，为每位成员分别打包成 **Reliable Message**；
5. 最后将所有 Reliable Message 消息包一起发送到 DIM 网络中。

由于接收方收到的直接就是 Reliable Message 消息包，所以解包过程跟私聊的情况几乎完全一样，只是在得到消息内容 content 之后，需要进一步检查消息内容中的 group 字段，然后交由相应的群消息处理器进行处理。

#### 方式二：服务端协助分包

由于每一条群消息的对称加密结果 data 和签名信息 signature 都是相同的，不同的仅仅是为每位群成员准备的 key，所以为了节省流量，客户端可以把所有 keys 合并成一个字典，与 data、signature 打包在一起组成**群消息包**，然后 receiver 字段填 Group ID，再将生成的群消息包发送到服务器端，由服务器端根据群成员列表进行拆包（逐个检出每位成员的 key 重新生成新包），最后 DIM 网络会负责将每一个 Reliable Message 消息包投递给对应的群成员。

由于省去了大量不必要的重复信息，所以可以大大节省发送方的带宽，但是要实现此功能，需要客户端（在用户确认授权的前提下）将所有群成员列表同步给服务端，并且服务端需要维护最新的群成员列表，以及记录每一个 key 历史以支持 PW 复用。

**群消息包**格式如下：

```javascript
{
    sender   : "SENDER_ID",
    receiver : "GROUP_ID",
    time     : 1544863058,
    
    data      : "base64_encode",    // PW.encrypt(content)
    signature : "base64_encode",    // sender.SK.sign(data)
    keys      : {"ID1":"key1", ...} // encrypted keys (PW) for each member in the group
}
```

当客户端确认当前连接的服务端支持 PW 重用时，可以在后续消息包中省略掉 keys 字段（仅传输 PW 后 6 字节的 digest 以确认是否为同一个 PW），进一步减少上行带宽的使用。

### 7.3. 优化功能点

1. **【密钥重用】** 为了节省流量与算力，PW 应该允许复用，即在一定时间间隔内（比如24小时）发消息时无需再生成新的 PW（也无需再把该 PW 打包成 key），客户端应保存每一个“**消息方向**”所对应的历史 PW。当接收到的新消息中找不到对应的 key 时，应该根据该“消息方向”从历史记录中寻找复用 PW；
2. **【密钥查询】** 考虑到可能丢失消息包（或者消息包顺序错乱）的情况，在升级优化功能点1的同时，需要支持查询 PW 的功能，以便接收方在无法找到有效的 PW 时，可以向消息发送方请求获取 PW，或者直接要求对方重发该消息（带上 key 字段）；
3. **【消息压缩】** 为了节省数据流量，在发送较长消息内容时，可以考虑先压缩再加密（压缩/解压需要额外消耗算力），要实现此功能只需要**扩展对称密钥算法**即可（比如 algorithm 由 "AES" 升级为支持 "AES+ZIP"，压缩参数也可以写进 key 字典），前提是接收方也需要支持相同的对称算法。
4. **【消息免签】** 对于已经通过握手等方式建立信任关系的收发双方，可以共同约定一个 **session key**（加有效期），此后每条消息都将该字符串放进 ```message.content['session']``` 中再加密；接收方解密后验证其是否有效即可。此功能可有效减少流量与算力的消耗，缺点是仅能用于点对点聊天，不适宜用于群聊，并且需要 station 支持“免签过境”特权（除非不经过任何 station 直接建立连接）。

## 8. <span id="extensions">扩展协议</span>

以下子协议非 DIMP 核心，但在实际应用中仍然十分重要。

### 8.1. <span id="handshake">握手协议（身份验证）</span>

为了确认用户身份，以便正确的投递消息包（虽然投递到错误地址也不会造成消息泄密，但是可能会导致真正的接收方丢失消息），Station 应该在收到客户端的连接请求时确认对方身份是否合法。因此基于 DIMP 扩展出此协议。

第一步，Client 向 Station 发起连接，并连接成功之后，向服务器端发送第一个 **Say Hello** 消息包：

```javascript
/* 1. 构建系统命令消息包 Instant Message */
{
    sender   : "USER_ID",
    receiver : "STATION_ID",
    time     : 1542156395,
    
    content  : {
        type    : 0x88, // DIMMessageType_Command
        sn      : 1234,
        command : "handshake",
        message : "Hello world!" // Hi!
    },
    
    meta     : {
        // 与 user ID 对应的 meta 信息（无需加密）
        // 如果是新用户，station 会在校验正确后保存此信息
        // 如果 station 已保存过该信息，则忽略之
    },
    visa     : {
        // 由 user.SK 签名的 visa document，内含 communication key（可选）
        // station 会检查生成时间，以更新缓存
    }
}
/**
 *  2. 然后加密+签名：
 *    PW        = random();
 *    data      = PW.encrypt(json(content));
 *    key       = station.PK.encrypt(PW);
 *    signature = user.SK.sign(data);
 *  生成 Reliable Message 再发送到 Station
 */
```

附件说明：如果是新创建的账号，第一次连接时 station 上并无该账号的 meta & visa document，也就无从验证消息签名，所以需要在第一个 handshake 消息中添加 meta & visa 附件，之后的连接请求则不需要再附带 meta 等信息。

第二步，如果 Station 收到 Client 的 Say Hello 消息包，或者当 Station 检测到该用户 ID 的收件箱里有新消息，需要向 Client 推送消息之前（如尚未确认身份），必须先发送**身份验证请求包**：

```javascript
/* 1. 构建系统命令消息包 Instant Message */
{
    sender   : "STATION_ID",
    receiver : "USER_ID",
    time     : 1542156754,
    
    content  : {
        type    : 0x88, // DIMMessageType_Command
        sn      : 2345,
        command : "handshake",
        message : "DIM?", // What's up?
        session : "RANDOM_STRING" // 由 Station 生成的随机字符串
    }
}
/**
 *  2. 然后加密+签名：
 *    PW        = random();
 *    data      = PW.encrypt(json(content));
 *    key       = user.PK.encrypt(PW);
 *    signature = station.SK.sign(data);
 *  生成 Reliable Message 再发送到 Client
 */
```

第三步，Client 收到 Station 发送的身份验证请求包后，必须回复一个**身份确认响应包**：

```javascript
{
    sender   : "USER_ID",
    receiver : "STATION_ID",
    time     : 1542157677,
    
    content  : {
        type    : 0x88, // DIMMessageType_Command
        sn      : 3456,
        command : "handshake",
        message : "Hello world!", // It's me!
        session : "RANDOM_STRING" // 由 Station 生成的随机字符串
    }
}
/* 同样需要加密+签名 */
```

第四步，如果 Station 验证后发现 session 信息不匹配，则拒绝服务并断开链接，否则回复**身份确认消息包**并继续后续通讯：

```javascript
{
    sender   : "STATION_ID",
    receiver : "USER_ID",
    time     : 1542157935,
    
    content  : {
        type    : 0x88, // DIMMessageType_Command
        sn      : 4567,
        command : "handshake",
        message : "DIM!" // OK!
    }
}
/* 同样需要加密+签名 */
```

### 8.2. <span id="broadcast">广播消息通讯协议</span>

与普通的[网络传输包(Reliable Message)](#reliable-message)类似，广播消息包(Broadcast Message)也可以在网络中传播。不同点主要有两个：

1. 广播消息只有 sender，没有 receiver（实际```receiver = "everyone@everywhere"```），所以每个节点都会收到；
2. 广播消息只有签名信息，没有解密密钥，因为内容是明文的（直接将 content 序列化）。

格式如下：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| sender | string | 发送方 ID |
| receiver | string | 恒等于 "everyone@everywhere" |
| time | integer | 发送时间戳 |
| data | string | 明文广播信息（JsON） |
| signature | binary | 签名信息（用发送方私钥对 data 进行签名 |
| traces | array | 广播路径记录，作为广播消息的附件一起发送 |

**广播消息格式示例**：

```javascript
{
    sender    : "USER_ID",
    receiver  : "everyone@everywhere", // constant
    time      : 1502119527,
    
    data      : "PLAIN_TEXT",          // json_encode(content);
    signature : "BASE64_ENCODE",       // user.SK.sign(data);
    
    traces    : [
        { /* 途经的每个 station 信息（含到达时间） */ },
    ]
}
```

### 8.3. <span id="visa">用户资料广播协议 (Visa Protocol)</span>

由于 meta 信息的不可变性，为了降低“历史攻击”的风险（攻击者一直存储特定用户的网络传输包，等待某天成功窃取到该用户私钥后再解密的一种攻击方式），需要扩展一个协议来支持通讯密钥和 meta.key 分离。

另外用户的其他资料（如昵称、头像等）也需要一套协议来同步信息，所以特此提出该协议。

Visa Document 信息结构如下：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| ID | string | 用户 ID |
| data | string | 用户资料信息（JsON） |
| signature | binary | 对 data 的签名 |

第一步，填写用户资料信息：

```javascript
info = {
    // 通讯密钥
    key : {
        algorithm: "RSA",
        data     : "..."
    },
    
    // 其他信息
    name   : "Albert Moky",
    avatar : "https://",
    // ...
}
```

第二步，签名生成 visa document：

```javascript
document = {
    ID        : "USER_ID",
    /**
     *    1. 先将 info 转换为 JsON 字符串 data
     *    2. 再对 data 进行签名
     */
    data      : "PLAIN_TEXT",   // json_encode(info);
    signature : "BASE64_ENCODE" // user.SK.sign(data);
}
```

第三步，打包 visa command：

```javascript
content = {
    type    : 0x88, // DIMContentType_Command
    sn      : 1234,
    command : "visa",
    visa    : document
}
```

第四步，将此信息打包作为广播消息发送出去：

```javascript
{
    sender    : "USER_ID",
    receiver  : "everyone@everywhere", // constant
    time      : 1502119527,
    
    data      : "PLAIN_TEXT",          // json_encode(content)
    signature : "BASE64_ENCODE",       // user.SK.sign(data);
    traces    : []
}
```

#### 登录授权

Visa Protocol 的另一个用途是多终端登录时，可以允许在不导出身份密钥的情况下授权另一个设备以此身份登录使用；
更进一步地，也可用于授权第三方应用使用该身份。

具体实现逻辑为：

1. 新客户端生成通讯密钥 Communication Private Key (CSK) 和 Public Key (CPK)；
2. 新客户端生成 Visa Document 并将 CPK 包含进去；
3. 新客户端请求原客户端用 Identiti Key 对该 Visa 信息进行签名；
4. 原客户端对该 Visa 签名并广播（或发回给新客户端）；
5. 此后新客户端即可以此身份进行通讯。

值得注意的是：

1. 由于 Identity Key 在原客户端上，没有导出到新客户端，所以新客户端需要对消息进行签名时，只能用 CSK 签名；
2. 同时 DIMP 规定 Visa Document 只能由该 ID 所对应的 Identity Key 签发，所以新客户端也不能用 CSK 对其他客户端进行转授权，即所有新客户端都必须由持有 Identity Key 的原客户端进行授权方可使用；
3. 原客户端在为新客户端签发 Visa Document 的时候，可以增加有效时间、用途等限制条件，以明确指定该 Visa 的使用范围；
4. 必要的时候，原客户端可以广播一个更新 Visa Document，以实现对之前签发的 Visa Document 的撤销（需要所有 SP 支持）。

### 8.4. <span id="login">登录点信息广播协议</span>

特别地，随着用户量增加，DIM 网络中的 Station 也会越来越多，为了更快捷高效地转发消息，需要增加一个子协议以协助 DIM 网络计算最短传输路径（即**路由算法**）。因此这里提出一个扩展建议：

1. 扩展一个包含当前连接的 Station 信息的数据结构（含签名），给到 Station；
2. Station 在收到此消息包并验证+去重后，更新本地数据库并向全网广播此数据结构；
3. 每一个 Station 在收到此数据结构时，将自身信息加入到该结构所附带的传播路径（traces）并转发给其余已建立连接的 Station。

信息结构如下：

```javascript
/* 登录命令 */
content = {
    type    : 0x88, // DIMMessageType_Command
    sn      : 1234,
    command : "login",
    
    login   : {
        // 当前登录的基站(节点)信息
        provider : "SP_ID",
        station  : "STATION_ID",
        host     : "211.66.6.1",
        port     : 9394,
        time     : 1542157677,  // 登录时间
        
        // 终端(客户端)信息
        account   : "USER_ID",
        terminal  : "DEVICE_ID", // 终端标识（可选）
        userAgent : "USER_AGENT" // 其他信息（可选）
    }
}
```

然后将登录命令用广播消息发送出去：

```javascript
/* 登录广播消息 */
{
    sender    : "USER_ID",
    receiver  : "everyone@everywhere", // constant
    time      : 1542157677,
    
    data      : "PLAIN_TEXT",          // json_encode(content)
    signature : "BASE64_ENCODE",       // user.SK.sign(data);
    traces    : []                     // 广播路径记录，作为广播消息的附件一起发送
}
```

### 8.5. <span id="receipt">消息包确认签收协议</span>

为追踪用户消息包到达情况，可扩展**消息签收**子协议。

```javascript
{
    sender   : "RECEIVER_ID",
    receiver : "SENDER_ID",
    time     : 1544779337,
    
    content  : {
        type    : 0x88,    // DIMMessageType_Command
        sn      : 1234,    // 原始消息的 sn
        
        sender   : "...",  // 原始消息的 sender （默认为 SENDER_ID）
        receiver : "...",  // 原始消息的 receiver （默认为 RECEIVER_ID）
        group    : "...",  // 如果是群消息收据，则须带上群 ID
        
        command : "receipt",
        message : "...",
        // extra info
        // ...
    }
}
```

### 8.6. <span id="contacts">好友列表</span>

由 Station 提供的增值服务。

客户端成功与某 Station 建立连接后，可以选择是否将自己的通讯录列表作为**好友列表**提交给该 Station，如果这样做，则可享受其提供的**自动过滤垃圾消息服务**：在此期间，Station 将自动丢弃所有发送者 ID 不在好友列表中的消息（群消息除外）。

### 8.7. <span id="blocked">屏蔽列表</span>

遇到骚扰账号时，可以将其 ID 加入到本地的屏蔽列表，则所有发送者 ID 在屏蔽列表中的消息将不会被显示（客户端自动屏蔽）；

同时，当客户端成功与可以提供**自动屏蔽骚扰消息服务**的 Station 建立连接后，可以选择是否将本地的屏蔽列表提交给该 Station，如果这样做，则 Station 将会自动丢弃所有发送者 ID 在屏蔽列表中的消息（群消息除外）。

### 8.8. <span id="ant-porter">蚂蚁搬运工</span>

由 Service Provider 提供的增值服务。

当用户需要访问**网络距离**十分遥远的资源时，SP 可以优化其服务器群以提供这样的服务：将大量用户重复访问的大文件（如视频）搬到本地缓存，以使其用户可以就近访问，极大地提高访问速度；同时由于避免了大量重复下载（同一份文件只需要搬运一次），也大大减少了跨网带宽资源的浪费，降低网络拥堵几率。

特别地，如果是用户 A 向用户 B 发送一个大文件，常规方式是用户 A 先将文件上传到就近的服务器，再将其 URL 放到消息体中发送给 B；用户 B 收到消息后，通过 URL 从用户 A 附近的服务器下载文件。而优化的方案则是 SP 判断到用户 B 距离该资源十分遥远时，其所在 SP 可以预先将该文件搬运到本地，如此则用户 B 登录后便可以从就近的服务器快速下载该文件。

该服务对于有**跨国沟通**需求的用户尤其重要，相信可以极大地提高用户体验。

## 9. <span id="conclusion">结论</span>
祝帝企鹅20周岁生日快乐！🎂
