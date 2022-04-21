# Webserver 
## 종류 별 구성
1. Apache
2. Nginx

## 구성 순서
1. 기본 웹서버
2. 가상호스트
3. HTTPS
4. 동적컨텐츠
5. DB 연동

## Apache 서버 구성
### 1. 기본 apache 웹서버 구성
#### 0) 준비
```bash
[root@www ~]# echo "10.0.2.10 www.nobreak.example.com www" >> /etc/hosts
[root@www ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.0.2.10 www.nobreak.example.com www
[root@www ~]# hostname
www.nobreak.example.com
```
DNS 서버가 없을 경우 접속 테스트를 할 시스템에 주소 등록 (IP로 테스트하려면 생략 가능)

#### 1) 패키지 설치
```bash
[root@www ~]# dnf install -y httpd
```

#### 2) 서비스 설정
##### 설정파일 수정 (필수 수정 항목은 없음, ServerName 항목 설정 가능)
```vim
ServerName www.nobreak.example.com:80
```
설정 안해도 상관없음

##### 컨텐츠 파일 생성 (그냥 텍스트파일도 가능)
```html
<html>
<body>
<div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
Basic Page
</div>
</body>
</html>
```

#### 3) 서비스 활성화
```bash
[root@www ~]# systemctl enable --now httpd
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.
```

#### 4) 방화벽 설정
```bash
[root@www ~]# firewall-cmd --add-service=http
success
[root@www ~]# firewall-cmd --add-service=http --permanent
success
```

### 2. 가상호스트 구성
```bash
[root@www ~]# mkdir /var/www/vhost
[root@www ~]# vim /var/www/vhost/index.html
[root@www ~]# vim /etc/httpd/conf.d/00-basic.conf
[root@www ~]# vim /etc/httpd/conf.d/01-vhost.conf
[root@www ~]# systemctl restart httpd
```
별도의 컨텐츠를 제공할 디렉토리와 파일 준비
기본 페이지 및 사용할 각종 이름 별 가상호스트 설정/컨텐츠 생성

```vim
<VirtualHost *:80>
    DocumentRoot /var/www/vhost
    ServerName virtual.nobreak.example.com
</VirtualHost>

<Directory "/var/www/vhost">
    AllowOverride All
    Require all granted
</Directory>
```
DNS 주소에 등록된 이름으로 사용하거나 /etc/hosts 파일에 설정 필요
ServerName 과 DocumentRoot 는 대상 별로 다르게 지정 
Directory 블록은 기본값으로 설정할 땐 생략 가능 (위 값이 기본)

