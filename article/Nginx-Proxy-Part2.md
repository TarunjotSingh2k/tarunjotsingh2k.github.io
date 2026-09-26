Hosting Multiple Websites Behind a Single Public IP — Part 2: Optimizing the Nginx Config
=========================================================================================

![Nginx-Proxy](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*sm3eonWRXd3-KmMeweMdGw.png)

[Reference](https://medium.com/@tarunjotsingh2k/hosting-multiple-websites-behind-a-single-public-ip-part-2-optimizing-the-nginx-config-eccc50cfe06d?sharedUserId=tarunjotsingh2k)


_A follow-up to_ [_Hosting Multiple Websites Behind a Single Public IP_](https://medium.com/@tarunjotsingh2k/hosting-multiple-websites-behind-a-single-public-ip-34b9b6382bde)

In Part 1, we configured Nginx as a reverse proxy to host multiple websites behind a single public IP.

The setup was simple: Nginx receives the request, checks the hostname, and forwards it to the correct backend server.

It works well, but there is one problem.

As the number of websites increases, the Nginx configuration also becomes repetitive.

The Problem With Multiple `server` Blocks
-----------------------------------------

In the previous setup, every website had its own `server` block:

```
server {
    listen 443 ssl;
    server_name abc.example.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;    location / {
        proxy_pass [http://192.168.10.20;](http://192.168.10.20;)        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

For another website, we would need another almost identical block.

The only things changing are:

*   Server_Name
*   Backend IP and Port

Everything else is repeated.

For a few websites, this is fine. But with 10, 20, or more websites, maintaining the same configuration multiple times becomes unnecessary.

_This is where Nginx’s_ `_map_` _directive helps._

Using `map` for Backend Routing
-------------------------------

Instead of creating a separate `server` block for every website, we can create a simple mapping between the hostname and its backend:

```
map $host $backend {
    lms.example.com       http://192.168.1.10:8000;
    admin.example.com     http://192.168.1.10:8001;
    student.example.com   http://192.168.1.10:8002;
}
```

Now Nginx knows:

```
lms.example.com      → 192.168.1.10:8000
admin.example.com    → 192.168.1.10:8001
student.example.com  → 192.168.1.10:8002
```

The main configuration can therefore be shared by all websites.

Optimized Configuration
-----------------------

Here is the complete configuration:

```
map $host $backend {
    lms.example.com       http://192.168.1.10:8000;
    admin.example.com     http://192.168.1.10:8001;
    student.example.com   http://192.168.1.10:8002;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}server {
    listen 80;
    server_name *.example.com;    return 301 [https://$host$request_uri;](https://$host$request_uri;)
}server {
    listen 443 ssl http2;
    server_name *.example.com;    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;    ssl_protocols TLSv1.2 TLSv1.3;    client_max_body_size 100M;    proxy_connect_timeout 10s;
    proxy_send_timeout    60s;
    proxy_read_timeout    60s;    location / {        if ($backend = "") {
            return 404;
        }        proxy_pass $backend;        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;        proxy_http_version 1.1;        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```

Now, adding another website only requires one line under the map section:

```
newapp.example.com   http://192.168.1.10:8003;
```

No additional `server` block is required.

Why `$host`?
------------

The routing is based on the Nginx `$host` variable:

```
map $host $backend
```

For example, when a user opens:

```
https://lms.example.com
```

Nginx gets:

```
$host = lms.example.com
```

It then looks for that hostname in the `map` and sends the request to the configured backend.

Using `$host` is preferable here to `$http_host` because Nginx normalizes `$host` and removes the port number if one is present.

What About Unknown Subdomains?
------------------------------

There is one important point here.

Our wildcard configuration:

```
server_name *.example.com;
```

can receive requests for any subdomain.

If we configure a default backend like this:

```
map $host $backend {
    lms.example.com       http://192.168.1.10:8000;
    admin.example.com     http://192.168.1.10:8001;

    default               http://192.168.1.10:8000;
}
```

then a request for:

```
random.example.com
```

could be sent to the default backend.

That’s not something we want.

Instead, we simply don’t define a default backend and check whether a match exists:

```
if ($backend = "") {
    return 404;
}
```

This means:

```
Configured hostname    → Backend
Unknown hostname       → 404
```

This prevents an unconfigured or random subdomain from being accidentally sent to a real application.

WebSocket Support
-----------------

The second `map` is for WebSocket connections:

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

The following headers then pass the WebSocket upgrade through Nginx:

```
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

This is required by applications that use WebSockets for features such as real-time notifications, chat, or live dashboards.

Without the correct WebSocket configuration, the normal website may work while real-time features fail.

Hiding the Nginx Version
------------------------

One small security improvement can also be made in `/etc/nginx/nginx.conf`.

Inside the `http {}` section:

```
server_tokens off;
```

This prevents Nginx from exposing its exact version in the `Server` header and default error pages.

For example, instead of:

```
Server: nginx/1.24.0
```

the response will show:

```
Server: nginx
```

This is not a replacement for keeping Nginx patched, but there is little reason to expose the exact version publicly.

Testing the Configuration
-------------------------

Before applying the changes:

```
sudo nginx -t
```

If the configuration is valid:

```
sudo systemctl reload nginx
```

Then test a configured hostname:

```
curl -vk https://lms.example.com
```

And test an unknown hostname:

```
curl -vk https://random.example.com
```

The configured hostname should reach its backend, while the unknown hostname should return:

```
404 Not Found
```

Finally, check the `Server` header:

```
curl -sI https://lms.example.com | grep -i server
```

It should show:

```
Server: nginx
```

without the Nginx version.

Summary
-------

The original configuration works well, but maintaining a separate `server` block for every website can become difficult as the environment grows.

Using Nginx’s `map` directive gives us a cleaner approach:

*   One common `server` block
*   One line per hostname
*   Centralized SSL and proxy settings
*   WebSocket support
*   Unknown subdomains return `404`
*   Easier maintenance
