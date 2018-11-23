# Decentralized Instant Messaging Protocol (DIMP)

## 0. Abstract
This document introduces a new protocol designed for instant messaging and an architecture for developing decentralized IM applications. The software provides accounts(user identity recognition) and communications(instant message) between accounts safely by end-to-end encryption.

It includes just TWO extremely simple parts:

1. User Identify
2. Messaging

### White papers

- [English](TechnicalWhitePaper.md)
- [Chinese](zh-CN/TechnicalWhitePaper.md)

## 1. User Identify

### 1.0. Meta
The **Meta** was generated by your private key,
it can be used to build a new ID for entity, or verify the ID/PK pair.

The 'Meta' consists of 4 fields:

| Field       | Description                   |
| ----------- | ----------------------------- |
| version     | Meta Algorithm Version        |
| seed        | Entity Name                   |
| key         | Public Key                    |
| fingerprint | Signature to generate address |

```
/* example: hulk */
{
    version     : 0x01,
    seed        : "hulk",
    key         : {
        algorithm : "RSA",
        data      : "-----BEGIN PUBLIC KEY-----\nMIGJAoGBAKHnPiSva9pjkTKYqfP1beikQBriPqGUPPAoBYUF5kwk+r3BfKswcWGV\nKGyuS++VYk5SyaiMLrmDMFETf26XE6yWPa+lakTg+/QgQdyE7/pIfbngdfxvWWO7\nTLRr6Q/Am61Otkb12kgmiJmLTDLV5a+6L19f85+g7YKvbL4G0k9BAgMBAAE=\n-----END PUBLIC KEY-----"
    },
    fingerprint : "cPeswyjeFKMgw963GcptO4aiwriAHXYImmJH+nxlLvvahPOLOO/Usi8hEGR1NUGg4iDFj9TzwyV7WhJ/X7bHB1/YcU5rouhEDn8XTqhR2hOkbn7UvlF4ASaB21e4ibDKjri4vQY0w8HY32GdxvR5BMMtE+DaaFOZKPHKwGTSNGc="
}
```

THe **fingerprint** field was generated by your **private key** and **seed**:

````
fingerprint = sign(seed, SK);
````

### 1.1. ID
The **ID** is used to identify an **entity**(account/group). It consists of 3 fields:

| Field       | Description                   |
| ----------- | ----------------------------- |
| name        | Same with meta.seed           |
| address     | Unique Identification         |
| terminal    | Login point, it's optional.   |

The ID format is ```name@address[/terminal]```.

```
/* examples */
ID1 = "hulk@4Qv359gss3FrZpZ2phxykvofmt9fyXx5gJ"; // immortal hulk
ID2 = "moki@4HaXeu62Q41eemWcL1X5m56Y5JwKK2JJUU"; // monkey king
```

#### Name
The **Name** field is an entity name (username, or just a random string for group).

1. The length of name must more than 1 byte, less than 32 bytes;
2. It should be composed by a-z, A-Z, 0-9, or charactors '_', '-', '.';
3. It cannot contain charactors '@' and '/'.

```
// For example
name = "Albert.Moky";
```

#### Address
The **Address** field was created by the **Meta** info and the **Network ID**:

````
// Network ID
enum {
    MKMNetwork_Main  = 0x08, // (Person)
    MKMNetwork_Group = 0x10, // (Multi-Persons)
};

// Address algorithm
function btcBuildAddress(fingerprint, network) {
    hash       = ripemd160(sha256(fingerprint));
    check_code = sha256(sha256(network + hash)).prefix(4);
    address    = base58(network + hash + code);
    return address;
}
````

### 1.2. Public Key
A **public key** (PK) was binding to an ID by the __Meta Algorithm__,
when you get a meta for the entity ID from the network,
you must verify it with the consensus algorithm before accept it:

```
/**
 *  network = MKMNetwork_Main; // ID.address.network
 *  name    = meta.seed;
 *  PK      = meta.key;
 *  address = btcBuildAddress(meta.fingerprint, network);
 */
function isMatch(ID, meta) {
    // 1. check 'seed', 'key' & 'fingerprint' in meta with ID.name
    if (meta.seed != ID.name) {
        return false;
    }
    if (verify(meta.seed, meta.fingerprint, meta.key)) {
        return false;
    }
    
    // 2. build address with meta, compare it with ID.address
    address = btcBuildAddress(meta.fingerprint, ID.address.network);
    if (address != ID.address) {
        return false;
    }
    
    // 3. if all of the above matches, get public key from meta
    ID.publicKey = meta.key;
    return true;
}
```

### 1.3. Entity (Account/Group)
**Entity** is the sender/receiver in the network communication.

An entity can be an account or a group. It has an **ID**, a **name**, and a **number** for searching.

An **account** will have a **public key**.

A **group** will have **founder**, **owner** and **members**.

```
user = new Account(ID, PK);
```

