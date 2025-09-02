---
## Guide Information
title: "OpenSMTPD + Dovecot email server"
date: 2024-09-18
icon: email.svg
description: A secure email server.
ports: [25, 465, 993]

## Author Information
author: zacoons
---

## Installation

For OpenBSD (`opensmtpd` should be installed by default):
```sh
pkg_add opensmtpd-filter-rspamd rspamd dovecot
```

For Arch Linux:
```sh
pacman -S opensmtpd opensmtpd-filter-rspamd rspamd dovecot
```

For Debian Linux:
```sh
apt install opensmtpd opensmtpd-filter-rspamd rspamd dovecot-imapd dovecot-lmtpd
```

## Prerequisites

Open ports 25 (SMTP), 465 (submission) and 993 (IMAPS). Port 25 will be used by other email servers to deliver mail to your server, 465 will be used by your server for submitting mail to other servers, and 993 will be used for serving mail to your email clients (e.g. Thunderbird, Betterbird, NeoMutt).

You will also need to generate SSL certificates for your domain. I recommend using `acme-client` shipped with OpenBSD. `certbot` is another good option.

## Making yourself credible

### Reverse DNS

An rDNS record allows other email servers to match a sender's IP address with the domain it claims to be.
How this is set up depends on how you're hosting your server.
A VPS might allow you to setup rDNS records using the dashboard. If you're hosting from home, you could try giving your ISP a phone call (this is what I did).
A lot of people will say that you need a VPS to self-host email, but this isn't true if you have an ISP that is willing to add an rDNS record for you.

You can check if you have an rDNS record with:
```sh
dig +short -x <public ip>
```

This should respond with the domain name of you email server, e.g.

```sh
dig +short -x 107.189.31.10
# denshi.org.
```

### DKIM (Domain Keys Identified Mail)

