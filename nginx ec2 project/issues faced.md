# Issues Faced When Connecting ahamid.dev to an EC2 Instance Running Nginx

A breakdown of the problems I ran into when trying to get my domain working with Nginx on an EC2 instance, and how I fixed each one.

---

## Issue 1: Domain wasn't showing the Nginx welcome page

After connecting Cloudflare to my EC2 instance's public IPv4 address, visiting `ahamid.dev` didn't show the Nginx welcome page — but visiting the IP directly did.

### Diagnostics

To figure out where the problem was, I ran four checks:

- ✅ Confirmed the EC2 security group had inbound rules open for **HTTP (port 80)** and **HTTPS (port 443)**
  
- ✅ Verified the IP on the EC2 instance matched the one in the Cloudflare DNS record
  
- ✅ Used `nslookup ahamid.dev` to confirm the domain resolved to the correct IP

- ✅ Confirmed Nginx was actively running:
  ```bash
  sudo systemctl status nginx
  ```
- ✅ Checked HTTP was reachable on the domain:
  ```bash
  curl -v http://ahamid.dev
  ```

All checks passed, which pointed to the **Nginx config** as the issue.

### Root Cause

Amazon Linux doesn't use the `sites-available/sites-enabled` folder structure. Running:

```bash
sudo ls /etc/nginx/sites-enabled/
```

Returned:
```
ls: cannot access '/etc/nginx/sites-enabled/': No such file or directory
```

Running `ls /etc/nginx/` showed a `conf.d` directory instead — this is where Amazon Linux keeps its Nginx configs. There was also no server block configured for the domain, so Nginx didn't know what to do when someone visited `ahamid.dev`.

### Fix

Created a config file for the domain:

```bash
sudo nano /etc/nginx/conf.d/ahamid.dev.conf
```

Added the following server block:

```nginx
server {
    listen 80;
    server_name ahamid.dev www.ahamid.dev;
    root /var/www/html;
    index index.html;
    location / {
        try_files $uri $uri/ =404;
    }
}
```

Tested and reloaded Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Issue 2: `ERR_QUIC_PROTOCOL_ERROR` — No SSL Certificate

<img width="392" height="144" alt="image" src="https://github.com/user-attachments/assets/326174a8-05bb-441c-89ac-27e87b72a6f0" />

After fixing the config, visiting `ahamid.dev` showed `ERR_QUIC_PROTOCOL_ERROR`. The browser was trying to connect over HTTPS but there was no SSL certificate installed.

### Fix

Installed Certbot and got a free SSL certificate from Let's Encrypt:

```bash
sudo dnf install certbot python3-certbot-nginx -y
sudo certbot --nginx -d ahamid.dev
```

Certbot automatically updated the Nginx config to handle HTTPS. Then reloaded Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Issue 3: `404 Not Found` — Wrong Root Directory

Even after SSL was working, the site was returning a 404. The config was pointing to `/var/www/html` as the root directory, but that folder doesn't exist on Amazon Linux.

### Fix

Updated the `root` line in `/etc/nginx/conf.d/ahamid.dev.conf`:

```nginx
root /usr/share/nginx/html;
```

This is where Amazon Linux keeps the default Nginx files. After saving the change, reloaded Nginx and did a hard refresh in the browser (`Ctrl+Shift+R` on Windows / `Cmd+Shift+R` on Mac) to clear the cache.

---

## Summary

| Issue | Cause | Fix |
|---|---|---|
| Domain not loading | No Nginx server block for the domain | Created config in `conf.d/` |
| `ERR_QUIC_PROTOCOL_ERROR` | No SSL certificate | Installed Certbot + Let's Encrypt cert |
| `404 Not Found` | Wrong root directory in config | Changed root to `/usr/share/nginx/html` |