### 3. HTTPS
#### 1) 인증서 구성
##### 개인키 생성
```bash
[root@www ~]# openssl genrsa -aes128 -out server.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
...........+++++
..............................+++++
e is 65537 (0x010001)
Enter pass phrase for server.key:
140452557555520:error:28078065:UI routines:UI_set_result_ex:result too small:crypto/ui/ui_lib.c:905:You must type in 4 to 1023 characters
Enter pass phrase for server.key:
Verifying - Enter pass phrase for server.key:
```
개인키 생성 시 pass phrase 는 4자리 이상으로 설정
```bash
[root@www ~]# openssl rsa -in server.key -out /etc/pki/tls/private/server.key
Enter pass phrase for server.key:
writing RSA key
[root@www ~]# ls -l /etc/pki/tls/private/server.key
-rw-------. 1 root root 1679 Apr 18 14:01 /etc/pki/tls/private/server.key
```
pass phrase 제거
##### 요청서 작성
```bash
[root@www ~]# openssl req -utf8 -new -key /etc/pki/tls/private/server.key -out server.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:kr
State or Province Name (full name) []:seoul
Locality Name (eg, city) [Default City]:seoul
Organization Name (eg, company) [Default Company Ltd]:nobreak
Organizational Unit Name (eg, section) []:test
Common Name (eg, your name or your server's hostname) []:www.nobreak.example.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
##### 인증서 생성
```bash
[root@www ~]# openssl x509 -in server.csr -out /etc/pki/tls/certs/server.crt -req -signkey /etc/pki/tls/private/server.key -days 365
Signature ok
subject=C = kr, ST = seoul, L = seoul, O = nobreak, OU = test, CN = www.nobreak.example.com
Getting Private key
[root@www ~]# ls -l /etc/pki/tls/certs/server.crt
-rw-r--r--. 1 root root 1241 Apr 18 14:05 /etc/pki/tls/certs/server.crt
```
생성 경로는 상관없으니 편의상 현재 작업디렉토리에서 다 만들고 이후 원하는 곳으로 이동 가능

#### 2) 서비스 설정
##### 패키지 설치
```bash
[root@www ~]# dnf install -y mod_ssl
```
##### 방화벽 설정 추가
```bash
[root@www ~]# firewall-cmd --add-service=https
success
[root@www ~]# firewall-cmd --add-service=https --permanent
```
##### 인증서 위치 지정
```bash
[root@www ~]# vim /etc/httpd/conf.d/ssl.conf
[root@www ~]# systemctl restart httpd
```
```vim
SSLCertificateFile /etc/pki/tls/certs/server.crt
...
SSLCertificateKeyFile /etc/pki/tls/private/server.key
```
생성한 파일 이름 및 위치에 맞게 수정 후 서비스 재시작
##### 확인
```bash
[root@www ~]# curl http://localhost
<html>
<body>
<div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
Basic Page
</div>
</body>
</html>
[root@www ~]# curl https://localhost
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```
http / https 둘 다 가능

##### https 리다이렉션 설정
```vim
<VirtualHost *:80>
    DocumentRoot /var/www/html
    ServerName www.nobreak.example.com
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>
```
가상호스트마다 설정값 추가
```bash
[root@www ~]# vim /etc/httpd/conf.d/00-basic.conf
[root@www ~]# systemctl restart httpd
[root@www ~]# curl http://localhost
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="https://localhost/">here</a>.</p>
</body></html>
```

### 4. 동적컨텐츠
#### 1) CGI
```bash
[root@www ~]# vim /var/www/cgi-bin/index.cgi
[root@www ~]# ls -l /var/www/cgi-bin/index.cgi
-rw-r--r--. 1 root root 257 Apr 18 14:36 /var/www/cgi-bin/index.cgi
[root@www ~]# chmod 755 /var/www/cgi-bin/index.cgi
[root@www ~]# ls -l /var/www/cgi-bin/index.cgi
-rwxr-xr-x. 1 root root 257 Apr 18 14:36 /var/www/cgi-bin/index.cgi
[root@www ~]# curl localhost/cgi-bin/index.cgi
<html>
<body>
<div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
CGI Script Test Page
</div>
</body>
</html>
```
cgi 파일 구성
```vim 
#!/usr/libexec/platform-python

print("Content-type: text/html\n")
print("<html>\n<body>")
print("<div style=\"width: 100%; font-size: 40px; font-weight: bold; text-align: center;\">")
print("CGI Script Test Page")
print("</div>")
print("</body>\n</html>")
```

#### 2) PHP
##### php 설치
```bash
[root@www ~]# dnf module list php
Last metadata expiration check: 2:05:25 ago on Mon 18 Apr 2022 12:56:25 PM KST.
Rocky Linux 8 - AppStream
Name              Stream               Profiles                               Summary
php               7.2 [d]              common [d], devel, minimal             PHP scripting language
php               7.3                  common [d], devel, minimal             PHP scripting language
php               7.4                  common [d], devel, minimal             PHP scripting language

Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled
[root@www ~]# dnf module install php
Last metadata expiration check: 2:05:33 ago on Mon 18 Apr 2022 12:56:25 PM KST.
Dependencies resolved.
================================================================================================================
 Package                  Architecture   Version                                        Repository         Size
