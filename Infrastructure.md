## Playground Infrastructure
All playground services are hosted on a single [Oracle Cloud](https://cloud.oracle.com) instance (Ampere A1 with 1 OCPU and 6 GB memory), because:
* This is part of Oracle's free tier and I do not want to ask for or even depend on donations.
* I do not expect much traffic on the playground, so a single instance is probably enough.

The instance is running the latest version of Ubuntu. First, I edited `/root/.ssh/authorized_keys` to enable root login, because this is more convenient for me. Then, I set up the system as follows:

```sh
# Install docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
apt update
apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Install nginx
apt install nginx
rm /etc/nginx/sites-enabled/default

# Install certbot
apt install certbot python3-certbot-nginx
echo 'certbot renew -q --nginx' > /etc/cron.daily/certbot

# Update firewall
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
iptables -I INPUT -p tcp --dport 443 -j ACCEPT

# Install flask-migrate
apt install python3-pip
pip3 install flask flask-migrate
```

I created the following file in `/etc/nginx/sites-available/sites`:

```nginx
server_tokens off;

server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

This redirects all HTTP requests to HTTPS. I enabled the configuration file as follows:

```sh
ln -s /etc/nginx/sites-available/sites /etc/nginx/sites-enabled/sites
```

## Adding a service

To install a new service, I use the following steps:

1. Add the relevant subdomains to the DNS records of nintendo-playground.com.

2. Obtain server certificates with certbot.
```sh
certbot certonly --nginx -d dauth-lp1.ndas.srv.nintendo-playground.com
certbot certonly --nginx -d dcert-lp1.ndas.srv.nintendo-playground.com
certbot certonly --nginx -d dadmin-lp1.ndas.mng.nintendo-playground.com
```

3. Clone the git repository: `git clone https://github.com/nintendo-playground/dauth-server`
4. Add missing files if necessary. For example, for the dauth server you must provide `prod.keys` and `dev.keys` manually.

5. Configure the service with `python3 scripts/configure.py`.
```
Project name: 
Issuer (dauth): dauth-lp1.ndas.srv.nintendo.net
JKU (dauth): https://dcert-lp1.ndas.srv.nintendo.net/keys
Port (dauth): 10000
Port (dcert): 10001
Port (dadmin): 10002
Username (dadmin): BKkmd93m33J56hDDtXc25w
Password (dadmin): 8tywMwjIqFPgMuQV80EH-w
Device type (cert): NX Prod 1
```

6. Initialize the database if necessary: `flask db upgrade`
7. Start the service: `docker compose up -d`

8. Add the service to the NGINX configuration file. Most services look like this:
```
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    ssl_certificate /etc/letsencrypt/live/dcert-lp1.ndas.srv.nintendo-playground.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dcert-lp1.ndas.srv.nintendo-playground.com/privkey.pem;

    server_name dcert-lp1.ndas.srv.nintendo-playground.com;

    location / {
        proxy_pass http://localhost:10001;
    }
}
```

If the service requires a client certificate, it might look like this:
```
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    ssl_certificate /etc/letsencrypt/live/dauth-lp1.ndas.srv.nintendo-playground.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dauth-lp1.ndas.srv.nintendo-playground.com/privkey.pem;

    ssl_client_certificate /etc/nginx/ca-certificates/NintendoNXCA2Prod1.pem;
    ssl_verify_client on;

    server_name dauth-lp1.ndas.srv.nintendo-playground.com;

    location / {
        proxy_pass http://localhost:10000;
        proxy_set_header X-Device-Certificate $ssl_client_escaped_cert;
    }
}
```

9. Reload NGINX configuration: `nginx -s reload`
10. Visit the admin panel and prepare the database (e.g. client ids for dauth).

Done!

## Updating a service
1. Pull the latest version from the git repository: `git pull`
2. Rebuild the docker images: `docker compose build`
3. Stop the service: `docker compose down`
4. Update the database: `flask db upgrade`
5. Restart the service: `docker compose up -d`

Step 3 and 4 can usually be skipped.
