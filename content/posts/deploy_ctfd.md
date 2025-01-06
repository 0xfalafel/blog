---
title: "Deploy a CTFd server"
date: 2025-01-06T10:29:08+01:00
draft: true
---

In this blog post, we are going to explore how to deploy a CTFd server with:

* HTTPS
* mail

## Renting the server

You can I use any Linux server you want.
In my case, I am going to use Digital Ocean, with an Ubuntu 24.10 droplet.

https://www.digitalocean.com/

> This is an __affiliate link__, I make a bit of money if create a new account.  
>Feel free to __use your prefered hosting__. It makes no difference whatsoever.


## Install Docker

We are going to install CTFd with the Docker installation, as it makes things a bit easier.  
First, let's [__install docker__](https://docs.docker.com/engine/install/ubuntu/) on our __Ubuntu__ server.

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


## Install CTFd

Now, let's install _CTFd_. First __clone__ the __repository__.

```bash
git clone --depth 1 https://github.com/CTFd/CTFd.git
```

Then, you can go in the folder, and __start CTFd__ with __`docker compose`__:

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">root@ubuntu-s-1vcpu-2gb-fra1-01:~# cd CTFd/
root@ubuntu-s-1vcpu-2gb-fra1-01:~/CTFd# docker compose up -d
<font color="#66D9EF">[+] Running 4/4</font>
 <font color="#A6E22E">✔</font> Container ctfd-db-1     <font color="#A6E22E">Started</font>                      <font color="#66D9EF">0.7s </font>
 <font color="#A6E22E">✔</font> Container ctfd-ctfd-1   <font color="#A6E22E">Started</font>                      <font color="#66D9EF">1.4s </font>
 <font color="#A6E22E">✔</font> Container ctfd-nginx-1  <font color="#A6E22E">Started</font>                      <font color="#66D9EF">2.4s </font>
 <font color="#A6E22E">✔</font> Container ctfd-cache-1  <font color="#A6E22E">Started</font>                      <font color="#66D9EF">0.7s </font>
</code></pre></div>
{{< /rawhtml >}}


You should see the __port 80__ and __8000 listening__ on the server.

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# ss -ltupn | grep docker
tcp   LISTEN 0      4096         0.0.0.0:80        0.0.0.0:*    users:(("docker-proxy",pid=9941,fd=4))
tcp   LISTEN 0      4096         0.0.0.0:8000      0.0.0.0:*    users:(("docker-proxy",pid=9831,fd=4))
tcp   LISTEN 0      4096            [::]:80           [::]:*    users:(("docker-proxy",pid=9947,fd=4))
tcp   LISTEN 0      4096            [::]:8000         [::]:*    users:(("docker-proxy",pid=9839,fd=4))
```

__Access__ the __setup pane__ at `http://yourip`, and fill the informations about your CTF.

![](/deploy_ctfd/ctfd_setup.png)

### Note about data persistance

In the `docker-compose.yml`, you will find the following `volumes` sections:

```yaml
volumes:
  - .data/CTFd/logs:/var/log/CTFd
  - .data/CTFd/uploads:/var/uploads
  - .:/opt/CTFd:ro

volumes:
  - ./conf/nginx/http.conf:/etc/nginx/nginx.conf

volumes:
  - .data/mysql:/var/lib/mysql

volumes:
- .data/redis:/data
```

All the __data__ you configure in the app will be __stored in__ the __`.data`__ folder.
If you stop and restart the containers, the __data will be keeped__.

You can stop the containers with:
```bash
docker compose stop
```

# HTTPS Configuration

Now let's explore the hard question, the HTTPS configuration.
We will add a certificate from _Let's encrypt_.

#### Stop running containers

First, let's stop all the running containers since we need the port 80.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">root@ubuntu-s-1vcpu-2gb-fra1-01:~/CTFd# docker compose stop
<font color="#66D9EF">[+] Stopping 4/4</font>
 <font color="#A6E22E">✔</font> Container ctfd-nginx-1  <font color="#A6E22E">Stopped</font>                                 <font color="#66D9EF">0.3s </font>
 <font color="#A6E22E">✔</font> Container ctfd-cache-1  <font color="#A6E22E">Stopped</font>                                 <font color="#66D9EF">0.3s </font>
 <font color="#A6E22E">✔</font> Container ctfd-ctfd-1   <font color="#A6E22E">Stopped</font>                                 <font color="#66D9EF">1.1s </font>
 <font color="#A6E22E">✔</font> Container ctfd-db-1     <font color="#A6E22E">Stopped</font>                                 <font color="#66D9EF">0.4s </font>

root@ubuntu-s-1vcpu-2gb-fra1-01:~/CTFd# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
</code></pre></div>
{{< /rawhtml >}}


### Request a Let's Encrypt certificate

