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
# pkg_add opensmtpd-filter-dkim opensmtpd-filter-rspamd rspamd dovecot
```

For Arch Linux:
```
# pacman -S opensmtpd opensmtpd-filter-dkim opensmtpd-filter-rspamd rspamd dovecot
```

## Prerequisites

Open ports 25 (SMTP), 587 (SMTPS), and 993 (IMAPS). Port 25 will be used for delivery, 587 will be used for sending, and 993 will be used for serving mail with Dovecot.

You will also need to generate SSL certificates for your domain. I recommend using `acme-client` shipped with OpenBSD.

## Making yourself credible

### Reverse DNS

An rDNS record allows other email servers to make sure your IP address matches the domain it claims to be. How this is set up depends on how you are hosting your server. A lot of people will say that you need a VPS to self-host an email server but this isn't necessarily true. If you have an ISP that is willing to add an rDNS record for you, then you can host from home. Otherwise you need a VPS.

You can check if you have an rDNS record like so:

```
dig +short -x <public ip>
```

This should respond with the domain name of you email server.

### DKIM (Domain Keys Identified Mail)

If you are on OpenBSD, read `/usr/local/share/doc/pkg-readmes/opensmtpd-filter-dkimsign`. This file was put there when you installed `opensmtpd-filter-dkimsign`. You may also find other package readmes in that directory which can be useful.

Run the following commands:

```
# doas -u _dkimsign openssl genpkey -algorithm ed25519 -outform PEM -out /etc/mail/dkim/private.ed25519.key
# printf "v=DKIM1;k=ed25519;p=%s\n" "$(doas -u _dkimsign openssl pkey -outform DER -pubout -in /etc/mail/dkim/private.ed25519.key | tail -c +13 | openssl base64)"
```

Copy the output of the `printf` command and paste it into a DNS TXT record on your domain: `<selector>._domainkey.example.com`. The selector can by anything. For my email server it is `default`. You can confirm this for yourself by running `dig +short TXT default._domainkey.zacoons.com`.

### SPF (Sender Policy Framework)

SPF records are designed to prevent forgery. They allow you to specify rules about how emails can be sent from your domain. For my server I have the following policy:

```
zacoons.com:   v=spf a -all
```

This will check that the sender's IP address matches an A record for zacoons.com.

Read about SPF [here](http://www.open-spf.org/SPF_Record_Syntax) and check out what others do by running `dig +short TXT <domain>` (e.g. `dig +short TXT gmail.com`)

### DMARC (Domain-based Message Authentication, Reporting, and Conformance)

DMARC records tell other email servers how to treat emails from our domain if they fail validation. For my server I have the following policy:

```
_dmarc.zacoons.com:   v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@zacoons.com
```

This will cause the other server to quarantine emails from our domain which fail our SPF and DKIM tests.

Read about DMARC [here](https://dmarc.org/overview) and check out what others do by running `dig +short TXT _dmarc.<domain>` (e.g. `dig +short TXT _dmarc.gmail.com`).

## Configuration

### Configuring OpenSMTPD

```
# /etc/mail/smtpd.conf

pki example.com cert "/etc/ssl/example.com.fullchain.pem"
pki example.com key "/etc/ssl/private/example.com.key"

table passwd file:/etc/mail/passwd
table aliases file:/etc/mail/aliases

filter dkimsign proc-exec "filter-dkimsign -d example.com -s default \
        -a ed25519-sha256 -k /etc/mail/dkim/private.ed25519.key" user _dkimsign group _dkimsign
filter rspamd proc-exec "filter-rspamd"

listen on lo0
# inbound
listen on bse0 \
        tls-require pki example.com \
        filter rspamd
# outbound
listen on bse0 port 587 \
        tls-require pki example.com auth <passwd> \
        filter dkimsign

action local_mail maildir alias <aliases>
action remote_mail lmtp "/var/dovecot/lmtp" rcpt-to
action outbound relay helo example.com

# inbound
match from local for local action local_mail
match from any for domain example.com action remote_mail
# outbound
match from local for any action outbound
match from any auth for any action outbound
```

The aliases file should already exist, but you'll need to create `/etc/mail/passwd` and populate it like so:

```
billy@example.com:$2b$08$ynb94C/pvvrZ./BCjtVPwOcTnOBILHM7ifsCk6Oce/GUqW08kFxR2
billyette@example.com:$2b$08$rERmMLIRNgo.ab/conrvI.VWB5RpOF3YE4lPi00LtxtCWhAEp7uhS
```

This follows the format `<username>:<encrypted password>`.

The encrypted password is generated with `smtpctl encrypt <password>`.

### Configuring Dovecot

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

namespace {
        inbox = yes
        separator = /
}
```

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

This will run the daemon directly in the terminal, where you can watch the output.

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
