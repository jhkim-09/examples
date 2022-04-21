# DNS 구성
## 시나리오
1. 정방향 조회 DNS 서버 구성
2. 역방향 조회 추가
3. 마스터/슬레이브 구성 (이중화)

## 구성 방법
### 1. 기본 DNS 서버 구성 (정방향/역방향)
#### 1) 패키지 설치
```bash
[root@nobreak ~]# dnf -y install bind bind-utils
```

#### 2) IP주소 및 호스트네임 설정
```bash
[root@nobreak ~]# nmcli con add type ethernet ifname enp0s3 con-name dns ipv4.addresses 10.0.2.10/24 ipv4.gateway 10.0.2.1 ipv4.dns 10.0.2.10 ipv4.method manual
Connection 'dns' (bdb0ffd0-b217-4696-a011-bff92f1cfb2f) successfully added.
[root@nobreak ~]# nmcli con up dns
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
[root@nobreak ~]# hostnamectl set-hostname dns.nobreak.example.com
```

#### 3) 설정파일 설정
```bash
[root@dns ~]# vim /etc/named.conf
```
```vim
acl internal-network {
        10.0.2.0/24;
};

options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { none; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { localhost; internal-network; };

...

zone "nobreak.example.com" IN {
        type master;
        file "nobreak.example.com.zone";
};
```
acl 항목과 zone 항목은 추가
options 항목은 값만 수정

#### 4) 영역파일 구성
```bash
[root@dns ~]# cp /var/named/named.empty /var/named/nobreak.example.com.zone
[root@dns ~]# vim /var/named/nobreak.example.com.zone
```
처음 구성 시 사용할 샘플 제공 (named.empty)
```vim
$TTL 3H
@       IN SOA  nobreak.example.com. root.nobreak.example.com. (
                                        20220415       ; serial
                                        3600        ;Refresh
                                        1800        ;Retry
                                        604800      ;Expire
                                        86400       ;Minimum TTL
        NS      dns.nobreak.example.com.
        A       10.0.2.1
dns     A       10.0.2.10
www     A       10.0.2.20
ftp     A       10.0.2.30
ipa     A       10.0.2.40
db      A       10.0.2.50
nfs     A       10.0.2.60
nas     CNAME   nfs.nobreak.example.com
```
각 서브도메인(dns,www,ftp등) 에 따른 시스템 주소 지정
기본은 A/AAAA 로 IP주소 매핑
마지막처럼 CNAME 항목 설정도 가능 (host 명령어로 확인 시 -t CNAME 옵션 지정 필수)

```bash
[root@dns ~]# ls -l /var/named
total 20
drwxrwx---. 2 named named   23 Apr 15 16:13 data
drwxrwx---. 2 named named   60 Apr 15 16:14 dynamic
-rw-r-----. 1 root  named 2253 Oct 12  2021 named.ca
-rw-r-----. 1 root  named  152 Oct 12  2021 named.empty
-rw-r-----. 1 root  named  152 Oct 12  2021 named.localhost
-rw-r-----. 1 root  named  168 Oct 12  2021 named.loopback
-rw-r-----. 1 root  root   610 Apr 15 16:13 nobreak.example.com.zone
drwxrwx---. 2 named named    6 Oct 12  2021 slaves
[root@dns ~]# chown :named /var/named/nobreak.example.com.zone
[root@dns ~]# chmod 660 /var/named/nobreak.example.com.zone
```
미 설정 시 2(SERVFAIL) 오류메시지 출력
서비스 활성화 후 권한/파일내용 등 변경 시 서비스 재시작 필요

#### 5) 서비스 활성화
```bash
[root@dns ~]# systemctl enable --now named
Created symlink /etc/systemd/system/multi-user.target.wants/named.service → /usr/lib/systemd/system/named.service.
```

#### 6) 방화벽 설정
```bash
[root@dns ~]# firewall-cmd --add-service=dns
success
[root@dns ~]# firewall-cmd --add-service=dns --permanent
success
```

#### 7) 동작 확인
```bash
[root@dns ~]# host -t A dns.nobreak.example.com
Host dns.nobreak.example.com not found: 2(SERVFAIL)
->영역파일 권한 오류. 권한 수정 후 서비스 재시작
[root@dns ~]# host -t A dns.nobreak.example.com
dns.nobreak.example.com has address 10.0.2.10
[root@dns ~]# host dns.nobreak.example.com
dns.nobreak.example.com has address 10.0.2.10
[root@dns ~]# host -t A nfs.nobreak.example.com
nfs.nobreak.example.com has address 10.0.2.60
[root@dns ~]# host -t A nas.nobreak.example.com
Host nas.nobreak.example.com not found: 3(NXDOMAIN)
[root@dns ~]# host nas.nobreak.example.com
Host nas.nobreak.example.com not found: 3(NXDOMAIN)
-> CNAME 검색 시 -t 옵션으로 타입 지정 필수 (기본타입은 A)
[root@dns ~]# host -t CNAME nas.nobreak.example.com
nas.nobreak.example.com is an alias for nfs.nobreak.example.com.nobreak.example.com.
```

#### 8) 역방향조회
/etc/named.conf 파일에 내용 추가
```vim
zone "2.0.10.in-addr.arpa" IN {
        type master;
        file "2.0.10.db";
        allow-update { none; };
};
```
/var/named/ 디렉토리에 해당 파일 생성
```vim
$TTL 86400
@   IN  SOA     nobreak.example.com. root.nobreak.example.com. (
        20220415  ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)
        IN  NS      nobreak.example.com.

1       IN  PTR     nobreak.example.com.
10      IN  PTR     dns.nobreak.example.com.
20      IN  PTR     www.nobreak.example.com.
30      IN  PTR     ftp.nobreak.example.com.
40      IN  PTR     ipa.nobreak.example.com.
50      IN  PTR     db.nobreak.example.com.
60      IN  PTR     nfs.nobreak.example.com.
```
```bash
[root@dns ~]# chown :named /var/named/2.0.10.db
[root@dns ~]# chmod 660 /var/named/2.0.10.db
[root@dns ~]# systemctl restart named
```

