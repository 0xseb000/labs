# Nginx Load Balancer Configuration

## Objective

Configure a Nginx load balancer to distribute HTTP traffic across three Apache app servers in a high-availability setup.

## Environment

| Role          | Host    |
|---------------|---------|
| Load Balancer | stlb01  |
| App Server 1  | stapp01 |
| App Server 2  | stapp02 |
| App Server 3  | stapp03 |

## Steps

### 1. Install Nginx on the load balancer

```bash
sudo yum install nginx -y
```

### 2. Check Apache port on app servers

Before configuring the upstream block, verifying which port Apache is listening on:

```bash
sudo ss -tlpn | grep httpd
```

In this case, Apache was running on port **6000** on all three app servers.

### 3. Configure /etc/nginx/nginx.conf

Added an `upstream` block inside the `http` context, and a `proxy_pass` directive inside the `location /` block:

```nginx
http {
    upstream appServers {
        server <app1-ip>:6000;
        server <app2-ip>:6000;
        server <app3-ip>:6000;
    }

    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;

        location = /404.html { }
        location = /50x.html { }

        location / {
            proxy_pass http://appServers;
        }

        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
    }
}
```

### 4. Validate and start Nginx

```bash
nginx -t
sudo systemctl start nginx
```

### 5. Verify

```bash
curl http://stlb01:80
```

## Key Concepts

- **Nginx** acts as a reverse proxy and load balancer, listening on port 80 and forwarding requests to the backend app servers
- **Apache** runs on the app servers on a non-standard port (6000).
- The `upstream` block defines the pool of backend servers
- `location /` is a catch-all that matches all incoming requests
- Always verify the backend port with `ss -tlpn` before configuring the upstream block
- Validate config with `nginx -t` before starting or reloading Nginx