See [the Rspamd manual on DKIM signing](https://rspamd.com/doc/modules/dkim_signing.html)

To get a keypair run the following command. For the selector I recommend putting `main`.
```sh
rspamadm dkim_keygen -s '<selector>' -d <domain> -k /etc/mail/dkim/<domain>.key
```

It should output something like this:
```
<selector>._domainkey IN TXT ( "v=DKIM1; k=rsa; "
  "p=MIGJAoGBALBrq9K6yxAXHwircsTnDTsd2Kg426z02AnoKTvyYNqwYT5Dxa02lyOiAXloXVIJsyfuGOOoSx543D7DGWw0plgElHXKStXy1TZ7fJfbEtuc5RASIKqOAT1iHGfGB1SZzjt3a3vJBhoStjvLulw4h8NC2jep96/QGuK8G/3b/SJNAgMBAAE=" ) ;
```

You'll need to copy that to a TXT record on your DNS. The TXT record in the output is weirdly formatted. It should really look like this:
```
v=DKIM1; k=rsa; p=<key>
```

You can inspect what other people's keys look like by running
```sh
dig +short TXT <selector>._domainkey.<domain>
```
e.g.
```sh
dig +short TXT dkim._domainkey.zacoons.com
# "v=DKIM1; k=rsa; p=gobbledegook"
```

If you want to be on the cutting edge, then use an Ed25519 key (see Rspamd manual above). Ed25519 keys are not widely supported in software, making it inadvisable to exclusively use them in a production environment, as it may result in rejected emails.

### SPF (Sender Policy Framework)

SPF records are designed to prevent forgery. They allow you to specify rules about how emails can be sent from your domain. For my server I have the following policy:

```
v=spf a -all
```

This will check that the sender's IP address matches an A record for zacoons.com. Otherwise it will reject all mail from my domain.

Read about SPF [here](http://www.open-spf.org/SPF_Record_Syntax).

You can see what others do by running
```sh
dig +short TXT <domain>
```
e.g.
```sh
dig +short TXT denshi.org
# "v=spf1 mx a:mail.denshi.org -all"
```

### DMARC (Domain-based Message Authentication, Reporting, and Conformance)

DMARC records tell other email servers how to treat emails from your domain if they **fail** validations such as DKIM and SPF. For my server I have the following policy:
```
v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@zacoons.com
```

This will cause the other server to quarantine emails from a domain which fails validation and send a report to the specified email address.

Read about DMARC [here](https://dmarc.org/overview).

You can see what others do by running
```sh
dig +short TXT _dmarc.<domain>
```
e.g.
```sh
dig +short TXT _dmarc.denshi.org
# "v=DMARC1; p=reject; rua=mailto:dmarc@denshi.org; fo=1"
```

## Configuration

### OpenSMTPD

```conf
# /etc/mail/smtpd.conf

pki "pki1" cert "/etc/ssl/example.com.fullchain.pem"
pki "pki1" key "/etc/ssl/private/example.com.key"

table passwd file:/etc/mail/passwd
table virtuals file:/etc/mail/virtuals
table aliases file:/etc/mail/aliases

filter rspamd proc-exec "filter-rspamd"

# --- Listeners ---

# Localhost, for local mail that doesn't leave the machine.
listen on lo

# Relay (from a server)
# You may need to change `bse0` to another interface. Run `ifconfig` to see what the relevant network interface is named on your computer.
# The following rule enables TLS support with the "pki1" keypair and applies Rspamd filters.
listen on bse0 port 25 \
        tls pki "pki1" \
        filter "rspamd"

# Submission (from your client)
# This listener requires authentication, making it only valid for sending outbound mail.
# Rspamd must be applied so that it can DKIM sign your mail.
listen on bse0 port 587 \
        tls pki "pki1" auth <passwd> \
        filter "rspamd"

# --- Actions and matchers ---

# Deliver mail to a machine-local maildir folder
action "deliver_local" maildir alias <aliases>
# Deliver mail to Dovecot (using Local Mail Transfer Protocol)
action "deliver_remote" lmtp "/var/dovecot/lmtp" rcpt-to virtual <virtuals>

# Relay mail to another SMTP server
action "relay" relay helo example.com

# --- Matchers ---

# Deliver mail from another server
match from local for local action "deliver_local"
match from any for domain example.com action "deliver_remote"

# Send/relay mail from local or authenticated clients to another server
match from local for any action "relay"
match from auth for any action "relay"
```

See `man smtpd.conf` or [the website](https://man.openbsd.org/smtpd.conf) for more information. There is also a section below which provides links to other email server guides.

The aliases file should already exist, but you'll need to create `/etc/mail/passwd` and populate it like so:

```
billy@example.com:$2b$08$ynb94C/pvvrZ./BCjtVPwOcTnOBILHM7ifsCk6Oce/GUqW08kFxR2
billyette@example.com:$2b$08$rERmMLIRNgo.ab/conrvI.VWB5RpOF3YE4lPi00LtxtCWhAEp7uhS
```

This follows the format `<username>:<encrypted password>`.

The encrypted password is generated with
```sh
smtpctl encrypt <password>
```
It's not actually encrypted, just salted and hashed. Essentially it jumbles up the password so that it can't be read, only checked against.

### Rspamd

```conf
# /etc/rspamd/local.d/dkim_signing.conf

domain {
    example.com {
        path = "/etc/mail/dkim/example.com.key";
        selector = "main";
    }
}
```

### Dovecot

```conf
# /etc/dovecot/dovecot.conf

dovecot_config_version = 2.4.0
dovecot_storage_version = 2.4.0

protocols = lmtp imap

ssl = required
ssl_server_cert_file = /etc/letsencrypt/live/example.com/fullchain.pem
ssl_server_key_file = /etc/letsencrypt/live/example.com/privkey.pem

mail_driver = maildir
mail_path = ~/Maildir
mail_inbox_path = ~/Maildir

passdb passwd-file {
	default_password_scheme = blf-crypt
	passwd_file_path = /etc/mail/passwd
}
userdb static {
	passwd_file_path = /etc/mail/passwd
	fields {
		uid = vmail
		gid = vmail
		home = /var/vmail/%{user | domain}/%{user | username}
	}
}

service imap-login {
	inet_listener imaps {
		port = 993
		ssl = yes
	}
	# Disable IMAP (we only want IMAPS)
	inet_listener imap {
		port = 0
	}
}
namespace inbox {
        inbox = yes
        mailbox "Sent" {
                auto = subscribe
                special_use = \Sent
        }
        mailbox "Drafts" {
                auto = subscribe
                special_use = \Drafts
        }
        mailbox "Trash" {
                auto = subscribe
                special_use = \Trash
        }
}
```

See [Dovecot's official documentation](https://doc.dovecot.org/2.3) for more details. You can also refer to one of the links in the final section of this guide.

You'll need to make a user called vmail:
```sh
useradd -d /var/vmail
```

## Conclusion

On OpenBSD run:
```sh
rcctl enable smtpd rspamd dovecot
rcctl restart smtpd rspamd dovecot
```

On Arch or Debian Linux run:
```sh
systemctl enable smtpd rspamd dovecot
systemctl restart smtpd rspamd dovecot
```

## Troubleshooting

The log file is at `/var/log/maillog`.

### OpenSMTPD

To check configuration file run `smtpd -nv`.

To run with extra verbosity, stop the `smtpd` service and run it manually with `smtpd -dv`.
This will run the daemon directly in the terminal, where you can watch the output.
You can also pass `-T <trace>` for debugging specific issues.

You can use OpenSSL to connect as a client:

```sh
openssl s_client -connect example.com:25 -starttls smtp
# helo example.com
```

The following demonstrates logging in and sending an email:

```sh
echo -ne "\0billy@example.com\0epicpwd" | base64
# AGJpbGx5QGV4YW1wbGUuY29tAGVwaWNwd2Q=
openssl s_client -connect example.com:587 -starttls smtp
# helo example.com
# auth plain AGJpbGx5QGV4YW1wbGUuY29tAGVwaWNwd2Q=
# mail from: <billy@example.com>
# rcpt to: <someone@gmail.com>
# data
# From: billy@example.com
# To: someone@gmail.com
# Subject: test
#
# Hello there, this is some test content.
# .
# QUIT
```

### Dovecot

For extra verbosity in the logs, you can add the following lines to your Dovecot configuration:

```conf
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
- [prefetch.eu](https://prefetch.eu/blog/2020/email-server)
- [prefetch.eu (extras)](https://prefetch.eu/blog/2020/email-server-extras)