We will use the [docker method](https://eff-certbot.readthedocs.io/en/stable/install.html#alternative-1-docker) to request a certificate.

```bash
sudo docker run -it --rm --name certbot \
            -v "/etc/letsencrypt:/etc/letsencrypt" \
            -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
            -p 80:80 \
            -p 443:443 \
            certbot/certbot certonly
```

__Choose__ the __option 1__ to spin-up a temporary server:

```plain
How would you like to authenticate with the ACME CA?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

1: Runs an HTTP server locally which serves the necessary validation files under
the /.well-known/acme-challenge/ request path. Suitable if there is no HTTP
server already running. HTTP challenge only (wildcards not supported).
(standalone)

2: Saves the necessary validation files to a .well-known/acme-challenge/
directory within the nominated webroot path. A separate HTTP server must be
running and serving files from the webroot path. HTTP challenge only (wildcards
not supported). (webroot)

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 1
```

And __enter__ your __domain name__:
```plain
Please enter the domain name(s) you would like on your certificate (comma and/or
space separated) (Enter 'c' to cancel): ctf.lasne.pro
Requesting a certificate for ctf.lasne.pro

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/ctf.lasne.pro/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/ctf.lasne.pro/privkey.pem
This certificate expires on 2025-04-06.
```

### Copy the certificate to the nginx container

The certificate should be in `/etc/letsencrypt/live/your_domain/`:

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">root@ctf:~/CTFd# ls /etc/letsencrypt/live/ctf.lasne.pro/
README  <font color="#A1EFE4"><b>cert.pem</b></font>  <font color="#A1EFE4"><b>chain.pem</b></font>  <font color="#A1EFE4"><b>fullchain.pem</b></font>  <font color="#A1EFE4"><b>privkey.pem</b></font>
</code></pre></div>
{{< /rawhtml >}}

Copy the files to the `CTFd/conf/nginx` directory.

```bash
# Inside your CTFd repo directory
cp /etc/letsencrypt/live/your_domain/fullchain.pem ./conf/nginx/fullchain.pem
cp /etc/letsencrypt/live/your_domain/privkey.pem ./conf/nginx/privkey.pem
```

### Edit the `docker-compose.yml`

Edit the `docker-compose.yml` so that __nginx__ can __access these files__.

```yaml
volumes:
  - ./conf/nginx/http.conf:/etc/nginx/nginx.conf
  - ./conf/nginx/fullchain.pem:/certificates/fullchain.pem:ro
  - ./conf/nginx/privkey.pem:/certificates/privkey.pem:ro
```

Let's also __expose__ the __port 443__ for HTTPS:

```yaml
ports:
  - 80:80
  - 443:443
```

The nginx section of the `docker-compose.yml` should look like this:

```yaml
  nginx:
    image: nginx:stable
    restart: always
    volumes:
      - ./conf/nginx/http.conf:/etc/nginx/nginx.conf
      - ./conf/nginx/fullchain.pem:/certificates/fullchain.pem:ro
      - ./conf/nginx/privkey.pem:/certificates/privkey.pem:ro
    ports:
      - 80:80
      - 443:443
    depends_on:
      - ctfd
```

### Edit the nginx configuration

Edit the file `conf/nginx/http.conf` and add a redirection from http to https.

```yaml
server {
  listen 80;
  return 301 https://$host$request_uri;
}
```

And __configure__ the __certificates__:

```yaml
server {
  listen 443 ssl;

  ssl_certificate /certificates/fullchain.pem;
  ssl_certificate_key /certificates/privkey.pem;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers HIGH:!aNULL:!MD5;
```

Your complete `conf/nginx/http.conf` should look something like this:

```yaml
worker_processes 4;

events {

  worker_connections 1024;
}

http {

  # Configuration containing list of application servers
  upstream app_servers {

    server ctfd:8000;
  }

  
  server {
    listen 80;
    return 301 https://$host$request_uri;
  }

  server {

    listen 443 ssl;

    ssl_certificate /certificates/fullchain.pem;
    ssl_certificate_key /certificates/privkey.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:!aNULL:!MD5;

    gzip on;

    client_max_body_size 4G;

    # Handle Server Sent Events for Notifications
    location /events {

      proxy_pass http://app_servers;
      proxy_set_header Connection '';
      proxy_http_version 1.1;
      chunked_transfer_encoding off;
      proxy_buffering off;
      proxy_cache off;
      proxy_redirect off;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Host $server_name;
    }

    # Proxy connections to the application servers
    location / {

      proxy_pass http://app_servers;
      proxy_redirect off;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Host $server_name;
    }
  }
}
```

### Restart the containers

Everything should be good now. Restart the containers with `docker compose up -d`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">root@ubuntu-s-1vcpu-2gb-fra1-01:~/CTFd# docker compose up -d
<font color="#66D9EF">[+] Running 4/4</font>
 <font color="#A6E22E">✔</font> Container ctfd-cache-1  <font color="#A6E22E">Started</font>                                           <font color="#66D9EF">0.7s </font>
 <font color="#A6E22E">✔</font> Container ctfd-db-1     <font color="#A6E22E">Started</font>                                           <font color="#66D9EF">0.7s </font>
 <font color="#A6E22E">✔</font> Container ctfd-ctfd-1   <font color="#A6E22E">Started</font>                                           <font color="#66D9EF">1.4s </font>
 <font color="#A6E22E">✔</font> Container ctfd-nginx-1  <font color="#A6E22E">Started</font>                                           <font color="#66D9EF">2.5s </font>
</code></pre></div>
{{< /rawhtml >}}

Your CTFd is now available with HTTPS.

![CTFd instance with HTTPS](/deploy_ctfd/ctfd_https.png)


# Conclusion

In this article we have seen how to deploy a CTFd instance with HTTPS.
The HTTPS configuration adds a few steps, but they are pretty straightforward.

Happy hacking !