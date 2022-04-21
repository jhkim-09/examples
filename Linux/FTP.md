# FTP 서버 구성 예제

## 1. FTP 서버 구성 
### 1) 패키지 설치
```bash
[root@ftp.nobreak.server.com ~]# dnf -y install vsftpd
```

### 2) 설정파일 변경
```bash
[root@ftp.nobreak.server.com ~]]# vi /etc/vsftpd/vsftpd.conf
# line 12 : make sure value is [NO] (no anonymous)
anonymous_enable=NO
# line 82,83 : uncomment ( allow ascii mode )
ascii_upload_enable=YES
ascii_download_enable=YES
# line 100,101 : uncomment ( enable chroot )
chroot_local_user=YES
chroot_list_enable=YES
# line 103 : uncomment ( chroot list file )
chroot_list_file=/etc/vsftpd/chroot_list
# line 109 : uncomment
ls_recurse_enable=YES
# line 114 : set YES if listen only IPv4
# if listen both IPv4 and IPv6, set NO
listen=NO
# line 123 : set NO if not listen IPv6
# if listen both IPv4 and IPv6, set YES
listen_ipv6=YES
# add to the end
# specify root directory
# if not specify, users' home directory become FTP home directory
local_root=public_html
# use local time
use_localtime=YES
# turn off for seccomp filter (if cannot login, add this line)
seccomp_sandbox=NO
```
### 3) 유저 생성 및 설정
testuser 생성
```bash
[root@ftp.nobreak.server.com ~]# useradd testuser
[root@ftp.nobreak.server.com ~]# passwd testuser

[root@ftp.nobreak.server.com ~]# vi /etc/vsftpd/chroot_list
# add users you allow to move over their home directory
testuser
```

### 4) 서비스 설정
```bash
[root@ftp.nobreak.server.com ~]# systemctl enable --now vsftpd
```

### 5) SELinux 설정
```bash
[root@ftp.nobreak.server.com ~]# setsebool -P ftpd_full_access on
```

### 6) 방화벽 설정
```bash
[root@ftp.nobreak.server.com ~]# firewall-cmd --add-service=ftp --permanent
success
[root@ftp.nobreak.server.com ~]# firewall-cmd --reload
success
```

## 2. FTP 클라이언트 연결
### 1) 패키지 설치
```bash
[root@ftp.nobreak.client.com ~]# dnf -y install lftp
```

### 2) FTP 로그인
```bash
# lftp [option] [hostname]
[rocky@ftp.nobreak.client.com ~]$ lftp -u cent www.srv.world
Password:     # login user password
lftp testuser@ftp.nobreak.server.com:~>
```

FTP 서버 현재위치 확인
```bash
lftp testuser@ftp.nobreak.server.com:~> pwd
ftp://testuser@ftp.nobreak.server.com

# show current directory on localhost
lftp testuser@ftp.nobreak.server.com:~> !pwd
/home/redhat
```

FTP 서버 디렉토리 파일 확인
```bash
lftp testuser@ftp.nobreak.server.com:~> ls
drwxr-xr-x    2 1000     1000           23 Mar 15 01:33 public_html
-rw-r--r--    1 1000     1000          399 Mar 15 16:32 test.py

# show current file on localhost
lftp testuser@ftp.nobreak.server.com:~> !ls -l
total 12
-rw-rw-r-- 1 redhat redhat 10 Mar 15 14:30 redhat.txt
-rw-rw-r-- 1 redhat redhat 10 Mar 15 14:59 test2.txt
-rw-rw-r-- 1 redhat redhat 10 Mar 15 14:59 test.txt
```

디렉토리 이동
```bash
lftp testuser@ftp.nobreak.server.com:~> cd public_html
lftp testuser@ftp.nobreak.server.com:~/public_html> pwd
ftp://testuser@ftp.nobreak.server.com/%2Fhome/cent/public_html
```