================================================================================================================
Installing group/module packages:
 php-cli                  x86_64         7.2.24-1.module+el8.4.0+413+c9202dda           appstream         3.1 M
 php-common               x86_64         7.2.24-1.module+el8.4.0+413+c9202dda           appstream         660 k
 php-fpm                  x86_64         7.2.24-1.module+el8.4.0+413+c9202dda           appstream         1.6 M
 php-json                 x86_64         7.2.24-1.module+el8.4.0+413+c9202dda           appstream          72 k
 php-mbstring             x86_64         7.2.24-1.module+el8.4.0+413+c9202dda           appstream         579 k
 php-xml                  x86_64         7.2.24-1.module+el8.4.0+413+c9202dda           appstream         187 k
Installing dependencies:
 nginx-filesystem         noarch         1:1.14.1-9.module+el8.4.0+542+81547229         appstream          23 k
Installing module profiles:
 php/common
Enabling module streams:
 nginx                                   1.14
 php                                     7.2

Transaction Summary
================================================================================================================
Install  7 Packages

Total download size: 6.2 M
Installed size: 23 M
Is this ok [y/N]: y
```
PHP 의 경우 7.2 ~ 7.4 버전에 대해 각각 모듈지원 (appstream)
원하는 대상을 다운로드 / 활성화 해야하며 변경 시 모듈 초기화 필요 (dnf module reset php)
예시
```bash
[root@www ~]# dnf module -y install php:7.4
Last metadata expiration check: 2:16:10 ago on Mon 18 Apr 2022 12:56:25 PM KST.
Dependencies resolved.
The operation would result in switching of module 'php' stream '7.2' to stream '7.4'
Error: It is not possible to switch enabled streams of a module unless explicitly enabled via configuration option module_stream_switch.
It is recommended to rather remove all installed content from the module, and reset the module using 'dnf module reset <module_name>' command. After you reset the module, you can install the other stream.
[root@www ~]# dnf module -y reset php
Last metadata expiration check: 2:16:27 ago on Mon 18 Apr 2022 12:56:25 PM KST.
Dependencies resolved.
================================================================================================================
 Package                   Architecture             Version                     Repository                 Size
================================================================================================================
Disabling module profiles:
 php/common
Resetting modules:
 php

Transaction Summary
================================================================================================================