#### 9) 역방향조회 확인
```bash
[root@dns ~]# host -t ptr 10.0.2.50
Host 50.2.0.10.in-addr.arpa. not found: 3(NXDOMAIN)
[root@dns ~]# vim /etc/named.conf
[root@dns ~]# systemctl restart named
[root@dns ~]# host -t ptr 10.0.2.50
50.2.0.10.in-addr.arpa domain name pointer db.nobreak.example.com.
```
설정파일에서 zone 항목 설정 오류 시 에러 발생
-> 반드시 IP주소.in-addr.arpa 형태로 이름 지정

### 2. 이중화 구성 (마스터/슬레이브)
#### 1) primary 구성
```vim
zone "test.example.com" IN {
        type master;
        file "test.example.com.zone";
        allow-transfer { 10.0.2.100; }; 
};
```
기존 구성파일에서 원하는 zone 항목마다 마지막 구문 추가 (options 항목에 추가 시 전체)
추가 시 해당 대상에게 영역파일을 전송
각 영역파일에 슬레이브 서버의 주소에 대한 설정도 추가

```bash
[root@dns ~]# systemctl restart named
```

#### 2) secondary 구성
##### IP 주소 / 호스트네임 설정
```bash
[root@nobreak ~]# nmcli con add type ethernet ifname enp0s3 con-name dns ipv4.addresses 10.0.2.100/24 ipv4.gateway 10.0.2.1 ipv4.dns 10.0.2.10 ipv4.method manual
Connection 'dns' (bdb0ffd0-b217-4696-a011-bff92f1cfb2f) successfully added.
[root@nobreak ~]# nmcli con up dns
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
[root@nobreak ~]# hostnamectl set-hostname secondary.nobreak.example.com
```

##### 패키지 설치
```bash
[root@secondary ~]# dnf install bind bind-utils -y
```

##### 서비스 활성화 및 영역파일 확인 (구성 전/후 변화 확인을 위한 부분)
```bash
[root@secondary ~]# ls /var/named
data  dynamic  named.ca  named.empty  named.localhost  named.loopback  slaves
[root@secondary ~]# systemctl enable --now named
Created symlink /etc/systemd/system/multi-user.target.wants/named.service → /usr/lib/systemd/system/named.service.
[root@secondary ~]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-04-18 09:42:25 KST; 5s ago
  Process: 33867 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 33864 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-che>
 Main PID: 33869 (named)
    Tasks: 4 (limit: 23545)
   Memory: 58.0M
   CGroup: /system.slice/named.service
           └─33869 /usr/sbin/named -u named -c /etc/named.conf

Apr 18 09:42:25 secondary.nobreak.example.com named[33869]: network unreachable resolving './DNSKEY/IN': 2001:5>
Apr 18 09:42:25 secondary.nobreak.example.com named[33869]: network unreachable resolving './NS/IN': 2001:500:2>
Apr 18 09:42:25 secondary.nobreak.example.com named[33869]: network unreachable resolving './DNSKEY/IN': 2001:5>
Apr 18 09:42:25 secondary.nobreak.example.com named[33869]: network unreachable resolving './NS/IN': 2001:500:9>
Apr 18 09:42:25 secondary.nobreak.example.com systemd[1]: Started Berkeley Internet Name Domain (DNS).
Apr 18 09:42:25 secondary.nobreak.example.com named[33869]: network unreachable resolving './DNSKEY/IN': 2001:d>
Apr 18 09:42:25 secondary.nobreak.example.com named[33869]: network unreachable resolving './DNSKEY/IN': 2001:5>
Apr 18 09:42:25 secondary.nobreak.example.com named[33869]: network unreachable resolving './DNSKEY/IN': 2001:7>
Apr 18 09:42:25 secondary.nobreak.example.com named[33869]: managed-keys-zone: Key 20326 for zone . acceptance >
Apr 18 09:42:26 secondary.nobreak.example.com named[33869]: resolver priming query complete
[root@secondary ~]# ls /var/named
data  dynamic  named.ca  named.empty  named.localhost  named.loopback  slaves
```
당연히 아직 영역파일은 없음
설정파일 수정을 통해 자동으로 primary 서버에서 복사해 올 예정

##### 설정파일 설정
```bash
[root@secondary ~]# vim /etc/named.conf
```
```vim
zone "nobreak.example.com" IN {
        type slave;
        masters { 10.0.2.10; };
        file "slaves/nobreak.example.com.zone";
};

zone "2.0.10.in-addr.arpa" IN {
        type slave;
        masters { 10.0.2.10; };
        file "slaves/ptr.nobreak.zone";
};
```
영역파일을 저장할 경로(secondary)와 그 파일을 제공해 줄 서버를 지정

```bash
[root@secondary ~]# systemctl restart named
[root@secondary ~]# ls /var/named/
data  dynamic  named.ca  named.empty  named.localhost  named.loopback  slaves
[root@secondary ~]# ls /var/named/slaves/
nobreak.example.com.zone  ptr.nobreak.zone
```
서비스 재시작 시 파일이 복사
복사가 안되면 primary 서버에서 권한/방화벽/파일이름 등을 확인
