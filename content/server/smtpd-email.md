---
## Guide Information
title: "OpenSMTPD + Dovecot email server"
date: 2024-09-18
icon: email.svg
description: A secure email server.
ports: [25, 587, 993]

## Author Information
author: zacoons
---

## Installation

For OpenBSD (`opensmtpd` should be installed by default):
```
# pkg_add opensmtpd-filter-rspamd rspamd dovecot
```

For Arch Linux:
```
# pacman -S opensmtpd opensmtpd-filter-rspamd rspamd dovecot
```

## Prerequisites

Open ports 25 (SMTP), 587 (SMTPS), and 993 (IMAPS). Port 25 will be used for incoming mail, 587 will be used for outgoing mail, and 993 will be used for serving mail to your email clients (e.g. neomutt).

You will also need to generate SSL certificates for your domain. I recommend using `acme-client` shipped with OpenBSD.

## Making yourself credible

### Reverse DNS

An rDNS record allows other email servers to match a sender's IP address with the domain it claims to be.
How this is set up depends on how you're hosting your server. For me, it was a simple phone call to my ISP.
A lot of people will say that you need a VPS to self-host email, but this isn't true if you have an ISP that is willing to add an rDNS record for you.

You can check if you have an rDNS record with

```
# dig +short -x <public ip>
```

This should respond with the domain name of you email server. e.g.

```
# dig +short -x 95.217.236.249
lists.archlinux.org.
```

### DKIM (Domain Keys Identified Mail)