## 2. Messaging

### 2.0. Envelope

```
{
    sender   : "moki@4HaXeu62Q41eemWcL1X5m56Y5JwKK2JJUU",
    receiver : "hulk@4Qv359gss3FrZpZ2phxykvofmt9fyXx5gJ",
    time     : 1542984590
}
```

### 2.1. Message Content

````
{
    type     : 0x01,       // message type
    sn       : 1682437361, // serial number (message ID in conversation)
    
    text     : "Hey guy!"
}
````

#### Message Type

````
enum {
    DIMMessageType_Unknown = 0x00,
    DIMMessageType_Text    = 0x01,
    
    DIMMessageType_File    = 0x10,
    DIMMessageType_Image   = 0x12,
    DIMMessageType_Audio   = 0x14,
    DIMMessageType_Video   = 0x16,
    
    DIMMessageType_Page    = 0x20,
    
    // quote a message before and reply it with text
    DIMMessageType_Quote   = 0x37,
    
    // system command
    DIMMessageType_Command = 0x88,
    
    // top-secret message forward by proxy
    DIMMessageType_Forward = 0xFF
};
````

### 2.2. Message

When the user want to send out a message, the client needs TWO steps before sending it:

1. encrypt the **Instant Message** to **Secure Message**;
2. sign the **Secure Message** to **Certified Message**.

Similarly, when the client received a message, it needs TWO steps to extract the content:

1. verify the **Certified Message** to **Secure Message**;
2. decrypt the **Secure Message** to **Instant Message**.

#### Instant Message

````
{
    //-------- head (envelope) --------
    sender   : "moki@4HaXeu62Q41eemWcL1X5m56Y5JwKK2JJUU",
    receiver : "hulk@4Qv359gss3FrZpZ2phxykvofmt9fyXx5gJ",
    time     : 1542984590,
    
    //-------- body (content) ---------
    content  : {
        type : 0x01,       // message type
        sn   : 1682437361, // serial number (ID)
        text : "Hey guy!"
    }
    
}
````

content -> JsON string: ```{"sn":1682437361,"text":"Hey guy!","type":1}```

#### Secure Message

```
/**
 *  Algorithm:
 *      json = json(content);
 *      PW   = random();
 *      data = encrpyt(json, PW);        // Symmetric
 *      key  = encrypt(PW, receiver.PK); // Asymmetric
 */
{
    //-------- head (envelope) --------
    sender   : "moki@4HaXeu62Q41eemWcL1X5m56Y5JwKK2JJUU",
    receiver : "hulk@4Qv359gss3FrZpZ2phxykvofmt9fyXx5gJ",
    time     : 1542984590,
    
    //-------- body (content) ---------
    data     : "f7UNcvxT8uTMujc4CUVHzrlPbz3FriSfxL8xPonvitRZMSOCGpHV3qfpL8vW/J6U",
    key      : "TcujklBJChTQZseEy7Q6UEtB/jZS9hLQdes7oUkoqA02c+qY8WDLAWCAORmkaUijyVgZCR/7MuFkM3qsxLb+A5bpKu+5Wbj141kTxr7PFDfqjGX06Qo9zJfYHFLYHSHMpcnCAX68WRU7kYYehNe7jCM7gco2K3TZrWw0Ot75API="
}
```

#### Certified Message

```
/**
 *  Algorithm:
 *      ...
 *      digest    = sha256(sha256(data));
 *      signature = sign(digest, sender.SK);
 */
{
    //-------- head (envelope) --------
    sender   : "moki@4HaXeu62Q41eemWcL1X5m56Y5JwKK2JJUU",
    receiver : "hulk@4Qv359gss3FrZpZ2phxykvofmt9fyXx5gJ",
    time     : 1542984590,
    
    //-------- body (content) ---------
    data      : "f7UNcvxT8uTMujc4CUVHzrlPbz3FriSfxL8xPonvitRZMSOCGpHV3qfpL8vW/J6U",
    key       : "TcujklBJChTQZseEy7Q6UEtB/jZS9hLQdes7oUkoqA02c+qY8WDLAWCAORmkaUijyVgZCR/7MuFkM3qsxLb+A5bpKu+5Wbj141kTxr7PFDfqjGX06Qo9zJfYHFLYHSHMpcnCAX68WRU7kYYehNe7jCM7gco2K3TZrWw0Ot75API=",
    signature : "K+YkHm6XRHmUz1X1XlwhrybkrBCkeo9nBPg3fzFJISxTtepUjaAQuOVpjEkte69dCKIMF+rQZKq1Gi3BBqXAGmvoUCnuFm9zy1f4T3PpCoOvoASca5fbYSSXSls0XV/BHtZJo+0SkMkzrFpWR9941y0XgnvfTYvwUeYqYrcYCw4="
}
```


---
Written by [Albert Moky](http://moky.github.com/) Sun Nov 11 23:18:08 CST 2018
