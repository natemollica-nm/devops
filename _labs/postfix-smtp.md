---
title:  Postfix SMTP Server
layout: default
nav_order: 2
---

# **Postfix SMTP Server on AWS EC2**
{: .no_toc }

Configuration steps/guideline for configuring a postfix smtp server.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Ubuntu 22.04 EC2 SMTP Server Configuration

| Domain       | EC2 Instance DNS                                  | EC2 Instance IP |
|--------------|---------------------------------------------------|-----------------|
| mydomain.com | ec2-18-118-32-129.us-east-2.compute.amazonaws.com | <ec2_public_ip> |


### Mail Server DNS

1. Create and register official domain name
2. Set domain root DNS name to EC2 Instance Public IP
3. For SMTP on Port 25 - Ensure you have a means of permitting Port 25
    - In my case, with NO-IP I use Alternate-port SMTP which configures Port 25 forwarding via SMTP relay on Port 3325
4. Configure DNS Record for EC2 Instance with desired hostname: `smtp-mail-server.mydomain.com`
5. Configure SPF Records for domain and mail server (see this [link](https://www.digitalocean.com/community/tutorials/how-to-use-an-spf-record-to-prevent-spoofing-improve-e-mail-reliability) for more details)
    - Google: `v=spf1 a include:_spf.google.com ~all`
    - NO-IP: `v=spf1 include:no-ip.com -all`

### Installing and Configuring Postfix

1. Install postfix: `sudo apt update -y && sudo ap install mailutils`
2. Follow pop-up GUI and select:
    - Internet Site
    - Enter mail server DNS name: `smtp-mail-server.mydomain.com`
3. Establish TLS Encryption via LetsEncrypt:

```shell
$ sudo certbot certonly --standalone --rsa-key-size 4096 --agree-tos --preferred-challenges http -d mydomain.com
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for mydomain.com

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/mydomain.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/mydomain.com/privkey.pem
This certificate expires on 2024-07-01.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
```

### SMTP Test Users

Create the `smtp-test` and `no-reply` for autoreply emails from Springboot application.

Create the below script on your SMTP Server as `smtp-users.sh`

```shell
#!/bin/bash

set -e 

delete_user() {
    local username="${1}"
    local home_dir="${2}"

  echo "deleting ${username} user | homedir: ${home_dir} | mailbox: /var/mail/${username}"
  if getent passwd "${username}" >/dev/null ; then
    sudo /usr/sbin/deluser \
      "${username}" 1>/dev/null
    ! test -d "$home_dir" || {
      sudo rm -rf "$home_dir"
    }
    sudo rm -rf /var/mail/"$username"
  fi
}
create_user() {
    local username="${1}"
    local home_dir="${2}"

  echo "creating ${username} user | homedir: ${home_dir}"
  if ! getent passwd "${username}" >/dev/null ; then
    test -d "$home_dir" || {
      sudo mkdir --parents "$home_dir"
    }
    sudo /usr/sbin/adduser \
      --system \
      --home "${home_dir}" \
      --no-create-home \
      "${username}" 1>/dev/null
    sudo /usr/sbin/groupadd --force --system "${username}"
    sudo /usr/sbin/usermod --gid "${username}" "${username}"
    sudo /usr/sbin/usermod -aG sudo "$username"
    sudo chown -R "$username":"$username" "$home_dir"
    sudo chmod -R 0755 "$home_dir"
  fi
}
delete_user smtp-test /opt/smtp-test
delete_user no-reply /opt/no-reply

create_user smtp-test /opt/smtp-test
create_user no-reply /opt/no-reply
```

Run the script and verify user creation:

```shell
$ ./smtp-users.sh

$ getent passwd smtp-test
smtp-test:x:116:999::/opt/smtp-test:/usr/sbin/nologin

$ getent passwd no-reply
no-reply:x:117:998::/opt/no-reply:/usr/sbin/nologin
```

### Configure System Mail Forwarding

In this step, you’ll set up email forwarding for user root, so that system-generated messages sent to it on your server get
forwarded to an external email address.

**/etc/aliases**

```shell
# Configure forwarding of system level messages to external email
echo 'root: me@gmail.com' | sudo tee -a /etc/aliases
```

```shell
# Update alias changes
sudo newaliases
```

### Configure Postfix for SMTP Authentication and STARTLS

Identify your OpenShift Cluster's IP Settings and VPC CIDRs:
* OpenShift: `10.128.0.0/14`
* SMTP Server VPC CIDR: `172.30.0.0/16`
* SMTP Server Public IP: `18.118.50.119/32`

Apply the below `main.cf` Postfix configuration file with tls cert locations:

{% highlight cfg %}
# See /usr/share/postfix/main.cf.dist for a commented, more complete version
# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname
smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no
# appending .domain is the MUA's job.
append_dot_mydomain = no
# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h
readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 3.6 on
# fresh installs.
compatibility_level = 3.6

# SASL SMTP Authentication
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_use_tls = no

# SMTP TLS Authentication (Client)
smtpd_tls_cert_file = /etc/letsencrypt/live/mydomain.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mydomain.com/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_auth_only = no
smtpd_reject_unlisted_recipient = no

# SMTP TLS Authentication (Server)
smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level = may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtpd_recipient_restrictions = check_recipient_access hash:/etc/postfix/autoreply

smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
myhostname = mydomain.com
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $myhostname, smtp-mail-server.mydomain.com, ip-172-31-26-33.us-east-2.compute.internal, localhost.us-east-2.compute.internal, localhost
relayhost = [smtp-auth.no-ip.com]:3325 
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 10.0.0.0/16 10.128.0.0/14 172.30.0.0/16 18.118.50.119/32

mailbox_size_limit = 0

recipient_delimiter = +
inet_interfaces = all
inet_protocols = all
masquerade_domains = mydomain.com

# Email Forwarding
# virtual_alias_maps = hash:/etc/postfix/virtual
smtpd_milters = inet:localhost:8891
non_smtpd_milters = $smtpd_milters
milter_default_action = accept
milter_protocol = 6
{% endhighlight %}

* **myhostname:** The FQDN (Fully Qualified Domain Name) of your server.
* **mydomain:** Your domain name.
* **inet_interfaces:** Setting this to all allows Postfix to listen on all network interfaces.
* **mydestination:** Specifies the domains that this Postfix instance will deliver emails to locally, rather than forwarding.
* **virtual_alias_domains:** List of domains that Postfix is handling but not listed in mydestination.
* **virtual_alias_maps:** Points to a file that contains mappings of virtual aliases.

Restart: `sudo systemctl restart postfix`

```shell
relayhost = [smtp-auth.no-ip.com]:3325
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_use_tls = yes
```

```shell
echo '[smtp-auth.no-ip.com]:3325 mydomain.com@noip-smtp:<smtp_domain_user_password>' | sudo tee /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
cat /var/log/mail.log
```

**Send Test Email**

```shell
sendmail me@gmail.com
From: no-reply@mydomain.com
Subject: SMTP Server Test Email
I'm testing this from my SMTP server
.
```

### Enabling Wrapper TLS on Postfix (Direct TLS SMTP Connection over Port 465)

```shell
sudo vi /etc/postfix/master.cf
```

Uncomment the below:

```shell
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no

```

Restart postfix: `sudo service postfix restart`

Verify Port 465 Open:

```shell
sudo netstat -ntplu

Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:8891          0.0.0.0:*               LISTEN      1704/opendkim       
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      422/systemd-resolve 
tcp        0      0 0.0.0.0:25              0.0.0.0:*               LISTEN      7167/master         
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      723/sshd: /usr/sbin 
tcp        0      0 0.0.0.0:465             0.0.0.0:*               LISTEN      7167/master         
tcp6       0      0 :::25                   :::*                    LISTEN      7167/master         
tcp6       0      0 :::22                   :::*                    LISTEN      723/sshd: /usr/sbin 
tcp6       0      0 :::465                  :::*                    LISTEN      7167/master         
udp        0      0 127.0.0.53:53           0.0.0.0:*                           422/systemd-resolve 
udp        0      0 172.31.26.33:68         0.0.0.0:*                           420/systemd-network 
udp        0      0 127.0.0.1:323           0.0.0.0:*                           498/chronyd         
udp6       0      0 ::1:323                 :::*                                498/chronyd        
```

## Configure Autoreply for smtp-test user

Reference: https://github.com/innovara/autoreply

Create the following script on your SMTP Server:

```shell
#!/usr/bin/env bash

set -e 

sudo useradd -d /opt/autoreply -s /usr/sbin/nologin autoreply
sudo mkdir /opt/autoreply
sudo chmod 700 /opt/autoreply

sudo su - autoreply -s /bin/bash -c 'wget https://github.com/innovara/autoreply/raw/master/autoreply.py'
sudo su - autoreply -s /bin/bash -c 'chmod 700 autoreply.py'
sudo su - autoreply -s /bin/bash -c 'cat << EOF >autoreply.json
{
  "logging": true,
  "SMTP": "localhost",
  "port": 25,
  "starttls": false,
  "smtpauth": true,
  "username": "natemollica-dev@noip-smtp",
  "password": "<domain_smtp_user_password>",
  "autoreply": [
    {
      "email": "smtp-test@mydomain.com",
      "from": "SMTP Server <no-reply@mydomain.com>",
      "reply-to": "no-reply@mydomain.com",
      "subject": "SMTP Server Test Auto-reply",
      "body": "Youve recently sent a test email to smtp-test@mydomain.com",
      "html": false
    }
  ]
}
EOF'
```

## Configure Postfix for autoreply

Create a Postfix lookup table input file.

```shell
echo 'smtp-test@mydomain.com FILTER autoreply:dummy' > /etc/postfix/autoreply
```

Create its corresponding Postfix lookup table.

```shell
sudo postmap /etc/postfix/autoreply
```
Back up `main.cf`.

```shell
sudo cp /etc/postfix/main.{cf,cf.bak}
```

Update `main.cf` with new lookup table as first item in `smtpd_recipient_restrictions`

```shell
echo 'smtpd_recipient_restrictions = check_recipient_access hash:/etc/postfix/autoreply' | sudo tee -a /etc/postfix/main.cf
```

Back up `master.cf`

```shell
sudo cp /etc/postfix/master.{cf,cf.bak}
```

Add the following autoreply pipe at the end of `master.cf`

```shell
# autoreply pipe
autoreply unix  -       n       n       -       -       pipe
  flags= user=autoreply null_sender=
  argv=/opt/autoreply/autoreply.py ${sender} ${recipient}
```

Restart Postfix

```shell
sudo systemctl restart postfix
```

### Postfix master.cf

{% highlight cfg %}
#
# Postfix master process configuration file.  For details on the format
# of the file, see the master(5) manual page (command: "man 5 master" or
# on-line: http://www.postfix.org/master.5.html).
#
# Do not forget to execute "postfix reload" after editing this file.
#
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (no)    (never) (100)
# ==========================================================================
smtp      inet  n       -       y       -       -       smtpd
#smtp      inet  n       -       y       -       1       postscreen
#smtpd     pass  -       -       y       -       -       smtpd
#dnsblog   unix  -       -       y       -       0       dnsblog
#tlsproxy  unix  -       -       y       -       0       tlsproxy
# Choose one: enable submission for loopback clients only, or for any client.
#127.0.0.1:submission inet n -   y       -       -       smtpd
#submission inet n       -       y       -       -       smtpd
#  -o syslog_name=postfix/submission
#  -o smtpd_tls_security_level=encrypt
#  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_tls_auth_only=yes
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
# Choose one: enable smtps for loopback clients only, or for any client.
#127.0.0.1:smtps inet n  -       y       -       -       smtpd
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
#628       inet  n       -       y       -       -       qmqpd
pickup    unix  n       -       y       60      1       pickup
cleanup   unix  n       -       y       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
#qmgr     unix  n       -       n       300     1       oqmgr
tlsmgr    unix  -       -       y       1000?   1       tlsmgr
rewrite   unix  -       -       y       -       -       trivial-rewrite
bounce    unix  -       -       y       -       0       bounce
defer     unix  -       -       y       -       0       bounce
trace     unix  -       -       y       -       0       bounce
verify    unix  -       -       y       -       1       verify
flush     unix  n       -       y       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       y       -       -       smtp
relay     unix  -       -       y       -       -       smtp
        -o syslog_name=postfix/$service_name
#       -o smtp_helo_timeout=5 -o smtp_connect_timeout=5
showq     unix  n       -       y       -       -       showq
error     unix  -       -       y       -       -       error
retry     unix  -       -       y       -       -       error
discard   unix  -       -       y       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       y       -       -       lmtp
anvil     unix  -       -       y       -       1       anvil
scache    unix  -       -       y       -       1       scache
postlog   unix-dgram n  -       n       -       1       postlogd
#
# ====================================================================
# Interfaces to non-Postfix software. Be sure to examine the manual
# pages of the non-Postfix software to find out what options it wants.
#
# Many of the following services use the Postfix pipe(8) delivery
# agent.  See the pipe(8) man page for information about ${recipient}
# and other message envelope options.
# ====================================================================
#
# maildrop. See the Postfix MAILDROP_README file for details.
# Also specify in main.cf: maildrop_destination_recipient_limit=1
#
maildrop  unix  -       n       n       -       -       pipe
  flags=DRXhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
#
# ====================================================================
#
# Recent Cyrus versions can use the existing "lmtp" master.cf entry.
#
# Specify in cyrus.conf:
#   lmtp    cmd="lmtpd -a" listen="localhost:lmtp" proto=tcp4
#
# Specify in main.cf one or more of the following:
#  mailbox_transport = lmtp:inet:localhost
#  virtual_transport = lmtp:inet:localhost
#
# ====================================================================
#
# Cyrus 2.1.5 (Amos Gouaux)
# Also specify in main.cf: cyrus_destination_recipient_limit=1
#
#cyrus     unix  -       n       n       -       -       pipe
#  flags=DRX user=cyrus argv=/cyrus/bin/deliver -e -r ${sender} -m ${extension} ${user}
#
# ====================================================================
# Old example of delivery via Cyrus.
#
#old-cyrus unix  -       n       n       -       -       pipe
#  flags=R user=cyrus argv=/cyrus/bin/deliver -e -m ${extension} ${user}
#
# ====================================================================
#
# See the Postfix UUCP_README file for configuration details.
#
uucp      unix  -       n       n       -       -       pipe
  flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
#
# Other external delivery methods.
#
ifmail    unix  -       n       n       -       -       pipe
  flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
bsmtp     unix  -       n       n       -       -       pipe
  flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
scalemail-backend unix -       n       n       -       2       pipe
  flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
mailman   unix  -       n       n       -       -       pipe
  flags=FRX user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py ${nexthop} ${user}
# autoreply pipe
autoreply unix  -       n       n       -       -       pipe
  flags= user=autoreply null_sender=
  argv=/opt/autoreply/autoreply.py ${sender} ${recipient}
{% endhighlight %}

### Troubleshooting

Review and trace journal logs for postfix and opendkim services

```shell
$ journalctl --follow --unit postfix.service --unit opendkim.service
```

References:
* [Postfix Docs][A002]
* [Install and Configure Postfix as Send Only SMTP Server (Digitial Ocean)][A001]
* [NO-IP Postfix Configuration Guide (Alternate Port SMTP)][A003]

[A001]: https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-postfix-as-a-send-only-smtp-server-on-ubuntu-22-04
[A002]: https://www.postfix.org/SASL_README.html#client_sasl
[A003]: https://www.noip.com/support/knowledgebase/configure-postfix-work-alternate-port-smtp