### 3) FTP 서버에 파일 업로드
```bash
# [-a] means ascii mode ( default is binary mode )
lftp testuser@ftp.nobreak.server.com:~> put -a redhat.txt
22 bytes transferred
Total 2 files transferred
lftp testuser@ftp.nobreak.server.com:~> ls
drwxr-xr-x    2 1000     1000           23 Mar 15 01:33 public_html
-rw-r--r--    1 1000     1000           10 Mar 15 17:01 redhat.txt
-rw-r--r--    1 1000     1000          399 Mar 15 16:32 test.py
-rw-r--r--    1 1000     1000           10 Mar 15 17:01 test.txt

lftp testuser@ftp.nobreak.server.com:~> mput -a test.txt test2.txt
22 bytes transferred
Total 2 files transferred
lftp testuser@ftp.nobreak.server.com:~> ls
drwxr-xr-x    2 1000     1000           23 Mar 15 01:33 public_html
-rw-r--r--    1 1000     1000          399 Mar 15 16:32 test.py
-rw-r--r--    1 1000     1000           10 Mar 15 17:06 test.txt
-rw-r--r--    1 1000     1000           10 Mar 15 17:06 test2.txt
```

### 4) FTP 서버에 파일 다운로드
권한 설정
```bash
# set permission to overwite files on localhost when using [get/mget]
lftp testuser@ftp.nobreak.server.com:~> set xfer:clobber on 
```

localhost에 다운로드
```bash
# [-a] means ascii mode ( default is binary mode )
lftp testuser@ftp.nobreak.server.com:~> get -a test.py
416 bytes transferred

lftp testuser@ftp.nobreak.server.com:~> mget -a test.txt test2.txt
20 bytes transferred
Total 2 files transferred
```
### 5) FTP 서버 파일 디렉토리 생성 및 삭제
FTP 서버 디렉토리 만들기
```bash
lftp testuser@ftp.nobreak.server.com:~> mkdir testdir
mkdir ok, `testdir' created
lftp testuser@ftp.nobreak.server.com:~> ls
drwxr-xr-x    2 1000     1000           23 Mar 15 01:33 public_html
-rw-r--r--    1 1000     1000          399 Mar 15 16:32 test.py
-rw-r--r--    1 1000     1000           10 Mar 15 17:06 test.txt
-rw-r--r--    1 1000     1000           10 Mar 15 17:06 test2.txt
drwxr-xr-x    2 1000     1000            6 Mar 15 17:16 testdir
226 Directory send OK.
```

FTP 서버 디렉토리 삭제
```bash
lftp testuser@ftp.nobreak.server.com:~> rmdir testdir
rmdir ok, `testdir' removed
lftp testuser@ftp.nobreak.server.com:~> ls
drwxr-xr-x    2 1000     1000           23 Mar 15 01:33 public_html
-rw-r--r--    1 1000     1000          399 Mar 15 16:32 test.py
-rw-r--r--    1 1000     1000           10 Mar 15 17:06 test.txt
-rw-r--r--    1 1000     1000           10 Mar 15 17:06 test2.txt
```

FTP 서버 파일 삭제
```bash
lftp testuser@ftp.nobreak.server.com:~> rm test2.txt
rm ok, `test2.txt' removed
lftp testuser@ftp.nobreak.server.com:~> ls
drwxr-xr-x    2 1000     1000           23 Mar 15 01:33 public_html
-rw-r--r--    1 1000     1000          399 Mar 15 16:32 test.py
-rw-r--r--    1 1000     1000           10 Mar 15 17:06 test.txt

lftp testuser@ftp.nobreak.server.com:~> mrm redhat.txt test.txt
rm ok, 2 files removed
lftp testuser@ftp.nobreak.server.com:~> ls
drwxr-xr-x    2 1000     1000           23 Mar 15 01:33 public_html
```

### 6) FTP 서버 명령어 실행
![command]를 사용하여 명령어 실행
```bash
lftp testuser@ftp.nobreak.server.com:~> !cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
.....
.....
cent:x:1001:1001::/home/cent:/bin/bash
```
종료
```bash
# exit
lftp testuser@ftp.nobreak.server.com:~> quit
221 Goodbye.
```
