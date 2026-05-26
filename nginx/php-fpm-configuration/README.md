# Nginx + PHP-FPM Configuration

## Overview

This lab covers the configuration of **nginx** as a web server and **php-fpm** as a PHP process manager, connecting them via a **Unix socket**. This is the foundation of the **LEMP stack** (Linux, Nginx, MySQL, PHP).

## Concepts

### LEMP Stack vs LAMP Stack

|               | LAMP               | LEMP                       |
|---------------|--------------------|----------------------------|
| Web Server    | Apache             | Nginx                      |
| PHP Execution | mod_php (built-in) | php-fpm (external process) |
| Architecture  | Thread-based       | Event-based (async)        |

With Apache, PHP runs directly inside the web server process (`mod_php`). With nginx, PHP execution is delegated to **php-fpm** via a socket. Nginx does not understand PHP natively.

### FastCGI

CGI (Common Gateway Interface) was the original way web servers ran external programs. A new process was spawned per request, then killed. Slow.

**FastCGI** solves this: processes stay **permanently running** and wait for incoming requests. No start/stop per request.

```
CGI:      Request -> spawn process -> execute -> kill process -> Response
FastCGI:  Request -> already running process -> execute -> Response
```

### Unix Socket

A Unix socket is a **special file** on the filesystem that acts as a communication channel between two processes on the same machine.

```
nginx process                    php-fpm process
     │                                │
     │---- writes request ---->       │
     │     /var/run/php-fpm/          │
     │     default.sock               │
     │<--- reads response ---------   │
```

The OS manages the data transfer in kernel memory. The `.sock` file is just the entry point, no data is actually written to disk. This makes it faster than TCP.

|           | Unix Socket                            | TCP Socket                    |
|-----------|----------------------------------------|-------------------------------|
| Transport | Kernel memory via filesystem           | Network stack                 |
| Scope     | Local only                             | Local or remote               |
| Speed     | Faster                                 | Slightly slower               |
| Config    | `fastcgi_pass unix:/path/to/file.sock` | `fastcgi_pass 127.0.0.1:9000` |

### php-fpm (FastCGI Process Manager)

php-fpm is a **process manager for PHP**. It:
1. Starts at boot
2. Creates a **pool** of PHP worker processes ready to handle requests
3. Listens for requests via socket
4. Distributes requests across available workers
```
php-fpm starts
    └── Pool [<pool-name>]
            ├── PHP Worker 1  -> idle
            ├── PHP Worker 2  -> idle
            ├── PHP Worker 3  -> processing request
            ├── PHP Worker 4  -> idle
            └── PHP Worker 5  -> idle
```

Each pool has its own configuration: socket path, user/group, number of workers, and log paths. Multiple pools allow process isolation between applications.

### Full Request Flow

```
curl http://server:<port>>/index.php
    │
    ▼
nginx (Port <port>)
    │ location ~ \.php$ matched
    ▼
/var/run/php-fpm/<pool-name>.sock
    │
    ▼
php-fpm Pool [<pool-name>]
    │
    ▼
Executes /var/www/html/index.php
    │
    ▼
Returns HTML → nginx → curl
```

## Steps

### 1. Install nginx

```bash
sudo yum install nginx -y
```

### 2. Configure nginx

Edit `/etc/nginx/nginx.conf` and update the server block:

```nginx
server {
    listen       8093;
    listen       [::]:8093;
    server_name  _;
    root         /var/www/html;
 
    include /etc/nginx/default.d/*.conf;
 
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php-fpm/<pool-name>.sock;
        fastcgi_index index.php;
        include fastcgi.conf;
    }
 
    error_page 404 /404.html;
    location = /404.html {}
 
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {}
}
```

Key directives explained:

