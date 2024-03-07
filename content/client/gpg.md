---
title: GnuPG
author: Denshi
description: Encrypt, decrypt, sign and verify messages and files.
icon: gpg.svg

draft: true

date: 2023-01-01
email: alex@denshi.org
monero: 48dnPpGgo8WernVJp5VhvhaX3u9e46NujdYA44u8zuMdETNC5jXiA9S7JoYMM6qRt1ZcKpt1J3RZ3JPuMyXetmbHH7Mnc9C
---

[GnuPG](https://gnupg.org/) is a free and libre implementation of the PGP (Pretty Good Privacy) encryption protocol. PGP, which has existed since 1997, is the de-facto standard for **asymmetric encryption** online and offline.

> Asymmetric encryption refers to a model of encryption where every individual holds **two mathematically-linked keys;** the ***private/secret*** key, and the ***public*** key.

Put simply, a **public key** allows you to encrypt information in such a way that only the person who possesses the **secret key** will be able to decrypt it; this is why it's safe to distribute your public key to everyone. If a user wanted to send an encrypted message to another user, all it takes is the **other user's public key** and some **PGP software** to encrypt that message so that **only the recipient will be able to read it.**

> If you want to know more about how asymmetric encryption actually works, [you can learn more here.](https://users.ece.cmu.edu/~adrian/630-f04/PGP-intro.html#p10)

## Generating Keys

While there are plenty of graphical clients available for PGP, the terminal client "GnuPG" will be demonstrated here. It's the most widely available on nearly every UNIX-like OS, like Linux and BSD.

Simply run:

```sh
gpg --full-gen-key
```
Now follow the prompts, and enter any relevant identity information you want to use with your key. 
 
## Identities

Every PGP public key supports **identities.** This means you can give a key a name and an associated email in [supported clients.](https://www.openpgp.org/software/) Not only can you send encrypted messages with PGP, but you can also **verify a user's identity** through the use of **fingerprints.**

To edit any of the parameters of your key, run the following command with your key's signature:

```sh
gpg --edit-key {{<hl>}}FCA165AAA90719CE3A9CEF3E3264540BB6A15BAA{{</hl>}}
```

This will make you enter the GnuPG shell. Run the `uid` command to list existing identities:

```txt
[ultimate] (1)  Billy (Billy's key) <billy@example.org>
```

Run the `adduid` command to add identities to the key, and the `deluid` command to delete old ones.

> Please note: Signatures made for older IDs will not be valid for the new oens.

## File Signature and Verification

With your **secret key**, you can "sign" files for public distribution. This cryptographic signature allows anyone with access to your **public key** to verify the integrity of the file, and that it was genuinely signed by the purported owner.

Suppose we have a file, `file.txt`, with the following contents:

```txt
Hello world, this is the message I intend to cryptographically sign. Isn't it neat?
```

There are 3 ways to sign a file:

 - "Normal" signing -- Compressing the file and merging it with its signature
 - Clearsigning -- **Not** compressing the file but still merging it with a signature
 - Detached signing -- Creating a separate file with the signature

### "Normal" Signing

"Normal" signing is done as follows:

```sh
gpg --sign {{<hl>}}file.txt{{</hl>}}
```

This would produce `file.txt.gpg`, a [compressed binary file.](file.txt.gpg)

To simply verify the integrity of the file **without uncompressing it,** run:

```sh
gpg --verify {{<hl>}}file.txt.gpg{{</hl>}}
```

To actually access the file contents while verifying the signature, run:
```sh
gpg --decrypt {{<hl>}}file.txt.gpg{{</hl>}}
```

### Clearsigning

Clearsinging uses the `--clearsign` option:

```sh
gpg --clearsign {{<hl>}}file.txt{{</hl>}}
```

This would produce `file.txt.asc` with the following plaintext file and signature:
```txt
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA256

Hello world, this is the message I intend to cryptographically sign. Isn't it neat?
-----BEGIN PGP SIGNATURE-----

iQIzBAEBCAAdFiEEPLo4gPw9IH8OGjBBcnM3RQFBRBYFAmXp3bEACgkQcnM3RQFB
RBb0Dw/+PudLdSdnvQexGnLDJt2HOqsg4Jg1+tCwN1+zeKRF0eH0cVSXkXV5Uf8c
s+PEY423BZf212gjgv/d8H3f20CuDUSDYsMr0m8xuqrczTvVXAGVkJ/Q9ljWtI0v
o3qEQYFAFBRUSOk7Q1RnHAudbNZ3+sGsSW7kmMJsvWTnvMgOG9sDhP63YuFZShXI
uPTmNtTM+mfjB3r13srzeMJG68hDsQsvS9gxjy8xNoffL/YkMHLOZ/LPOXPglyuo
h4elHjORMcleRYl8zsaiRZIFyhzvUZwYJBgSPIdL9F5xVo1HgpCUz9KGE+5dCro2
eeioyProQZ9uvfhotLB7W7AmamaxFiorF1loxNwg7F62wvjEvEohTjYYrBw9ZTYC
W8C6PM7HbMfwIVHYjc9v5KZNALBqnZrTkWeF7ix10oniASY+QdBJuM7YHqie8d2P
ffdFtU06XubWlEZKKMwu9vIdDntlmXIkmPRdWTpYQEIpxh4MMe1nTGHXZZ0+O8Cn
E3ABKVTCFeiAdUWovGPWx5pZBkiVg68vg+g11EKv8ryDyAfzk3rAMGfipnGwzDkQ
jxdwfnnRyt7BSgN1IVYvydo8a+cJBcT/xhgJMnKxAxMzuaIUFEjmV8sepZnB8jrI
X+i4uUAn5UWQAojya2PU+KqF8O520+VykaLQZPbzUJBME04Ghy0=
=Aomm
-----END PGP SIGNATURE-----
```

> The bottom part with all the seemingly random characters is the actual signature of the file. Copying only the bottom part allows you to "detach" the plaintext signature for use on webpages or on paper.

To verify a plaintext clearsigned signature, run:

```sh
gpg --verify {{<hl>}}file.txt.asc{{</hl>}}
# Include the message in the output
gpg --decrypt {{<hl>}}file.txt.asc{{</hl>}}
```

### Detached Signing

Detached signatures can be created as follows:

```sh
gpg --detach-sig {{<hl>}}file.txt{{</hl>}}
```

This would produce [`file.txt.sig`](file.txt.sig), which can be distributed alongside the original `file.txt` for anyone to optionally verify the signature.

To verify a file with its detached signature, run the following command:

```sh
gpg --verify file.txt.sig file.txt
```

## Communication
PGP is a supported encryption standard for Emails and XMPP messaging. You can use it to maintain a single, consistent private key for all your communication, which is linked to your email address identity.

To encrypt a **plaintext** message for some specific **recipient** using their **public key,** use the following command:

```sh
echo {{<hl>}}"Hello, world"{{</hl>}} | gpg --encrypt --armor -r {{<hl>}}alex@denshi.org{{</hl>}}
```

This will output the following:
```txt
-----BEGIN PGP MESSAGE-----

hQIMA7mF0M3CHdaFAQ//eh6a92pVpktH4bi2D1XXujTJpJvN7xtxfDkOpWnspD0I
ekWWsxu89abH20fWKQI9hT44YMQaom6vn7ZlSpK2xse3intPDVwMQC/cmJa9Nfpc
8ol7h8Z7x1HodoNcneDJRJpMTfRUhy3gShbN55jI3H7MryEQyt14RdYfnteqSxfD
vgyNHy3tDdeu25b12njHflupp+pfd15VfmVZ+mCY8/xcWUz4gUdiljbiuEapdwow
Rrs3RH30jThew8FG7v0i9rm92+ijx2mprSIGDKWUCFslkvV7IqLjd1jvb8coo11j
v/H3Lj67tVHX0UHlRrBy3LjMVV37q1gNmUhzMrPzpvIzKxqxP2NVut5aHxDmjdTw
Jw6VfBBpCrV8cHgTgfs9SUnujnrbZu5AUha4sTvxJ2/SDWeBOBOES3Ej7xTVDtPO
PTbZItEXBu5c+ZEsXaVAsF76j0iAxmgn3Y+Sp6D1nmUwRWk6YL5ex5+qd2vHfKwc
XcmBFbxJggPJjkgI8BAsdPjUqq4LuGZJaZfNWLoHAN+K7610knC8lSMjCpJ8N32/
xJGCUy8g2Z1Sz/MPZqSrdZKRNSqgdAC2xXie02Rskasgd1SfGeNrm5fnuyhaZvrS
LiMm7TAkqBZEDFf4xYIIJKkgGMkH2K0Z9kubJHyQ0pJOzOTzqXZ80MeRphGHglTS
SAEblj0E29J+1n74PN6+RcJmmWQD+fUbJNSASF/fQ9puxjnpCG7WNKkCTFAoQmed
bC4Gw6JFAoxoXVFUp28OpTsT1HIC2DcMaA==
=fZ8m
-----END PGP MESSAGE-----
```

> Note: Use the `--output` option in `gpg` to give specific filenames for your resulting file. In this case, `--output file.txt.encrypted` could've been picked for easier transport.

To encrypt a plaintext **file** and output a plaintext encrypted message, run:

```sh
gpg --encrypt --armor -r {{<hl>}}alex@denshi.org{{</hl>}} {{<hl>}}file.txt{{</hl>}}
```

The above command will output `file.txt.asc` by default, which can then be sent to the recipient for decryption.

To decrypt a plaintext message you receive, simply run:
```sh
gpg --decrypt {{<hl>}}file.txt.asc{{</hl>}}
``` 

This will print out the message to the command line.

## File Encryption

To encrypt any binary file, such as `file.zip`, simply run:

```sh
gpg --encrypt -r {{<hl>}}alex@denshi.org{{</hl>}} {{<hl>}}file.zip{{</hl>}}
```

This will produce `file.zip.gpg` by default, which can be decrypted by running:

```sh
gpg --output {{<hl>}}file.zip{{</hl>}} --decrypt {{<hl>}}file.zip.gpg{{</hl>}}
```

Congratulations! You now know how to use GnuPG!