See [the Rspamd manual on DKIM signing](https://rspamd.com/doc/modules/dkim_signing.html)

To get a keypair run the following command. For the selector I recommend putting `dkim`.
```
# rspamadm dkim_keygen -s '<selector>' -d <domain> -t ed25519 -k /etc/mail/dkim/<domain>.key
```

It should output something like this
```
<selector>._domainkey IN TXT ( "v=DKIM1; k=ed25519;"
        "p=ml82zTjl3EGAAwjpHezOeZQHzaDrxi64/jFfA+kdY2E=" ) ;
```

You'll need to copy that to a TXT record on your DNS. The TXT record in the output is weirdly formatted. It should really look like this:
```
v=DKIM1; k=ed25519; p=<key>
```

You can inspect what other people's keys look like by running
```
# dig +short TXT <selector>._domainkey.<domain>
```
e.g.
```
# dig +short TXT dkim._domainkey.zacoons.com
"v=DKIM1; k=ed25519; p=ZD6c3x5YiLDljo0xsP5LAs5IbONeziS+NpcZlOA1800="
```

> WARNING: If you don't want to be on the cutting edge, then use a good ol' RSA key (see Rspamd manual above). Ed25519 keys are not widely supported in software, making it inadvisable to exclusively use them in a production environment, as it may result in rejected emails. If you choose to use Ed25519 keys, it is recommended to pair them with an RSA key, providing a fallback option in case a recipient domain is unable to parse Ed25519 keys or signatures.

### SPF (Sender Policy Framework)

SPF records are designed to prevent forgery. They allow you to specify rules about how emails can be sent from your domain. For my server I have the following policy:

```
v=spf a -all
```

This will check that the sender's IP address matches an A record for zacoons.com. Otherwise it will reject all mail from my domain.

Read about SPF [here](http://www.open-spf.org/SPF_Record_Syntax).
You can see what others do by running
```
# dig +short TXT <domain>
```
e.g.
```
# dig +short TXT lists.archlinux.org
"v=spf1 ip4:95.217.236.249 ip6:2a01:4f9:c010:9eb4::1 ~all"
```

### DMARC (Domain-based Message Authentication, Reporting, and Conformance)

DMARC records tell other email servers how to treat emails from your domain if they **fail** validations such as DKIM and SPF. For my server I have the following policy:

```
v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@zacoons.com
```

This will cause the other server to quarantine emails from a domain which fails validation, and it will send a report to dmarc-reports@zacoons.com.

Read about DMARC [here](https://dmarc.org/overview).
You can see what others do by running
```
# dig +short TXT _dmarc.<domain>
```
e.g.
```
# dig +short TXT _dmarc.denshi.org
"v=DMARC1; p=reject; rua=mailto:dmarc@denshi.org; fo=1"
```

## Configuration

### Configuring OpenSMTPD

```
# /etc/mail/smtpd.conf

pki example.com cert "/etc/ssl/example.com.fullchain.pem"
pki example.com key "/etc/ssl/private/example.com.key"

table passwd file:/etc/mail/passwd
table aliases file:/etc/mail/aliases
table virtuals file:/etc/mail/virtuals

filter rspamd proc-exec "filter-rspamd"

# --- listeners ---

listen on lo

# inbound
# you may need to change `bse0` to another interface. run `ifconfig` to see what the relevant network interface is called on your computer
# this requires connecting servers to establish a TLS connection and they must have a valid SSL certificate
# it also applies Rspamd filters
listen on bse0 \
        tls-require verify pki example.com \
        filter rspamd

# outbound
# this listener requires authentication, making it only valid for sending outbound mail
# rspamd must be applied so that it can DKIM sign your mail
listen on bse0 port 587 \
        tls-require pki example.com auth <passwd> \
        filter rspamd

# --- actions and matchers ---

action local_mail maildir alias <aliases>
action remote_mail lmtp "/var/dovecot/lmtp" rcpt-to
action outbound relay helo example.com

# inbound
match from local for local action local_mail
match from any for domain example.com action remote_mail
# outbound
match from local for any action outbound
# requires auth for a remote connection to send mail
match from any auth for any action outbound
```

See `man smtpd.conf` for more information. There is also a section below which provides links to other email server guides.

The aliases file should already exist, but you'll need to create `/etc/mail/passwd` and populate it like so:

```
billy@example.com:$2b$08$ynb94C/pvvrZ./BCjtVPwOcTnOBILHM7ifsCk6Oce/GUqW08kFxR2
billyette@example.com:$2b$08$rERmMLIRNgo.ab/conrvI.VWB5RpOF3YE4lPi00LtxtCWhAEp7uhS
```

This follows the format `<username>:<encrypted password>`.

The encrypted password is generated with
```
# smtpctl encrypt <password>
```
It's not actually encrypted, just salted and hashed. Essentially it jumbles up the password so that it can't be read, only checked against.

### Configuring Dovecot

Before you do anything, make a backup of the original configuration.

```
# mv /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.backup
```

```
# /etc/dovecot/dovecot.conf

protocols = imap lmtp

ssl = required
ssl_cert = </etc/ssl/example.com.fullchain.pem
ssl_key = </etc/ssl/private/example.com.key

mail_location = maildir:~/Maildir

passdb {
        driver = passwd-file
        args = /etc/mail/passwd
}

userdb {
        driver = static
        args = uid=vmail gid=vmail home=/var/vmail/%d/%n
}

service imap-login {
        inet_listener imaps {
                port = 993
                ssl = yes
        }
        # disable imap
        inet_listener imap {
                port = 0
        }
}

# you can add more namespaces such as "sent", "trash", "drafts", etc.
# see 
namespace {
        inbox = yes
        separator = /
}
```

See [Dovecot's official documentation](https://doc.dovecot.org/2.3) for more details. You can also refer to one of the links in the final section of this guide.

## Conclusion

On OpenBSD run:
```
# rcctl enable smtpd rspamd dovecot
# rcctl restart smtpd rspamd dovecot
```

On Arch Linux run:
```
# systemctl enable smtpd rspamd dovecot
# systemctl restart smtpd rspamd dovecot
```

## Troubleshooting

The log file is at `/var/log/maillog`.

### OpenSMTPD

To check configuration file run `smtpd -nv`.

To run with extra verbosity, execute the following commands:

```
# rcctl stop smtpd
# smtpd -dv
```

This will run the daemon directly in the terminal, where you can watch the output. You can also pass `-T <trace>` to smtpd for debugging specific issues. See `man smtpd` for a list of options.

You can use OpenSSL to connect as a client:

```
# openssl s_client -connect example.com:25 -starttls smtp
helo example.com
```

The following demonstrates logging in and sending an email:

```
# echo -ne "\0billy@example.com\0epicpwd" | base64
AGJpbGx5QGV4YW1wbGUuY29tAGVwaWNwd2Q=
# openssl s_client -connect example.com:587 -starttls smtp
helo example.com
auth plain AGJpbGx5QGV4YW1wbGUuY29tAGVwaWNwd2Q=
mail from: <billy@example.com>
rcpt to: <someone@gmail.com>
data
From: billy@example.com
To: someone@gmail.com
Subject: test

Yo
.
QUIT
```

### Dovecot

For extra verbosity in the logs, you can add the following lines to your Dovecot configuration:

```
auth_verbose = yes
auth_verbose_passwords = yes
auth_debug = yes
auth_debug_passwords = yes
mail_debug = yes
verbose_ssl = yes
```

Furthermore, the Dovecot website has an excellent manual on testing and troubleshooting. It can be found [here](https://doc.dovecot.org/2.3/admin_manual/test_installation).

### Other resources

Here are some other guides which you might find useful to cross-reference with this one.

- [c0ffee.net](https://www.c0ffee.net/blog/mail-server-guide)
- [unixdigest.com](https://www.unixdigest.com/tutorials/arch-linux-mail-server-tutorial-part-2-opensmtpd-dovecot-dkimproxy-and-lets-encrypt.html)
- [poolp.org](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd)
- [omarpolo.com](https://www.omarpolo.com/post/opensmtd-dovecot-virtual-users.html)
