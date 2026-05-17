# Tomcat Installation & WAR Deployment

## Overview

Apache Tomcat is a Java Servlet container used to run Java web applications (`.war` files). Tomcat runs as a Java process : `ss -tlnp` will show `java` as the process name, not `tomcat`.

## Prerequisites

Tomcat 9 (Servlet 4.0) requires Java 11:

```bash
yum install java-11-openjdk -y
```

## Installation

```bash
yum install tomcat -y
```

## Directory Structure (yum installation)

| Path | Contents |
|---|---|
| `/etc/tomcat/` | Configuration files (incl. `server.xml`) |
| `/var/lib/tomcat/webapps/` | Web applications |
| `/var/log/tomcat/` | Logs |

## Configuring the Port

Apache Tomcat's `server.xml` is its main configuration file, defining the server's core settings such as the port it listens on, virtual hosts, and how requests are routed to web applications.

Edit the port in `/etc/tomcat/server.xml`:

```bash
grep -n "Connector" /etc/tomcat/server.xml
# Locate the line with port="8080" and adjust as needed
```

## Starting & Enabling the Service

```bash
systemctl start tomcat
systemctl enable tomcat
systemctl status tomcat
```

## Deploying a WAR File

Since `/var/lib/tomcat/webapps/` requires root write access, copy the file via `/tmp/` first:

```bash
# From the jump host
scp ROOT.war user@server:/tmp/
 
# On the app server
sudo cp /tmp/ROOT.war /var/lib/tomcat/webapps/
```

Tomcat detects the `.war` file automatically and deploys it — no restart required.

## Verification

```bash
# Check the port
sudo ss -tlnp | grep 8088
 
# Test the application
curl http://localhost:8088
```

## Useful Commands

```bash
# Find the Tomcat process
ps aux | grep tomcat
 
# Locate the webapps directory
find / -type d -name webapps 2>/dev/null
 
# Follow the logs
tail -f /var/log/tomcat/catalina.out
```