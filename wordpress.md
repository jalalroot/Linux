# Install Wordpress on Linux

Deploying WordPress on Rocky Linux v10.2 AMD/Intel (x86_64) with nginx version ? and mariadb version ? Along with an SSL/TLS certificate and on port https/443.

---

## 📌 Architecture Overview



Our topology consists of two Rocky Linux machines of the mentioned version: one machine for installing Nginx and WordPress itself, and the other for installing the database, and finally connecting the two together.
```text
                        Web Browser
                    +--------------------+
                    |IP: 192.168.114.130 |    
                    |      SUCCESS       |   
                    |     WORDPRESS      |
                    +--------------------+
                            |
                            |
                            |
                            |
                            |
nginx server(wordpress)     |        database server
+-------------------+       |        +---------------------+
| Rocky 10.2 server |       |        |  Rocky 10.2 server  |
|  -nginx  v?       |       |        |  -MariaDB  v?       | 
|  -IP=114.130      |-------|--------|  -IP=114.131        |
|  -port 443        |                |  -SElinux ✅       |
|  -SElinux ✅     |                |  -firewall ✅       | 
|  -firewall ✅    |                |                     |    
+-------------------+               +----------------------+
```
## ⚙️ Prerequisites
We did this project on VMware Workstation v26.0.1. Both of our machines have 2 CPU cores and 2 GB of RAM. Both machines have internet access (NAT) with a single network interface and both are in same network, and we allocated 20 GB of disk space to each of them. 
#### OS : Rocky linux 10.2 AMD/intel(X86_64) (DVD ISO)
#### Hardware Resources : 2 vCORE CPU , 2G RAM , 20G Disk
#### Network Adapter(s):
    ens33 = 192.168.X.X/24

## 🚀 Step-by-Step Installation
Our focus is not very much on how to partition the disk, so we leave it to Linux to do automatically. We assign an IP address to the network interface, choose an appropriate hostname, and let it install.
we can do it with ``` nmtui ``` .
### 🖥️ after installing linux on Nginx machine: 
---
make sure your both machines are updated: 
```text 
sudo dnf upgrade -y #This command updates the packages inside your machines.
```
As we mentioned, we need to install Nginx on the machine we designated for Nginx and WordPress. 
```text
sudo dnf install nginx #installing latest version of nginx
```
## 🌐 so after install now we should start service : 

```text 
sudo systemctl enable --now nginx #start and enable nginx
```
## 🔥 firewall 
Now, we need to configure the firewall to allow port 443 (https). Since we are using a Red Hat-based distribution (Rocky Linux), we will use the ```firewall-cmd``` command to manage firewalld.
```text 
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --permanent --zone=public --add-service=http
 #for networks that we do not fully trust, such as the internet.
sudo firewall-cmd --reload
```
## 🐘 install and config php
```text 
sudo dnf install php #install primary php package
``` 
One of the packages that is critical for WordPress to function is php-fpm(FastCGI Process Manager). This package passes requests to PHP for the web server, which itself cannot open WordPress pages that require PHP.
```text
sudo dnf install php-fpm 
```
Another critical and important package for being able to display images in WordPress is `php-gd` (Graphical Draw). Without it, even the admin management panel might not load properly.
```text
sudo dnf install php-gd
```
Another critical package is `php-curl` (PHP Client URL), which allows WordPress to communicate with external web servers and APIs, and is essential for WordPress self-updates, plugins, or WooCommerce payment gateways. Of course, if security is very important to you, it can be disabled, but you would have to say goodbye to certain plugins and also bring and install update files manually.
```text
sudo dnf install php-curl
```
If you use languages other than English on your site, then you definitely need to use a specific package called `php-mbstring`.
```text 
sudo dnf install php-mbstring
```
Now we need to enable the php-fpm service:
```text
sudo systemctl enable --now php-fpm
```
## ⚠️ Important
**The PHP-FPM service is configured to run under the apache user and group (user = apache and group = apache). However, our web server is Nginx, and its processes run under the nginx user and group.**

**As a result, when Nginx tries to pass a request to PHP-FPM, it encounters a “Permission denied” error. PHP-FPM is listening on the socket file /run/php-fpm/www.sock, which is owned by apache:apache. Nginx cannot access the socket, so the user sees a 502 Bad Gateway error.**

