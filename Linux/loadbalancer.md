# 로드밸런서 (haproxy)
## L7 로드밸런서 (http)
### 사전 설정
같은 네트워크 대역에 웹서버 2대 이상 준비
(가상호스트도 가능)
### 패키지 설치
```bash
[root@nobreak ~]# hostnamectl set-hostname loadbalancer.nobreak.example.com
[root@nobreak ~]# bash
[root@loadbalancer ~]# dnf -y install haproxy
```
### 서비스 설정
```bash
[root@loadbalancer ~]# vim /etc/haproxy/haproxy.cfg
[root@loadbalancer ~]# systemctl enable --now haproxy
Created symlink /etc/systemd/system/multi-user.target.wants/haproxy.service → /usr/lib/systemd/system/haproxy.service.
[root@loadbalancer ~]# firewall-cmd --add-service=http
success
[root@loadbalancer ~]# firewall-cmd --add-service=http --permanent
success
```
/etc/haproxy/haproxy.cfg 내용 추가
```vim
frontend http-in
    bind *:80
    default_backend    backend_servers
    option             forwardfor

backend backend_servers
    balance            roundrobin
    server             web1     192.168.56.11:80 check
    server             web2     192.168.56.12:80 check
```
frontend / backend 이름은 독립적
default_backedn 항목과 backend 이름 통일


### 확인
```bash
[root@loadbalancer ~]# curl 192.168.56.11
host1
[root@loadbalancer ~]# curl 192.168.56.12
host2
[root@loadbalancer ~]# curl 192.168.56.10
host1
[root@loadbalancer ~]# curl 192.168.56.10
host2
```

## L4 로드밸런서
```bash
[root@loadbalancer ~]# setsebool -P haproxy_connect_any on
[root@loadbalancer ~]# firewall-cmd --add-service=mysql
success
[root@loadbalancer ~]# firewall-cmd --add-service=mysql --permanent
success
```
SELinux 동작 시 부울 설정 필요
로드밸런싱 해줄 포트에 대한 방화벽 설정 필요

```vim
defaults
    mode                    tcp
```
defaults 영역에서 mode 만 tcp 로 변경하면 됨
나머지는 L7 때와 동일
ex)
```vim
frontend db
    bind *:3306
    default_backend    backend_db

backend backend_db
    balance            roundrobin
    server             db1      192.168.56.11:3306 check
    server             db2      192.168.56.12:3306 check
#----------------------------------------------------------------------
frontend web
    bind *:80
    default_backend    backend_web

backend backend_web
    balance            roundrobin
    server             web1     192.168.56.11:80 check
    server             web2     192.168.56.12:80 check
```
각 명시하는 이름만 잘 구분