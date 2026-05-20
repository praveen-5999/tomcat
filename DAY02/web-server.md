## web servers vs app servers

| Web Server                            | App Server                                                |
| ------------------------------------- | --------------------------------------------------------- |
| Static content handling               | Business logic + dynamic content                          |
| Serves HTML, CSS, JS, images          | Runs application code (Java, .NET, etc.)                  |
| Examples: Apache HTTP Server, Nginx   | Examples: Apache Tomcat, WildFly (JBoss)                  |
| Works on HTTP/HTTPS (Ports 80/443)    | Works on application ports (8080, 8443, etc.)             |
| Can do reverse proxy & load balancing | Manages application lifecycle, JVM, threads               |
| Lightweight and fast for static files | Heavier, supports sessions, servlets, transactions        |
| Commonly exposed to the internet      | Usually placed behind a web server in 3-tier architecture |


# Apache httpd server (Web Server) - Full Lab on RHEL

Lab Goal

1. Install httpd on RHEL machine
2. Start and enable the service
3. Test default website
4. Understand default path
5. Change DocumentRoot (default path)
6. Change Port number (default 80 -> custom)
7. Update Firewall + AWS Security Group
8. Validate and troubleshoot

## Pre-checks

1. Login to RHEL EC2 instance

2. Check OS details
   cat /etc/redhat-release

3. Check your user
   whoami

4. Switch to root (recommended for lab)
   sudo -i

## Step 1: Check httpd already installed or not

rpm -qa | grep -i httpd

If service exists, check status:
systemctl status httpd

## Step 2: Install Apache httpd

On RHEL, installation depends on how repo is configured (RHSM or mirrors). In most training EC2 RHEL, below works:

yum install httpd -y

If your machine uses dnf:
dnf install httpd -y

After install, confirm package:
httpd -v

## Step 3: Start httpd service

systemctl start httpd
systemctl status httpd

If you want old style command:
service httpd status

## Step 4: Enable httpd (auto-start after reboot)

systemctl enable httpd

## Step 5: Confirm Apache is listening on default port 80

ss -tulnp | grep :80

or
netstat -tulnp | grep :80

## Step 6: Test locally inside server

curl -I [http://localhost](http://localhost)

Expected: HTTP/1.1 200 OK (or 403/404 if no index, but service should respond)

## Step 7: Test from browser (public)

http://<EC2-Public-IP>

Example
[http://65.0.182.70](http://65.0.182.70)

## Step 8: AWS Security Group inbound rule (Very Important)

EC2 Security Group -> Inbound rules

Add rule:
Type: HTTP
Port: 80
Source: My IP (recommended) or 0.0.0.0/0 for public demo

## Step 9: OS Firewall (RHEL firewall)

RHEL uses firewalld commonly.

Check status:
systemctl status firewalld

Allow HTTP:
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

Verify:
firewall-cmd --list-all

## Default Apache Website Path

Default DocumentRoot:
/var/www/html

Default config file:
/etc/httpd/conf/httpd.conf

Other config directory:
/etc/httpd/conf.d/

## Step 10: Create your custom website content (default path test)

echo "KK FUNDA DEVOPS - Apache Web Server" > /var/www/html/index.html

Restart and test:
systemctl restart httpd
curl [http://localhost](http://localhost)

# Change Default Document Root (Path)

Goal: Change from /var/www/html to /data/website

## Step 11: Create new web directory

mkdir -p /data/website
echo "Welcome to KK FUNDA DEVOPS" > /data/website/index.html

Ownership and permissions (Important):
chown -R apache:apache /data/website
chmod -R 755 /data/website

## Step 12: Update Apache config

Open:
vi /etc/httpd/conf/httpd.conf

Find and change:
DocumentRoot "/var/www/html"
to
DocumentRoot "/data/website"

Now you must also allow access to that folder using Directory block.

Add or update:
<Directory "/data/website">
AllowOverride None
Require all granted </Directory>

Why this is required:
If Directory permissions not granted, Apache will show 403 Forbidden

## Step 13: SELinux fix (Most common real issue in RHEL)

Check SELinux:
getenforce

If output is Enforcing, then Apache cannot read /data/website until proper SELinux context is applied.

Apply context:
chcon -R -t httpd_sys_content_t /data/website

Restart:
systemctl restart httpd

Test:
curl [http://localhost](http://localhost)

# Change Apache Port Number (80 -> 8080)

Goal: Run Apache on custom port, example 8080

## Important points before changing port

1. Apache listens on port defined by "Listen"
2. If VirtualHost is configured, it may also need port change
3. OS firewall must allow new port
4. AWS Security Group must allow new port
5. SELinux may block non-standard port unless allowed (Very important in RHEL)

## Step 14: Change Listen Port in httpd.conf

Open:
vi /etc/httpd/conf/httpd.conf

Find:
Listen 80

Change to:
Listen 8080

## Step 15: If VirtualHost exists, update it

Search for VirtualHost:
grep -R "VirtualHost" -n /etc/httpd/

Common example:
<VirtualHost *:80>

Change to:
<VirtualHost *:8080>

If you are not using VirtualHost, skip this step.

## Step 16: SELinux allow Apache to bind to 8080 (Mandatory in Enforcing mode)

First check:
getenforce

If Enforcing, do this:

Check if 8080 already allowed:
semanage port -l | grep http_port_t | head

If semanage command not found, install it:
yum install policycoreutils-python-utils -y

Now allow 8080:
semanage port -a -t http_port_t -p tcp 8080

If it already exists and you want to modify:
semanage port -m -t http_port_t -p tcp 8080

## Step 17: Open OS Firewall for 8080

firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

Verify:
firewall-cmd --list-ports

## Step 18: Restart Apache and validate port

systemctl restart httpd
systemctl status httpd

Check listening:
ss -tulnp | grep 8080

## Step 19: Update AWS Security Group for 8080

Inbound Rule:
Custom TCP
Port: 8080
Source: My IP (recommended)

## Step 20: Test access

Local:
curl [http://localhost:8080](http://localhost:8080)

Browser:
http://<EC2-Public-IP>:8080

# Most Common Errors and Fixes

1. Error: Connection refused

* httpd not running
* firewall not opened
* SG not opened
  Fix:
  systemctl status httpd
  firewall-cmd --list-all
  check SG inbound

2. Error: 403 Forbidden after changing path

* Directory block not updated
* SELinux context missing
  Fix:
  Check httpd.conf Directory block
  chcon -R -t httpd_sys_content_t /data/website

3. Error: Apache fails after changing port

* SELinux blocking non-standard port
  Fix:
  yum install policycoreutils-python-utils -y
  semanage port -a -t http_port_t -p tcp 8080

# Quick Full Demo Commands (Copy-Paste Lab)

sudo -i
yum install httpd -y
systemctl start httpd
systemctl enable httpd

echo "Apache Default Path /var/www/html" > /var/www/html/index.html
systemctl restart httpd

mkdir -p /data/website
echo "KK FUNDA DEVOPS New DocRoot" > /data/website/index.html
chown -R apache:apache /data/website
chmod -R 755 /data/website
chcon -R -t httpd_sys_content_t /data/website

vi /etc/httpd/conf/httpd.conf
(change DocumentRoot and Directory block)
(change Listen 80 to Listen 8080)

yum install policycoreutils-python-utils -y
semanage port -a -t http_port_t -p tcp 8080

firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

systemctl restart httpd
ss -tulnp | grep 8080
curl [http://localhost:8080](http://localhost:8080)