Complete!
[root@www ~]# dnf module -y install php:7.4
[root@www ~]# dnf module enable php:7.4
Last metadata expiration check: 2:18:23 ago on Mon 18 Apr 2022 12:56:25 PM KST.
Dependencies resolved.
Nothing to do.
Complete!
```
```bash
[root@www ~]# cat test.php
<?php echo "PHP Test Page \n"; ?>
[root@www ~]# php test.php
PHP Test Page
```
##### 웹서버에서 사용
```bash
[root@www ~]# systemctl status php-fpm
● php-fpm.service - The PHP FastCGI Process Manager
   Loaded: loaded (/usr/lib/systemd/system/php-fpm.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
[root@www ~]# systemctl restart httpd
[root@www ~]# systemctl status php-fpm
● php-fpm.service - The PHP FastCGI Process Manager
   Loaded: loaded (/usr/lib/systemd/system/php-fpm.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-04-18 15:25:55 KST; 3s ago
 Main PID: 41247 (php-fpm)
   Status: "Ready to handle connections"
    Tasks: 6 (limit: 23545)
   Memory: 9.8M
   CGroup: /system.slice/php-fpm.service
           ├─41247 php-fpm: master process (/etc/php-fpm.conf)
           ├─41248 php-fpm: pool www
           ├─41249 php-fpm: pool www
           ├─41250 php-fpm: pool www
           ├─41251 php-fpm: pool www
           └─41252 php-fpm: pool www

Apr 18 15:25:55 www.nobreak.example.com systemd[1]: Starting The PHP FastCGI Process Manager...
Apr 18 15:25:55 www.nobreak.example.com systemd[1]: Started The PHP FastCGI Process Manager.
```
php-fpm 서비스는 php를 설치하고 httpd 서비스를 활성화하면 자동으로 실행됨
지금처럼 httpd 동작 중 php 설치 시 서비스 재시작 필요

##### 컨텐츠 파일 생성 및 확인
```bash
[root@www ~]# vim /var/www/html/index.php
[root@www ~]# cat /var/www/html/index.php
<?php echo "PHP Test Page \n"; ?>
[root@www ~]# php /var/www/html/index.php
PHP web Test Page
[root@www ~]# curl localhost/index.php
PHP web Test Page
```

### 5. DB연동 (PHP/Mariadb)
#### 기본준비
웹서버 / 데이터베이스 / PHP 설치 및 동작
```bash
[root@www ~]# httpd -v
Server version: Apache/2.4.37 (rocky)
Server built:   Mar 24 2022 17:33:25
[root@www ~]# php -v
PHP 7.2.24 (cli) (built: Oct 22 2019 08:28:36) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
[root@www ~]# mysql --version
mysql  Ver 15.1 Distrib 10.3.28-MariaDB, for Linux (x86_64) using readline 5.1
```

#### 추가 패키지 설치
```bash
[root@www ~]# dnf install php-mysqlnd
```

#### php 파일 작성
```php
<?php
$mysqli = new mysqli("localhost", "root", "1", "test");
$mysqli->query("CREATE TABLE webdb10 (id int, name varchar(255))" );
$mysqli->query("INSERT INTO webdb10 (id,name) VALUES (1,'php')" );
$result = $mysqli->query("SELECT id FROM webdb10 WHERE name LIKE 'php'");
printf("PHP users id is : %d \n", $result);
?>
```
순서대로
연결 설정 ( DB위치 : localhost , 사용자 : root , 패스워드 : 1 , DB이름 : test )
테이블 생성
테이블에 데이터 입력
데이터 확인 (확인 시 SELECT 구문만 쓰면 출력값이 없어서 printf 함수로 결과값 출력)
만약 printf 함수가 없으면 아무것도 안뜸

## WordPress 구성
### 구성 순서
1. PHP 설치
2. httpd 설치
3. mariadb 설치
4. WP 구성
### 구성방법
#### 1) 추가 패키지 설치
```bash
[root@www ~]# dnf -y install epel-release
[root@www ~]# dnf --enablerepo=epel -y install php-pear php-mbstring php-pdo php-gd php-mysqlnd php-IDNA_Convert php-enchant enchant hunspell
```
추가적인 PHP 모듈 관련 패키지 설치 (epel-release 레포지토리 활성화 후 설치)

#### 2) 설정파일 설정
```bash
[root@www ~]# vi /etc/php-fpm.d/www.conf
[root@www ~]# systemctl restart php-fpm
```
필요한 항목이 있으면 파일에 추가
```vim
php_value[max_execution_time] = 600
php_value[memory_limit] = 2G
php_value[post_max_size] = 2G
php_value[upload_max_filesize] = 2G
php_value[max_input_time] = 600
php_value[max_input_vars] = 2000
php_value[date.timezone] = Asia/Tokyo
```

#### 3) DB 구성
```bash
[root@www ~]# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8
Server version: 10.3.28-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE DATABASE wordpress;
Query OK, 1 row affected (0.001 sec)

MariaDB [(none)]> GRANT all privileges ON wordpress.* to wordpress@'*' identified by 'password';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> FLUSH privileges;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> exit
Bye
```
워드프레스에서 사용할 데이터베이스와 사용자에 대한 설정

#### 4) WP 설치
```bash
[root@www ~]# wget https://wordpress.org/latest.tar.gz
--2022-04-19 11:06:58--  https://wordpress.org/latest.tar.gz
Resolving wordpress.org (wordpress.org)... 198.143.164.252
Connecting to wordpress.org (wordpress.org)|198.143.164.252|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18725197 (18M) [application/octet-stream]
Saving to: ‘latest.tar.gz’

latest.tar.gz               100%[===========================================>]  17.86M  8.20MB/s    in 2.2s

2022-04-19 11:07:01 (8.20 MB/s) - ‘latest.tar.gz’ saved [18725197/18725197]

[root@www ~]# ls -l latest.tar.gz
-rw-r--r--. 1 root root 18725197 Apr  6 04:14 latest.tar.gz
[root@www ~]# file latest.tar.gz
latest.tar.gz: gzip compressed data, last modified: Tue Apr  5 19:14:06 2022, from Unix, original size 60794880
[root@www ~]# tar zxvf latest.tar.gz -C /var/www/
[root@www ~]# chown -R apache. /var/www/wordpress
[root@www ~]# vim /etc/httpd/conf.d/wordpress.conf
[root@www ~]# systemctl restart httpd
```
/etc/httpd/conf.d/wordpress.conf
```vim 
Timeout 600
ProxyTimeout 600

Alias /wordpress "/var/www/wordpress/"
<Directory "/var/www/wordpress">
    Options FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```
컨텐츠파일 다운로드 및 해당 디렉토리에 대한 설정 추가 (/var/www/html 도 가능)

#### 5) SELinux 설정
```bash
[root@www ~]# setsebool -P httpd_can_network_connect on
[root@www ~]# setsebool -P domain_can_mmap_files on
[root@www ~]# setsebool -P httpd_unified on
```
로컬DB 시 필요없음

## NginX 구성
### 1. 기본구성
#### 1) 패키지 설치
```bash
[root@www ~]# dnf -y install nginx
```
#### 2) 설정파일 수정
/etc/nginx/nginx.conf
```vim
server_name www.nobreak.example.com;
```
접속 시 사용할 이름주소 설정

#### 3) 서비스 활성화
```bash
[root@www ~]# systemctl enable --now nginx
```
#### 4) 방화벽 설정
```bash
[root@www ~]# firewall-cmd --add-service=http
success
[root@www ~]# firewall-cmd --add-service=http --permanent
```

### 2. 가상호스트
#### 1) 설정파일 추가
/etc/nginx/conf.d/01-vhost.conf
```vim
server {
    listen       80;
    server_name  vhost.nobreak.example.com;

    location / {
        root   /usr/share/nginx/vhost;
        index  index.html index.htm;
    }
}
```
```bash
[root@www ~]# mkdir /usr/share/nginx/vhost
[root@www ~]# systemctl restart nginx
```
#### 2) 컨텐츠 추가 생성
/usr/share/nginx/vhost/index.html
```html
<html>
<body>
<div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
Nginx Virtual Host Test Page
</div>
</body>
</html>
```

### 3. HTTPS
#### apache 때와 마찬가지로 인증서 준비는 기본
#### 설정파일 설정
/etc/nginx/conf.d/ssl.conf
```vim
server {
    listen       443 ssl http2 default_server;
    listen       [::]:443 ssl http2 default_server;
    server_name  www.nobreak.example.com;
    root         /usr/share/nginx/html;

    ssl_certificate "인증서파일경로";
    ssl_certificate_key "키파일경로";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers PROFILE=SYSTEM;
    ssl_prefer_server_ciphers on;

    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```
```bash
[root@www ~]# systemctl restart nginx
```
#### 리다이렉트 설정
/etc/nginx/nginx.conf
```vim
    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        return       301 https://$host$request_uri;
        server_name  www.nobreak.example.com;
        root         /usr/share/nginx/html;
```
```bash
[root@www ~]# systemctl restart nginx
[root@www ~]# firewall-cmd --add-service=https
success
[root@www ~]# firewall-cmd --runtime-to-permanent
success
```

### 4. CGI / PHP
#### CGI
##### 패키지 설치
```bash
[root@www ~]# dnf -y install epel-release
[root@www ~]# dnf --enablerepo=epel -y install fcgiwrap
```
##### nginx 설정
/etc/nginx/fcgiwrap.conf 파일 생성
```vim
location /cgi-bin/ {
    gzip off;
    root  /usr/share/nginx;
    fastcgi_pass  unix:/var/run/fcgiwrap.socket;
    include /etc/nginx/fastcgi_params;
    fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
}
```
/etc/nginx/nginx.conf 파일의 server 블록에 내용 추가 (SSL 설정 시 해당 항목에 추가)
```vim
include fcgiwrap.conf;
```
```bash
[root@www ~]# mkdir /usr/share/nginx/cgi-bin
[root@www ~]# systemctl restart nginx
```
##### systemd 서비스 생성 (fcgiwrap)
```bash
[root@www ~]# vim /usr/lib/systemd/system/fcgiwrap.service
[root@www ~]# vim /usr/lib/systemd/system/fcgiwrap.socket
[root@www ~]# systemctl enable --now fcgiwrap
Created symlink /etc/systemd/system/sockets.target.wants/fcgiwrap.socket → /usr/lib/systemd/system/fcgiwrap.socket.
```
/usr/lib/systemd/system/fcgiwrap.service
```vim
[Unit]
Description=Simple CGI Server
After=nss-user-lookup.target
Requires=fcgiwrap.socket

[Service]
EnvironmentFile=/etc/sysconfig/fcgiwrap
ExecStart=/usr/sbin/fcgiwrap ${DAEMON_OPTS} -c ${DAEMON_PROCS}
User=nginx
Group=nginx

[Install]
Also=fcgiwrap.socket
```
/usr/lib/systemd/system/fcgiwrap.socket
```vim
[Unit]
Description=fcgiwrap Socket

[Socket]
ListenStream=/run/fcgiwrap.socket

[Install]
WantedBy=sockets.target
```
##### SELinux 설정
```bash
[root@www ~]# vim nginx-server.te
[root@www ~]# checkmodule -m -M -o nginx-server.mod nginx-server.te
[root@www ~]# semodule_package --outfile nginx-server.pp --module nginx-server.mod
[root@www ~]# semodule -i nginx-server.pp
```
nginx-server.te
```
module nginx-server 1.0;

require {
        type httpd_t;
        type var_run_t;
        class sock_file write;
}

#============= httpd_t ==============
allow httpd_t var_run_t:sock_file write;
```

#### PHP
##### PHP 설치
```bash
[root@www ~]# dnf module -y reset php
Last metadata expiration check: 0:09:22 ago on Wed 20 Apr 2022 12:44:39 PM KST.
Dependencies resolved.
Nothing to do.
Complete!
[root@www ~]# dnf module -y enable php:7.4
Last metadata expiration check: 0:09:37 ago on Wed 20 Apr 2022 12:44:39 PM KST.
Dependencies resolved.
================================================================================================================
 Package                   Architecture             Version                     Repository                 Size
================================================================================================================
Enabling module streams:
 httpd                                              2.4
 php                                                7.4

Transaction Summary
================================================================================================================
Complete!
[root@www ~]# dnf module -y install php:7.4/common
[root@www ~]# systemctl restart nginx
[root@www ~]#  systemctl status php-fpm
● php-fpm.service - The PHP FastCGI Process Manager
   Loaded: loaded (/usr/lib/systemd/system/php-fpm.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-04-20 12:55:54 KST; 11s ago
 Main PID: 3784 (php-fpm)
   Status: "Processes active: 0, idle: 5, Requests: 0, slow: 0, Traffic: 0req/sec"
    Tasks: 6 (limit: 23545)
   Memory: 9.7M
   CGroup: /system.slice/php-fpm.service
           ├─3784 php-fpm: master process (/etc/php-fpm.conf)
           ├─3785 php-fpm: pool www
           ├─3786 php-fpm: pool www
           ├─3787 php-fpm: pool www
           ├─3788 php-fpm: pool www
           └─3789 php-fpm: pool www

Apr 20 12:55:54 www.nobreak.example.com systemd[1]: Starting The PHP FastCGI Process Manager...
Apr 20 12:55:54 www.nobreak.example.com systemd[1]: Started The PHP FastCGI Process Manager.
```
##### 컨텐츠 생성
```bash
[root@www ~]# echo '<?php phpinfo(); ?>' > /usr/share/nginx/html/index.php
```