| Directive                 | Purpose                                                           |
|---------------------------|-------------------------------------------------------------------|
| `listen 8093`             | Port nginx listens on                                             |
| `root /var/www/html`      | Document root where files are served from                         |
| `location ~ \.php$`       | Matches all requests ending in `.php`                             |
| `fastcgi_pass unix:...`   | Forwards PHP requests to php-fpm via Unix socket                  |
| `fastcgi_split_path_info` | Splits URL into script name and path info (useful for frameworks) |
| `include fastcgi.conf`    | Loads predefined FastCGI params including `SCRIPT_FILENAME`       |

### 3. Start nginx

```bash
sudo systemctl enable nginx
```

Verify the port is listening:

```bash
ss -tlnp | grep 8093
```

### 4. Install php-fpm 8.2

```bash
sudo yum install php
```

Install required PHP modules:

```bash
sudo yum install php:8.2
```

### 5. Configure php-fpm Pool

Copy the default pool config as a base:

```bash
sudo cp /etc/php-fpm.d/www.conf /etc/php-fpm.d/<pool-name>.conf
```

Edit `/etc/php-fpm.d/<pool-name>.conf` with the following key settings:

```ini
[<pool-name>]
user = nginx
group = nginx
listen = /var/run/php-fpm/<pool-name>.sock
listen.owner = nginx
listen.group = nginx
listen.allowed_clients = 127.0.0.1
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
slowlog = /var/log/php-fpm/<pool-name>-slow.log
php_admin_value[error_log] = /var/log/php-fpm/<pool-name>-error.log
php_admin_flag[log_errors] = on
php_value[session.save_handler] = files
php_value[session.save_path] = /var/lib/php/session
php_value[soap.wsdl_cache_dir] = /var/lib/php/wsdlcache
```

Pool parameter reference:

| Parameter                       | Purpose                                                   |
|---------------------------------|-----------------------------------------------------------|
| `[<pool-name>]`                 | Pool name. Determines socket name convention              |
| `user` / `group`                | Linux user the php-fpm workers run as                     |
| `listen`                        | Path to the Unix socket file                              |
| `listen.owner` / `listen.group` | Owner of the socket file (must match nginx user)          |
| `listen.allowed_clients`        | Restrict socket access to localhost                       |
| `pm = dynamic`                  | Process manager mode. Workers scale up/down based on load |
| `pm.max_children`               | Maximum number of worker processes                        |
| `pm.start_servers`              | Workers spawned at startup                                |
| `pm.min_spare_servers`          | Minimum idle workers                                      |
| `pm.max_spare_servers`          | Maximum idle workers                                      |

### 6. Create the Socket Directory

php-fpm creates the socket file automatically on start, but the parent directory must exist:

```bash
sudo mkdir -p /var/run/php-fpm/
```

### 7. Validate and Start php-fpm

Test the configuration:

```bash
sudo php-fpm --test
```

Enable and start:

```bash
sudo systemctl enable php-fpm
```

Verify the socket was created:

```bash
ls -la /var/run/php-fpm/
# Look for: srwxrwxrwx ... <pool-name>.sock
# 's' at the start of permissions = socket file
```

Verify with ss:

```bash
sudo ss -xlnp | grep php
```

### 8. Test

```bash
curl http://server:8093/index.php
curl http://server:8093/info.php
# Expected: Full phpinfo() HTML output
```

## Troubleshooting

| Issue                  | Command                        | Fix                                            |
|------------------------|--------------------------------|------------------------------------------------|
| php-fpm fails to start | `sudo php-fpm --test`          | Check config syntax                            |
| Socket not created     | `ls -la /var/run/php-fpm/`     | Create directory with `mkdir -p`               |
| nginx config error     | `sudo nginx -t`                | Check nginx config syntax                      |
| Port not listening     | `ss -tlnp \| grep 8093`        | Check nginx is running                         |
| 502 Bad Gateway        | `sudo journalctl -xeu php-fpm` | Socket path mismatch between nginx and php-fpm |

### Common Mistakes

- Socket directory not created before starting php-fpm
- `listen.owner` not matching the nginx user (permission denied on socket)
- Missing required pool parameters: `user`, `group`, `pm`, `[pool]`, `listen`