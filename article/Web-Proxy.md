Hosting Multiple Websites Behind a Single Public IP
===================================================

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*sm3eonWRXd3-KmMeweMdGw.png)







Route multiple domains or subdomains to different backend servers using one public IP, SSL termination, and host-based routing

One of the most common challenges in infrastructure design is exposing multiple websites to the internet when you only have one public IP address. And fortunately, modern web servers solve this problem elegantly, by placing Nginx in front of your backend servers as a reverse proxy, you can host dozens (or even hundreds) of websites behind a single public IP while centrally managing SSL certificates, routing, and security.

Let’s build a simple architecture where multiple websites share one public IP, Nginx performs SSL termination, and requests are routed to the appropriate backend server based on the requested hostname.

Architecture
------------


Installation Steps
------------------

### Step 1 — Configure DNS

Create DNS records for each website pointing to the same public IP.

```
abc.example.com A x.x.x.x
xyz.example.com A x.x.x.x
```

### Step 2 — Configure Port Forwarding

Configure your firewall or edge router to forward HTTPS traffic to the Nginx server.

```
Public IP :443
        ↓
Nginx Internal IP :443
```

This is the only forwarding rule required.

As additional websites are added later, the firewall configuration remains unchanged.

### Step 3 — Install Nginx

Ubuntu/Debian:

```
sudo apt update
sudo apt install nginx -y
nginx -v (verify the installation)
```

### Step 4 — Install the SSL Certificate

Copy your SSL certificate and private key to the Nginx server.

```
sudo mkdir -p /etc/nginx/ssl
sudo cp fullchain.crt /etc/nginx/ssl/
sudo cp private.key /etc/nginx/ssl/
sudo chmod 600 /etc/nginx/ssl/private.key
```

### Step 5 — Configure Nginx

Create one server block for each website.

Example:

```
server {
    listen 443 ssl;
    server_name abc.example.com;
    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    location / {
        proxy_pass http://192.168.10.20;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Create another server block for the second website.

```
server {
    listen 443 ssl;
    server_name xyz.example.com;
    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    location / {
        proxy_pass http://192.168.10.30;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

To redirect HTTP to HTTPS, you can add the snippet to each block

```
server {
    listen 80;
    server_name abc.example.com;
    return 301 https://$host$request_uri;
}
```

Repeat this pattern for every additional website.

Backend Server Configuration
----------------------------

The backend web server only needs to serve the website normally.

Whether you’re using:

*   IIS
*   Apache
*   Nginx
*   Tomcat
*   Node.js
*   Docker containers

the reverse proxy forwards the original Host header so the application knows which website was requested.

For IIS, an example binding would be:

```
Type: HTTP
Port: 80
Host Name:
abc.example.com
```
