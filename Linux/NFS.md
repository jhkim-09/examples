# NFS 서버 구성 예제

## 0. NFS 서버 구성
### 1) 패키지 설치
7버전 이상은 기본 설치 되어있다.
```bash
[root@nfs.server.com ~]# dnf -y install nfs-utils
```

### 2) 설정파일 생성 및 변경
domain 설정
```bash
[root@nfs.server.com ~]# vi /etc/idmapd.conf
# line 5 : uncomment and change to your domain name
Domain = server.com
```

내보내기(export) 설정
```bash
[root@nfs.server.com ~]# vi /etc/exports
# create new
# for example, set [/home/nfsshare] as NFS share
/home/nfsshare 10.0.0.0/24(rw,no_root_squash)
```

### 3) 디렉토리 생성
공유할 디렉토리 생성
```bash
[root@nfs.server.com ~]# mkdir /home/nfsshare
```

### 4) 서비스 설정
```bash
[root@nfs.server.com ~]# systemctl enable --now rpcbind nfs-server
```

### 5) 방화벽 설정
```bash
[root@nfs.server.com ~]# firewall-cmd --add-service=nfs --permanent
success
[root@nfs.server.com ~]# firewall-cmd --reload
success
```
NFSv3 사용 시 설정
```bash
[root@nfs.server.com ~]# firewall-cmd --add-service={nfs3,mountd,rpc-bind} --permanent
success
[root@nfs.server.com ~]# firewall-cmd --reload
success
```

## 1. NFS 클라이언트 구성
### 1) 패키지 설치
7버전 이상은 기본 설치 되어있다.
```bash
[root@nfs.client.com ~]# dnf -y install nfs-utils
```

### 2) 설정파일 변경
domain 변경
```bash
[root@nfs.client.com ~]# vi /etc/idmapd.conf
# line 5 : uncomment and change to your domain name
Domain = client.com
```

### 3) MOUNT
서버에 생성했던 디렉토리와 원하는 디렉토리와 마운트
```bash
[root@nfs.client.com ~]# mount -t nfs nfs.server.com:/home/nfsshare /mnt/nfs
[root@nfs.client.com ~]# df -hT
Filesystem                   Type      Size  Used Avail Use% Mounted on
devtmpfs                     devtmpfs  1.9G     0  1.9G   0% /dev
tmpfs                        tmpfs     1.9G     0  1.9G   0% /dev/shm
tmpfs                        tmpfs     1.9G  8.6M  1.9G   1% /run
tmpfs                        tmpfs     1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/cs-root          xfs        26G  2.3G   24G   9% /
/dev/vda1                    xfs      1014M  259M  756M  26% /boot
tmpfs                        tmpfs     374M     0  374M   0% /run/user/0
nfs.server.com:/home/nfsshare nfs4       26G  2.3G   24G   9% /mnt/nfs
```

NFSv3 사용 시 마운트 방법
```bash
[root@nfs.client.com ~]# mount -t nfs -o vers=3 dlp.srv.world:/home/nfsshare /mnt/nfs
[root@nfs.client.com ~]# df -hT /mnt/nfs
Filesystem                   Type  Size  Used Avail Use% Mounted on
nfs.server.com:/home/nfsshare nfs    26G  2.3G   24G   9% /mnt/nfs
```

영구적 마운트를 위한 /etc/fstab 설정
```bash
[root@nfs.client.com ~]# vi /etc/fstab
/dev/mapper/cs-root     /                       xfs     defaults        0 0
UUID=72e65d16-7d1a-40bc-9bc1-e45a8ba6d084 /boot xfs     defaults        0 0
/dev/mapper/cs-swap     none                    swap    defaults        0 0
# add to the end : set NFS share
nfs.server.com:/home/nfsshare /mnt/nfs             nfs     defaults        0 0
```

## 3. autofs 설정
### 1) 패키지 설치
```bash
[root@nfs.client.com ~]# dnf -y install autofs
```

### 2) 설정파일 변경
직접설정 방식
```bash
[root@nfs.client.com ~]# vi /etc/auto.master.d/direct.autofs
/-    /etc/auto.mount

[root@nfs.client.com ~]# vi /etc/auto.mount
# create new : [mount point] [option] [location]
/mnt/nfs   -fstype=nfs,rw  nfs.server.com:/home/nfsshare
```

간접설정 방식
```bash
[root@nfs.client.com ~]# vi /etc/auto.master.d/indirect.autofs
/mnt    /etc/auto.mount2

[root@nfs.client.com ~]# vi /etc/auto.mount2
# create new : [mount point] [option] [location]
nfs   -fstype=nfs,rw  nfs.server.com:/home/nfsshare
```

와일드카드맵
```bash
[root@nfs.client.com ~]# vi /etc/auto.master.d/indirect.autofs
/mnt    /etc/auto.mount2

[root@nfs.client.com ~]# vi /etc/auto.mount2
# create new : [mount point] [option] [location]
*   -fstype=nfs,rw  nfs.server.com:/home/&
```

### 3) 서비스 설정
[root@nfs.client.com ~]# systemctl enable --now autofs

### 4) mount point에서 작업하기
server의 디렉토리와 mount한 디렉토리에서 작업
```bash
[root@nfs.client.com ~]# cd /mnt
[root@nfs.client.com mnt]# ll
total 8
-rw-r--r--. 1 root root 10 Mar  3 19:14 testfile.txt
-rw-r--r--. 1 root root  5 Mar  3 19:17 test.txt
```