**To fix this, we need to change the user and group ownership of the socket configuration file to nginx.**
```text
/etc/php-fpm.d/www.conf 
user = nginx
group = nginx
```
and after that restart php-fpm and nginx: 
```text
systemctl restart php-fpm.service nginx.service
```
## 📝 TEST 
Now, to create a test page and verify that PHP is installed correctly and is successfully communicating with Nginx, add the following code to the file below:
```text 
/usr/share/nginx/html/info.php 

#This path is the default root directory for web files in many Linux distributions  This means when someone visits your server’s IP address or domain, Nginx reads the files from this folder and displays them to the user.
``` 
simple code for testing : 
```text
<?php
phpinfo()
?>
```
## 🪪 enable 443 and SSL/TLS
We create a self-signed certificate using OpenSSL.
install openssl
```text
sudo dnf install openssl
```
After, create a directory to store the key and certificate:
```text 
sudo mkdir -p /etc/nginx/ssl
```
Now we generate the certificate and private key with the following command (setting its validity to 365 days):
```text
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/nginx/ssl/nginx-selfsigned.key \
-out /etc/nginx/ssl/nginx-selfsigned.crt
```
Note: When running this command, it asks a few questions like your country, city, etc., which you can just press Enter to skip. However, in the **Common Name (e.g. server FQDN or YOUR name)** field, make sure to enter the server's actual IP address, which is `192.168.114.130`.
## 🌎 installing wordpress
We install WordPress inside /opt.
```text
cd /opt
sudo curl -O https://wordpress.org/latest.tar.gz
sudo tar -zxvf latest.tar.gz
```
Nginx is not yet configured to point to /opt/wordpress. You need to create a configuration file for it:
```text 
/etc/nginx/conf.d/wordpress.conf
```
---
```text
server {
    listen 443 ssl;
    server_name 192.168.114.130; 
    root /opt/wordpress;
    index index.php;

    ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```
Before restarting the service, make sure to test the configuration to ensure there are no errors:
```text 
sudo nginx -t # syntax is ok -> now you can restart nginx.service
```
Usually, a small block is also created for port 80 so that if a user accesses via HTTP, they are automatically redirected to HTTPS:
```text
server {
    listen 80;
    server_name 192.168.114.130;
    return 301 https://$host$request_uri;
}
```
Now restart Nginx

Changes recursive file and directory ownership to the Nginx user so the web server can manage them: 
```text 
sudo chown -R nginx:nginx /opt/wordpress
```
Sets directory permissions to 755, allowing the owner full access and others read/execute access:
```text
sudo find /opt/wordpress -type d -exec chmod 755 {} \;
```
Sets file permissions to 644, allowing the owner read/write access and others read-only access:
```text
sudo find /opt/wordpress -type f -exec chmod 644 {} \;
```
Applies the SELinux web content context recursively so Nginx is allowed to read the files:
```text
sudo chcon -t httpd_sys_content_t /opt/wordpress -R
```
Grants SELinux read-write context to the wp-content directory for uploads and modifications:
```text
sudo chcon -t httpd_sys_rw_content_t /opt/wordpress/wp-content -R
```
Configures SELinux to permanently allow the web server to establish network connections to a remote database server:
```text
sudo setsebool -P httpd_can_network_connect_db 1
```
## 🗃️ Now on Database Machine 
first install mariadb , These two packages are completely vital and required; if they are missing, the work cannot be done. `mariadb-server` is the core database engine that handles all operations, and `mariadb` (the client package) enables terminal-based commands for the administrator, which is also necessary.
```text
sudo yum install mariadb mariadb-server
```
now securing MariaDB :
```text
sudo mysql_secure_installation
```
We configure the database to listen on the network interface with the IP address 192.168.114.131, allowing other servers within the network to connect to it:
```text
/etc/my.cnf.d/mariadb-server.cnf

bind-address = 192.168.114.131
```
and after restart mariadb

Creating a database user and granting access privileges to the web server’s IP address:
```text
sudo mariadb -u root -p
```
create database :
```text
CREATE DATABASE app_db;
```
Creating a database user with host-based access control: 
```text
CREATE USER 'app_user'@'192.168.114.130' IDENTIFIED BY 'Amjl@1384';
```
Granting access to the database:
```text
GRANTE ALL PRIVILEGES ON app_db.* TO 'app_user'@'192.168.114.130';
```
Applying the changes :
```text
FLUSH PRIVILEGES;
EXIT;
```
**Now we need to open port `3306` on the firewall for the machine at `192.168.114.130` to connect to MariaDB through this port.**
```text
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.114.130" port port="3306" protocol="tcp" accept'
```
and now ```firewall-cmd --reload```

---
now we should connection test between nginx server and mariadb server .

## 🚩 The final steps for testing connectivity on the Nginx server.
install MariaDB client : 
```text
sudo dnf install mariadb
```
and run this command in nginx server :
```text
mariadb -h 192.168.114.131 -u app_user -p
```

If you can successfully log in to the MariaDB console, it confirms that the installation and configuration were successful


# Now you can enter the Nginx server's IP address in your browser to see if WordPress opens correctly for you or not. 
![alt text](image-1.png)
