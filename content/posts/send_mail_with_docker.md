---
title: "Send mails from your server with docker-mailserver"
date: 2025-01-07T09:34:57+01:00
draft: true
---


I wanted to be able to send some emails for I project I had.

Instead of going for something like SendGrid, I figured it couldn't be *that hard* to deploy a mail server on a VPS.

Here is what you need:

* A domain name, where you can add DNS entries.
* A VPS, here I have an Ubuntu 24.10, but it should carry to any Linux.


## Install Docker and Docker Compose

Let's [__install docker__](https://docs.docker.com/engine/install/ubuntu/) on our __Ubuntu__ server.

Add the docker repository:

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

And then install `docker`, and `docker-compose`:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

You should now have the `docker` and `docker compose` commands.

## Clone the project

We are going to use the [docker-mailserver](https://github.com/docker-mailserver/docker-mailserver) project.
Let's __clone__ the __repository__.

```bash
git clone https://github.com/docker-mailserver/docker-mailserver.git
```

There is quite a lot we can do with this. Here we are just setting it up to send emails.

The __full documentation__ of the project is here: https://docker-mailserver.github.io/docker-mailserver/latest/examples/tutorials/basic-installation/.


The main file we are going to use is the `compose.yaml`. It should look something like this by default:

```yaml
services:
  mailserver:
    image: ghcr.io/docker-mailserver/docker-mailserver:latest
    container_name: mailserver
    # Provide the FQDN of your mail server here (Your DNS MX record should point to this value)
    hostname: mail.example.com
    env_file: mailserver.env
    # More information about the mail-server ports:
    # https://docker-mailserver.github.io/docker-mailserver/latest/config/security/understanding-the-ports/
    ports:
      - "25:25"    # SMTP  (explicit TLS => STARTTLS, Authentication is DISABLED => use port 465/587 instead)
      - "143:143"  # IMAP4 (explicit TLS => STARTTLS)
      - "465:465"  # ESMTP (implicit TLS)
      - "587:587"  # ESMTP (explicit TLS => STARTTLS)
      - "993:993"  # IMAP4 (implicit TLS)
    volumes:
      - ./docker-data/dms/mail-data/:/var/mail/
      - ./docker-data/dms/mail-state/:/var/mail-state/
      - ./docker-data/dms/mail-logs/:/var/log/mail/
      - ./docker-data/dms/config/:/tmp/docker-mailserver/
      - /etc/localtime:/etc/localtime:ro
    restart: always
    stop_grace_period: 1m
    # Uncomment if using `ENABLE_FAIL2BAN=1`:
    # cap_add:
    #   - NET_ADMIN
    healthcheck:
      test: "ss --listening --tcp | grep -P 'LISTEN.+:smtp' || exit 1"
      timeout: 3s
      retries: 0
```

You can see from the _volumes_ that all the data will be stored inside the `./docker-data/` folder.

```yaml
volumes:
    - ./docker-data/dms/mail-data/:/var/mail/
    - ./docker-data/dms/mail-state/:/var/mail-state/
    - ./docker-data/dms/mail-logs/:/var/log/mail/
    - ./docker-data/dms/config/:/tmp/docker-mailserver/
    - /etc/localtime:/etc/localtime:ro
```

The first thing we are going to configure is the `hostname`:

```yaml
# Provide the FQDN of your mail server here (Your DNS MX record should point to this value)
hostname: mail.example.com
```

## DNS entries

In my example, I am going to use the domain `ctf.lasne.pro`.

Modify the `compose.yml` to point to your domain.

```yaml
# Provide the FQDN of your mail server here (Your DNS MX record should point to this value)
hostname: ctf.lasne.pro
```

As indicated in the file, you will also need to add an __MX entry__ for your domain.  

Note that your MX entry [should not point to an IP](https://serverfault.com/a/54807/516452).


I am using OVH for my domain. Adding an a DNS entry look like this:

![Adding a DNS entry on the OVH website](/send_mail_docker/add_DNS_entry_OVH.png)

Confirmation dialog:
![Adding a DNS entry on the OVH website](/send_mail_docker/add_DNS_entry_OVH2.png)


After this, your DNS entries should look like this:

![Adding a DNS entry on the OVH website](/send_mail_docker/OVH_dns_entries.png)

You can test your MX entry with the `host` command:

```bash
$ host -mx ctf.lasne.pro

ctf.lasne.pro has address 164.92.168.121
ctf.lasne.pro mail is handled by 1 ctf.lasne.pro.
```

You also need to add an SPF field to authorize you domain to send emails.

## Add an email address

You can use the `./setup.sh` script to add an email account.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">root@mail-srv:~/docker-mailserver# ./setup.sh 
<font color="#AE81FF">SETUP</font><font color="#F92672">(</font><font color="#F4BF75">1</font><font color="#F92672">)</font>

<font color="#FFAF00">NAME</font>
    setup - &apos;docker-mailserver&apos; Administration &amp; Configuration CLI

<font color="#FFAF00">SYNOPSIS</font>
    setup [ OPTIONS<font color="#F92672">...</font> ] COMMAND [ help <font color="#F92672">|</font> ARGUMENTS<font color="#F92672">...</font> ]

    COMMAND <font color="#F92672">:=</font> { email <font color="#F92672">|</font> alias <font color="#F92672">|</font> quota <font color="#F92672">|</font> dovecot-master <font color="#F92672">|</font> config <font color="#F92672">|</font> relay <font color="#F92672">|</font> debug } SUBCOMMAND

<font color="#FFAF00">DESCRIPTION</font>
    This is the main administration command that you use for all your interactions with
    &apos;docker-mailserver&apos;. Initial setup, configuration, and much more is done with this CLI tool.

    Most subcommands can provide additional information and examples by appending &apos;help&apos;.
    For example: &apos;setup email add help&apos;

<font color="#F92672">[</font><font color="#FFAF00">SUB</font><font color="#F92672">]</font><font color="#FFAF00">COMMANDS</font>
    <font color="#66D9EF"><b>COMMAND</b></font> email <font color="#F92672">:=</font>
        setup email <font color="#A1EFE4">add</font> &lt;EMAIL ADDRESS&gt; [&lt;PASSWORD&gt;]
        setup email <font color="#A1EFE4">update</font> &lt;EMAIL ADDRESS&gt; [&lt;PASSWORD&gt;]
        setup email <font color="#A1EFE4">del</font> [ OPTIONS<font color="#F92672">...</font> ] &lt;EMAIL ADDRESS&gt; [ &lt;EMAIL ADDRESS&gt;<font color="#F92672">...</font> ]
        setup email <font color="#A1EFE4">restrict</font> &lt;add<font color="#F92672">|</font>del<font color="#F92672">|</font>list&gt; &lt;send<font color="#F92672">|</font>receive&gt; [&lt;EMAIL ADDRESS&gt;]
        setup email <font color="#A1EFE4">list</font>
[...]
</code></pre></div>
{{< /rawhtml >}}

We create an email address with this command:

```bash
./setup.sh email add no-replay@your.domain.com Str0ngP4ssw0rd
./setup.sh email add tom@ctf.lasne.pro littletommy
```



## Start the Server, and send an email

We can now start the mail server with `docker compose up -d`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">root@ubuntu-s-1vcpu-2gb-fra1-01:~/docker-mailserver# docker compose up -d
<font color="#66D9EF">[+] Running 2/2</font>
 <font color="#A6E22E">✔</font> Network docker-mailserver_default  <font color="#A6E22E">Created</font>    <font color="#66D9EF">0.2s </font>
 <font color="#A6E22E">✔</font> Container mailserver               <font color="#A6E22E">Started</font>    <font color="#66D9EF">0.7s </font>
</code></pre></div>
{{< /rawhtml >}}

To test this, let's send an email with `swaks`, the SMTP swiss-army knife:

```bash
sudo apt install swaks
```

__Send an email__ with swaks:

```bash
swaks -t lasneolivier@gmail.com -s localhost:587 -au tom@ctf.lasne.pro -ap littletommy -f tom@ctf.lasne.pro --body "Hi mom!" --h-Subject "This is a test message"
```

If everything works correctly. The following mail should appear in your spams.

![Mail in your spams](/send_mail_docker/spam.png)


## Configure TLS for the outgoing messages

You can see the documentation on how to do this here: https://docker-mailserver.github.io/docker-mailserver/edge/config/security/ssl/#lets-encrypt-recommended.

### Get a certificate from Let's Encrypt

The certificate of Let's Encrypt are stored in `/etc/letsencrypt/live/` by domain name.
In our example it will be in `/etc/letsencrypt/live/ctf.lasne.pro/`.

You need to give access to this folder to the docker container.

```yaml
services:
  mailserver:
    hostname: ctf.lasne.pro
    environment:
      - SSL_TYPE=letsencrypt
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt
```

You also need to edit the `mailserver.env` file.  
To specify that the certificate is done with Let's Encrypt:

```yaml
# Please read [the SSL page in the documentation](https://docker-mailserver.github.io/docker-mailserver/latest/config/security/ssl) for more information.
#
# empty => SSL disabled
# letsencrypt => Enables Let's Encrypt certificates
# custom => Enables custom certificates
# manual => Let's you manually specify locations of your SSL certificates for non-standard cases
# self-signed => Enables self-signed certificates
SSL_TYPE=letsencrypt
```

Restart the container with

```bash
docker compose up -d
```


We send an email again, but this time we add the `-tls` option.

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~/docker-mailserver# swaks -t lasneolivier@gmail.com -tls -s localhost:587 -au tom@ctf.lasne.pro -ap littletommy -f tom@ctf.lasne.pro --body "Hi mom!" --h-Subject "This is a test message"
=== Trying localhost:587...
=== Connected to localhost.
<-  220 ctf.lasne.pro ESMTP
 -> EHLO ubuntu-s-1vcpu-2gb-fra1-01
<-  250-ctf.lasne.pro
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250 CHUNKING
 -> STARTTLS
<-  220 2.0.0 Ready to start TLS
=== TLS started with cipher TLSv1.3:TLS_AES_256_GCM_SHA384:256
=== TLS client certificate not requested and not sent
=== TLS no client certificate set
=== TLS peer[0]   subject=[/CN=ctf.lasne.pro]
===               commonName=[ctf.lasne.pro], subjectAltName=[DNS:ctf.lasne.pro] notAfter=[2025-04-06T13:15:36Z]
=== TLS peer[1]   subject=[/C=US/O=Let's Encrypt/CN=E6]
===               commonName=[E6], subjectAltName=[] notAfter=[2027-03-12T23:59:59Z]
=== TLS peer certificate passed CA verification, failed host verification (using host localhost to verify)
 ~> EHLO ubuntu-s-1vcpu-2gb-fra1-01
<~  250-ctf.lasne.pro
<~  250-PIPELINING
<~  250-SIZE 10240000
<~  250-ETRN
<~  250-AUTH PLAIN LOGIN
<~  250-AUTH=PLAIN LOGIN
<~  250-ENHANCEDSTATUSCODES
<~  250-8BITMIME
<~  250-DSN
<~  250 CHUNKING
 ~> AUTH LOGIN
<~  334 VXNlcm5hbWU6
 ~> dG9tQGN0Zi5sYXNuZS5wcm8=
<~  334 UGFzc3dvcmQ6
 ~> bGl0dGxldG9tbXk=
<~  235 2.7.0 Authentication successful
 ~> MAIL FROM:<tom@ctf.lasne.pro>
<~  250 2.1.0 Ok
 ~> RCPT TO:<lasneolivier@gmail.com>
<~  250 2.1.5 Ok
 ~> DATA
<~  354 End data with <CR><LF>.<CR><LF>
 ~> Date: Tue, 07 Jan 2025 13:31:09 +0000
 ~> To: lasneolivier@gmail.com
 ~> From: tom@ctf.lasne.pro
 ~> Subject: This is a test message
 ~> Message-Id: <20250107133109.110869@ubuntu-s-1vcpu-2gb-fra1-01>
 ~> X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
 ~> 
 ~> Hi mom!
 ~> 
 ~> 
 ~> .
<~  250 2.0.0 Ok: queued as 39B25F5C4D
 ~> QUIT
<~  221 2.0.0 Bye
=== Connection closed with remote host.
root@ubuntu-s-1vcpu-2gb-fra1-01:~/docker-mailserver# 
root@ubuntu-s-1vcpu-2gb-fra1-01:~/docker-mailserver# swaks -t lasneolivier@gmail.com -tls -s localhost:587 -au tom@ctf.lasne.pro -ap littletommy -f tom@ctf.lasne.pro --body "Hi mom!" --h-Subject "This is a test message"
=== Trying localhost:587...
=== Connected to localhost.
<-  220 ctf.lasne.pro ESMTP
 -> EHLO ubuntu-s-1vcpu-2gb-fra1-01
<-  250-ctf.lasne.pro
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250 CHUNKING
 -> STARTTLS
<-  220 2.0.0 Ready to start TLS
=== TLS started with cipher TLSv1.3:TLS_AES_256_GCM_SHA384:256
=== TLS client certificate not requested and not sent
=== TLS no client certificate set
=== TLS peer[0]   subject=[/CN=ctf.lasne.pro]
===               commonName=[ctf.lasne.pro], subjectAltName=[DNS:ctf.lasne.pro] notAfter=[2025-04-06T13:15:36Z]
=== TLS peer[1]   subject=[/C=US/O=Let's Encrypt/CN=E6]
===               commonName=[E6], subjectAltName=[] notAfter=[2027-03-12T23:59:59Z]
=== TLS peer certificate passed CA verification, failed host verification (using host localhost to verify)
 ~> EHLO ubuntu-s-1vcpu-2gb-fra1-01
<~  250-ctf.lasne.pro
<~  250-PIPELINING
<~  250-SIZE 10240000
<~  250-ETRN
<~  250-AUTH PLAIN LOGIN
<~  250-AUTH=PLAIN LOGIN
<~  250-ENHANCEDSTATUSCODES
<~  250-8BITMIME
<~  250-DSN
<~  250 CHUNKING
 ~> AUTH LOGIN
<~  334 VXNlcm5hbWU6
 ~> dG9tQGN0Zi5sYXNuZS5wcm8=
<~  334 UGFzc3dvcmQ6
 ~> bGl0dGxldG9tbXk=
<~  235 2.7.0 Authentication successful
 ~> MAIL FROM:<tom@ctf.lasne.pro>
<~  250 2.1.0 Ok
 ~> RCPT TO:<lasneolivier@gmail.com>
<~  250 2.1.5 Ok
 ~> DATA
<~  354 End data with <CR><LF>.<CR><LF>
 ~> Date: Tue, 07 Jan 2025 13:31:48 +0000
 ~> To: lasneolivier@gmail.com
 ~> From: tom@ctf.lasne.pro
 ~> Subject: This is a test message
 ~> Message-Id: <20250107133148.110965@ubuntu-s-1vcpu-2gb-fra1-01>
 ~> X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
 ~> 
 ~> Hi mom!
 ~> 
 ~> 
 ~> .
<~  250 2.0.0 Ok: queued as AE7BEF5C4D
 ~> QUIT
<~  221 2.0.0 Bye
=== Connection closed with remote host.
```

We recieve the mail, but this time it was encrypted with TLS.

![Mail encrypted with TLS](/static/send_mail_docker/spam2.png)




## DKIM

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"># docker exec -it mailserver setup config dkim
2025-01-07 13:45:05+00:00 <font color="#66D9EF">INFO </font> open-dkim: Creating DKIM private key &apos;/tmp/docker-mailserver/opendkim/keys/ctf.lasne.pro/mail.private&apos;
</code></pre></div>
{{< /rawhtml >}}

Your DKIM informations will be stored in `docker-data/dms/config/opendkim/keys/your-domain/mail.txt`

