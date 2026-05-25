# Apache Multi-Site Configuration on Custom Port

## Overview

Configured an Apache HTTP server (`httpd`) on a Linux app server to serve two static websites on a non-default port. Both sites are accessible as subdirectories under the same host.

## Steps

### 1. Install Apache

Connected to the app server and verified that `httpd` was not installed, then installed it along with its dependencies.

```bash
sudo systemctl status httpd
sudo yum install httpd
```

---

### 2. Configure Custom Port

Edited the Apache configuration file to change the listening port from `80` to `8086`.

```bash
cd /etc/httpd/conf
sudo vim httpd.conf
```

Changed:
```
Listen 80
```
To:
```
Listen 8086
```

---

### 3. Start Apache and Verify Port

Started the `httpd` service and installed `iproute` to use `ss` for port verification.

```bash
sudo systemctl start httpd
sudo yum install iproute
sudo ss -tlpn | grep httpd
```

Confirmed that Apache was listening on port `8086`.

---

### 4. Set Correct Ownership on Web Root

Verified that `/var/www/html` was owned by `root`. Changed ownership to the `apache` user following the principle of least privilege.

```bash
sudo chown -R apache:apache /var/www/html
```

> The `apache` user is created automatically during `httpd` installation and runs the web server process. It should own the web root — not a regular system user.

---

### 5. Transfer Website Files

Copied the two website directories from the jump host to `/tmp/` on the app server using `scp`, then moved them into the web root.

On the jump host:
```bash
scp -r /home/thor/blog banner@stapp03:/tmp/
scp -r /home/thor/cluster banner@stapp03:/tmp/
```

On the app server:
```bash
sudo mv blog/ cluster/ -t /var/www/html/
```

> `/tmp/` is used as an intermediate location because the `banner` user does not have write access to `/var/www/html` directly. The `-t` flag in `mv` allows specifying multiple sources with a single target directory.

---

### 6. Verify Both Sites

Tested both URLs with `curl` to confirm the sites are served correctly.

```bash
curl http://localhost:8086/blog/
curl http://localhost:8086/cluster/
```