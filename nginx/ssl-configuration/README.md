# Nginx SSL Configuration on CentOS 9

## Task

Install and configure Nginx on App Server 3 (`stapp03`) with SSL using a self-signed certificate. Create a basic `index.html` and verify the setup from the jump host via HTTPS.

**Environment:** CentOS Stream 9

---

## Steps

### 1. Check OS Version

```bash
cat /etc/os-release
# VERSION_ID="9" -> CentOS Stream 9, nginx available in default repos
```

### 2. Install and Enable Nginx

```bash
sudo yum install nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 3. Move SSL Certificate and Key

The certificate and key were pre-placed in `/tmp/`. Moved them to the appropriate system locations following best practice:

```bash
# certs/ already existed, private/ had to be created
sudo mkdir /etc/ssl/private
sudo chmod 700 /etc/ssl/private
 
sudo mv /tmp/nautilus.crt /etc/ssl/certs/
sudo mv /tmp/nautilus.key /etc/ssl/private/
 
sudo chmod 644 /etc/ssl/certs/nautilus.crt
sudo chmod 600 /etc/ssl/private/nautilus.key
```

**Why separate folders:**
- `/etc/ssl/certs/` -> public certificates, readable by all (`644`)
- `/etc/ssl/private/` -> private keys, root-only access (`600`), directory itself `700`
### 4. Locate and Edit Nginx Configuration

```bash
sudo find / -type f -name "nginx*"
sudo vim /etc/nginx/nginx.conf
```

The config already contained a commented-out TLS server block. Uncommented it and updated the certificate paths:

```nginx
server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  _;
    root         /usr/share/nginx/html;
 
    ssl_certificate     "/etc/ssl/certs/nautilus.crt";
    ssl_certificate_key "/etc/ssl/private/nautilus.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers PROFILE=SYSTEM;
    ssl_prefer_server_ciphers on;
 
    include /etc/nginx/default.d/*.conf;
 
    error_page 404 /404.html;
        location = /40x.html {
    }
 
    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```

### 5. Create index.html

```bash
echo "Welcome!" | sudo tee /usr/share/nginx/html/index.html
```

> The document root is defined by `root /usr/share/nginx/html;` in the server block. An existing default CentOS test page was overwritten.

### 6. Test Configuration and Restart

```bash
sudo nginx -t
# nginx: configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful
 
sudo systemctl restart nginx
sudo systemctl status nginx
```

> `nginx -t` validates the config without restarting the service. Important habit before any restart to avoid taking down a running server with a broken config.

### 7. Verify from Jump Host

```bash
curl -Ik https://stapp03/
```

**Response:**

```
HTTP/2 200
server: nginx/1.20.1
date: Sun, 17 May 2026 20:06:02 GMT
content-type: text/html
content-length: 9
```

`200 OK` over HTTPS confirms Nginx is running with SSL. `content-length: 9` corresponds to `Welcome!\n`.

> `-I` fetches headers only, `-k` ignores the self-signed certificate warning.

---

## Key Concepts

**SSL/TLS:** Encrypts communication between client and server using a public/private key pair. The `.crt` (public) is sent to the browser, the `.key` (private) stays on the server. Self-signed certificates work but trigger browser warnings. Use Let's Encrypt for production.

**`nginx -t`:** Always run before `systemctl restart nginx` to catch config errors without taking down the running service.

**File permissions for SSL:**

| File           | Permission | Reason                     |
|----------------|------------|----------------------------|
| `.crt`         | `644`      | Public, readable by all    |
| `.key`         | `600`      | Private, root-only         |
| `private/` dir | `700`      | Root-only directory access |

**`server_name _;`** Wildcard, accepts requests regardless of hostname or IP.