# EC2 + NGINX + Custom Domain Deployment

## Objective

Deploy an AWS EC2 instance running NGINX and connect it to a custom domain using Cloudflare DNS.

---

## Step 1 — Buy a Domain

Bought the domain `ahamid.dev` on Cloudflare.

---

## Step 2 — Launch EC2 Instance

Created an EC2 instance on AWS with the following configuration:

- **AMI:** Amazon Linux
- **Inbound Rules:**

| Port | Protocol | Description |
|------|----------|-------------|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |

Accessed the instance using **AWS EC2 Instance Connect**

---

## Step 3 — Install and Run NGINX

Update packages:

```bash
sudo yum update -y
```

Install NGINX:

```bash
sudo yum install nginx -y
```

Start NGINX:

```bash
sudo systemctl start nginx
```

Enable NGINX to start automatically on boot:

```bash
sudo systemctl enable nginx
```

Check that it's running:

```bash
sudo systemctl status nginx
```

---

## Step 4 — Verify the Web Server

Opened the EC2 public IP in the browser:

```
http://YOUR_PUBLIC_IP
```

Successfully received the default NGINX welcome page — confirming the server was running correctly.

---

## Step 5 — Configure the Domain

In Cloudflare DNS, created an A record pointing the domain to the EC2 instance:

| Type | Name | Value |
|------|------|-------|
| A | @ | EC2_PUBLIC_IP |

> **Note:** Cloudflare proxy was initially enabled (orange cloud). It was switched to **DNS only** (grey cloud) during troubleshooting to rule out proxy-related issues.

---

## Step 6 — Verify DNS Resolution

Confirmed the domain resolved to the correct EC2 public IP:

```bash
nslookup ahamid.dev
```

---

## Step 7 — Create an NGINX Config for the Domain

Amazon Linux doesn't use the `sites-available/sites-enabled` structure. Instead, configs go in `conf.d/`:

```bash
sudo nano /etc/nginx/conf.d/ahamid.dev.conf
```

Added the following server block:

```nginx
server {
    listen 80;
    server_name ahamid.dev www.ahamid.dev;
    root /usr/share/nginx/html;
    index index.html;
    location / {
        try_files $uri $uri/ =404;
    }
}
```

Test and reload NGINX:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Step 8 — Install SSL Certificate

Installed Certbot to get a free SSL certificate from Let's Encrypt:

```bash
sudo dnf install certbot python3-certbot-nginx -y
```

Run Certbot for the domain:

```bash
sudo certbot --nginx -d ahamid.dev
```

Certbot automatically updated the NGINX config to handle HTTPS and set up auto-renewal.

Reload NGINX to apply changes:

```bash
sudo systemctl reload nginx
```

---

## Step 9 — Verify the Deployment

Visited `https://ahamid.dev` in the browser and confirmed the site was live and secure.
