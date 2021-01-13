# Contents for DIMP

[![license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/moky/DIMP/blob/master/LICENSE)
[![Version](https://img.shields.io/badge/alpha-0.1.0-red.svg)](https://github.com/moky/DIMP/wiki)

## DIMP 消息内容格式规范

每一份消息内容都包含 2 个公共字段 ```type, sn``` 以区分不同的消息：

| 字段名 | 类型 | 说明 |
|-------|:---:|------|
| type | integer | 消息类型（对应 text、file、image、audio、video、command 等） |
| sn | integer | 消息序列号（正整数，由发送方生成，以唯一标识具体某个信息） |
| time | Date | 发送时间（可选），由消息发送方给出的消息发送时间戳 |
| group | ID | 群 ID（可选），标志该消息属于哪个群 |
| ... | ... | （其他字段） |

下面列举几个常见的消息内容类型的格式规范：

- 0x01 - <span id="text-message">**文本消息**</span>

```javascript
{
    type : 0x01, // DIMMessageType_Text
    sn   : 1234,
    
    text : "Hello world!"  // say anything
}
```

- 0x10 - <span id="file-message">**文件消息**</span>

```javascript
{
    type : 0x10, // DIMMessageType_File
    sn   : 1234,
    
    URL      : "http://", // encrypt & upload to CDN
    filename : "dir.zip"
}
```

- 0x12 - <span id="image-message">**图片消息**</span>

```javascript
{
    type : 0x12, // DIMMessageType_Image
    sn   : 1234,
    
    URL      : "http://",       // encrypt & upload to CDN
    snapshot : "BASE64_ENCODE", // base64_encode(smallImage)
    filename : "photo.png"
}
```

- 0x14 - <span id="audio-message">**语音消息**</span>

```javascript
{
    type : 0x14, // DIMMessageType_Audio
    sn   : 1234,
    
    URL  : "http://", // encrypt & upload to CDN
    text : "ASR_TEXT" // Automatic Speech Recognition
}
```

- 0x16 - <span id="video-message">**视频消息**</span>

```javascript
{
    type : 0x16, // DIMMessageType_Video
    sn   : 1234,
    
    URL      : "http://",      // encrypt & upload to CDN
    snapshot : "BASE64_ENCODE" // base64_encode(smallImage)
}
```

- 0x20 - <span id="web-page">**网页消息**</span>

```javascript
{
    type : 0x20, // DIMMessageType_Page
    sn   : 1234,
    
    URL   : "http://",       // Web Page URL
    icon  : "BASE64_ENCODE", // base64_encode(icon)
    title : "...",
    desc  : "..."
}
```

- 0x37 - <span id="quote-reply">**引用回复**<span>

```javascript
{
    type : 0x37, // DIMMessageType_Quote
    sn   : 5678,
    
    quote : 1234, // referenced serial number of previous message
    text  : "I like it!"  // say anything
}
```

- 0x88 - <span id="system-command">**系统命令**</span>

```javascript
{
    type : 0x88, // DIMMessageType_Command
    sn   : 1234,
    
    command : "...", // command name
    params  : ...    // extra parameters
}
```

- 0xFF - <span id="forward-message">**转发消息**</span>

此类格式的消息常用于消息发送者希望隐藏消息路径的使用场景。其中```forward```字段包含的是需要转发的实际消息包（已加密+已签名）。

***转发消息的实施过程***：

1. 发送方先创建一个包含实际消息内容的**待发送信息包**（加密+签名）；
2. 发送方将整个实际消息包进行二次打包，生成一个发给提供转发服务的 Station 的消息（receiver 为 **Station ID**，消息类型为 **DIMMessageType_Forward**）；
2. 该 Station 收到后，先验证并用自己的私钥解密，判断消息类型是否为 DIMMessageType_Forward；
3. 如果是，则读取```forward```的信息头，获取**实际接收方 ID**，并将```forward```重新打包，生成一个由 Station 发给实际接收方的消息包并发送到 DIM 网络中（sender 为 Station ID，receiver 为实际接收方 ID，消息类型为 DIMMessageType_Forward）；
5. 实际 receiver 收到数据包后，解开并判断消息类型，如果是 **DIMMessageType_Forward** 则对```forward``内容**递归执行解包操作**，直到获得最终的真实消息内容。

```javascript
{
    type : 0xFF, // DIMMessageType_Forward
    sn   : 5678,
    
    forward : { // top-secret message
        sender   : "moki@4WDfe3zZ4T7opFSi3iDAKiuTnUHjxmXekk",
        receiver : "hulk@4YeVEN3aUnvC1DNUufCq1bs9zoBSJTzVEj",
        time     : 1542075610,
        
        data      : "BASE64_ENCODE", // top-secret content
        key       : "BASE64_ENCODE",
        signature : "BASE64_ENCODE"
    }
}
```

***应用示例***：

* 请参考 [无痕通讯网络 (Invisible Network)](InvisibleNetwork.md)。

Copyright &copy; 2018-2021 Albert Moky
