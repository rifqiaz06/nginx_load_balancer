---
# 🚀 Load Balancer Setup Documentation NGINX
---

## 📘 1. Manual NGINX Load Balancer (on VM)

### 🧠 Overview:

- Pure VM setup (VirtualBox)
- No containers, no automation
- Simple reverse proxy using NGINX

### 📡 Topology:

```
+------------------+
|   VM1 (LB)       |  --> NGINX (Load Balancer)
|  192.168.0.11    |
+------------------+
        |
        |   Round-robin to:
        V
+------------------+          +------------------+
|   VM2 (App1)     |   <--->  |   VM3 (App2)     |
|  192.168.0.12:80 |          |  192.168.0.13:80 |
+------------------+          +------------------+
```

### 🔧 Setup Steps:

#### Step 1: Install NGINX on VM1

```bash
sudo apt update
sudo apt install nginx -y
```

#### Step 2: Configure NGINX Load Balancer

Edit `/etc/nginx/sites-available/default`:

```nginx

upstream appRoundRobin { #Round Robin
    server 192.168.0.12;
    server 192.168.0.13;
}
upstream appLeastConn { #Least Connection
    least_conn;
    server 192.168.0.12;
    server 192.168.0.13;
}
upstream appIpHash { #IP Hash
    ip_hash;
    server 192.168.0.12;
    server 192.168.0.13;
}
#Log Format
log_format upstreamlog 'remote_addr: $remote_addr | '
                'remote_user: $remote_user | '
                'time_local: $time_local | '
                'request: $request | '
                'status: $status | '
                'body_bytes_sent: $body_bytes_sent | '
                'http_referer: $http_referer | '
                'upstream_addr: $upstream_addr | '
                'upstream_response_time: $upstream_response_time | '
                'request_time: $request_time | '
                'msec: $msec | '
                'http_user_agent: $http_user_agent';

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=cache_one:5m inactive=10m; #Cache Path
limit_req_zone $binary_remote_addr zone=ourRateLimiter:10m rate=15r/m; #max 60/15 = 4, tiap user cuma bisa akses tiap 4 detik sekali. Dan bisa diubah sesuai kebutuhan. misal: 15r/s,untuk 15 request per-detik. #Rate Limiter

server {
        listen 80;
        access_log /var/log/app/app.log upstreamlog; #Logging

        location / {
                limit_req zone=ourRateLimiter; #Rate Limiter
                proxy_set_header X-custom-header this-is-example-of-load-balancer; #Custom Header/Circuit Breaker
                proxy_pass http://appRoundRobin; #Round Robin
        }
        location /ipHash {

                proxy_pass http://appIpHash; #IP Hash
        }
        location /leastConn {
                proxy_pass http://appLeastConn; #Least Connection
        }
        location /metadata { #Caching, Pastikan endpoint /metadata sudah di buat di aplikasinya
            proxy_cache cache_one;
            proxy_cache_min_uses 5; #Cache akan aktif minimal kalau user 5x akses
            proxy_cache_methods HEAD GET;
            proxy_cache_valid 200 304 30s; #Cache cuma berlaku selama 30 detik
            proxy_cache_key $uri;
            proxy_pass http://app;
        }
}
```

#### Step 3: Restart and Test

```bash
sudo nginx -t
sudo systemctl restart nginx
```

#### Step 4: Logging

```bash
sudo apt install ccze
sudo tail -f /var/log/nginx/access.log atau error.log | ccze
sudo tail -f /var/log/nginx/app/app.log | ccze
```
#### Optional Enhancements:

- Failover: use `backup;` server
- Spliting Load Weight: use `weight`; server
- HTTPS: setup self-signed or Let’s Encrypt cert
