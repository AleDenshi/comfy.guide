---
title: Encryption
author: Denshi
description: An overview of encryption on modern computer systems.
icon: gpg.svg

draft: true

date: 2023-01-01
email: alex@denshi.org
monero: 48dnPpGgo8WernVJp5VhvhaX3u9e46NujdYA44u8zuMdETNC5jXiA9S7JoYMM6qRt1ZcKpt1J3RZ3JPuMyXetmbHH7Mnc9C
---

Computers can utilize **encryption** to allow for **private communication, identity verification, transport-layer security** and **file or disk encryption.** This article is a rundown of all the different ways you can make use of this powerful mathematical tool.

## PGP
PGP has existed since 1997. It is the de-facto standard for **asymmetric encryption** online and offline.

> Asymmetric encryption refers to a model of encryption where every individual holds **two mathematically-linked keys;** the ***private/secret*** key, and the ***public*** key.

Put simply, a **public key** allows you to encrypt information in such a way that only the person who possesses the **secret key** will be able to decrypt it; this is why it's safe to distribute your public key to everyone. If a user wanted to send an encrypted message to another user, all it takes is the **other user's public key** and some **PGP software** to encrypt that message so that **only the recipient will be able to read it.**

> If you want to know more about how asymmetric encryption actually works, [you can learn more here.](https://users.ece.cmu.edu/~adrian/630-f04/PGP-intro.html#p10)

### Generating Keys

While there are plenty of graphical clients available for PGP, the terminal client "GnuPG" will be demonstrated here. It's the most widely available on nearly every UNIX-like OS, like Linux and BSD.

Simply run:

```sh
gpg --full-gen-key
```
Now follow the prompts, and enter any relevant identity information you want to use with your key. 
 
### Identities
Every PGP public key supports **identities.** This means you can give a key a name and an associated email in [supported clients.](https://www.openpgp.org/software/) Not only can you send encrypted messages with PGP, but you can also **verify a user's identity** through the use of **fingerprints.**

### Use in communication
PGP is a supported encryption standard for Emails and XMPP messaging. You can use it to maintain a single, consistent private key for all your communication, which is linked to your email address identity.

(Insert guide on how to encrypt text with PGP here)


### Use in File Encryption

(Insert guide here)

## TLS/SSL

Transport Layer Security is the cornerstone of the most basic online security. Nearly all information you access through web protocols, like HTTPS, will be encrypted on the transport layer. This means the packets of information being sent across the web cannot be inspected for their contents.

## Disk Encryption
