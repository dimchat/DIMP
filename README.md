# Decentralized Instant Messaging Protocol (DIMP)

[![license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/moky/DIMP/blob/master/LICENSE)
[![Version](https://img.shields.io/badge/alpha-0.1.0-red.svg)](https://github.com/moky/DIMP/wiki)

## 0. Abstract
This document introduces a new protocol designed for instant messaging (IM) and an architecture for developing decentralized IM applications. The software provides accounts(user identity recognition) and communications between accounts safely by end-to-end encryption.

It includes just TWO extremely simple parts:

1. User Identify
2. Messaging

Copyright &copy; 2018 Albert Moky

### White papers

- [English](TechnicalWhitePaper.md)
- [Chinese](zh-CN/TechnicalWhitePaper.md)

## 1. User Identify

### 1.0. Meta

See [mkm.Meta](https://github.com/dimchat/mkm-objc#meta) for details.

### 1.1. ID

See [mkm.ID](https://github.com/dimchat/mkm-objc#id) for details.

### 1.2. Public Key

A **public key** (PK) was binded to an ID by the [Meta Algorithm](https://github.com/dimchat/mkm-objc#id-address).

### 1.3. Entity (Account/Group)
**Entity** is the sender/receiver in the network communication.

An entity can be an account or a group. It has an **ID**, a **name**, and a **number** for searching.

An **account** will have a **public key**.

A **group** will have **founder**, **owner** and **members**.

```javascript
// create account
user = new Account(accountID, accountPK);

// create group
group = new Group(groupID, founderID);
```

## 2. Messaging

### 2.0. Envelope

See [dkd.Envelope](https://github.com/dimchat/dkd-objc#envelope) for details.

### 2.1. Content

See [dkd.Content](https://github.com/dimchat/dkd-objc#content) for details.

### 2.2. Instant Message

See [dkd.InstantMessage](https://github.com/dimchat/dkd-objc#instant-message) for details.

### 2.3. Reliable Message

See [dkd.ReliableMessage](https://github.com/dimchat/dkd-objc#reliable-message) for details.

---
Version 0.1 by [Albert Moky](http://moky.github.com/) [Sun Nov 11 23:18:08 CST 2018]
