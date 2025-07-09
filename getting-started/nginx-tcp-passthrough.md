````markdown
# Nginx TCP Passthrough Guide

This guide explains how to configure Nginx for pure TCP passthrough from ports **80 → 30080** and **443 → 30443** using the `stream` module.

---

## 1. Install the dynamic stream module

```bash
sudo apt-get update
sudo apt-get install -y libnginx-mod-stream
````

---

## 2. Update your main Nginx configuration

1. Backup your existing `/etc/nginx/nginx.conf`.
2. Edit `/etc/nginx/nginx.conf` so it:

   * Loads the `ngx_stream_module`.
   * Defines a top-level `stream` block that includes your stream configs.
   * Keeps your normal HTTP blocks intact.

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;

# 1) load the stream (TCP) module
load_module modules/ngx_stream_module.so;

events {
    worker_connections 1024;
}

# 2) top-level stream block
stream {
    # include all your TCP-proxying configs here
    include /etc/nginx/streams-enabled/*.conf;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # your normal HTTP / virtual-host configs
    include /etc/nginx/conf.d/*.conf;
}
```

---

## 3. Create your stream-enabled directory

```bash
sudo mkdir -p /etc/nginx/streams-enabled
```

---

## 4. Add the passthrough config

Create the file `/etc/nginx/streams-enabled/k8s-passthrough.conf` with **only** the TCP proxy definitions:

```nginx
# /etc/nginx/streams-enabled/k8s-passthrough.conf

upstream ingress_http {
    server 127.0.0.1:30080;
}

upstream ingress_https {
    server 127.0.0.1:30443;
}

server {
    listen      80       so_keepalive=on;
    proxy_pass  ingress_http;
}

server {
    listen      443      so_keepalive=on;
    proxy_pass  ingress_https;
}
```

---

## 5. Test and reload Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

After completing these steps, Nginx will forward raw TCP traffic on ports **80** and **443** to your Kubernetes Ingress controller on ports **30080** and **30443**, leaving HTTP and TLS termination to the Ingress.
