# iSCSI 설정
## 실습 시나리오
1. 논리볼륨 생성
2. 생성한 논리볼륨으로 공유 설정
3. 이니시에이터에서 연결 및 확인
4. 논리볼륨 확장 및 확인

## 기본 설정방법
### 1. target 설정
#### 0) 사전 준비
1. 별도의 디스크 준비
2. 필요 시 IP 주소 및 호스트네임 설정
3. 해당 디스크에 대한 파티셔닝 및 논리볼륨 생성
```bash
[root@target ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                   8:0    0  100G  0 disk
├─sda1                8:1    0    1G  0 part /boot
└─sda2                8:2    0   99G  0 part
  ├─rl_nobreak-root 253:0    0 63.9G  0 lvm  /
  ├─rl_nobreak-swap 253:1    0    4G  0 lvm  [SWAP]
  └─rl_nobreak-home 253:2    0 31.2G  0 lvm  /home
sdb                   8:16   0    8G  0 disk
└─sdb1                8:17   0    8G  0 part
[root@target ~]# vgcreate vg0 /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
  Volume group "vg0" successfully created
[root@target ~]# lvcreate -n iscsi -L 2G vg0
  Logical volume "iscsi" created.
[root@target ~]# lvs
  LV    VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home  rl_nobreak -wi-ao---- <31.18g
  root  rl_nobreak -wi-ao---- <63.86g
  swap  rl_nobreak -wi-ao----   3.96g
  iscsi vg0        -wi-a-----   2.00g
```

#### 1) 패키지 설치
```bash
[root@target ~]# dnf install -y targetcli
```

#### 2) 서비스 설정
도구 실행 및 기본 상태 확인
```bash
[root@target ~]# targetcli
Warning: Could not load preferences file /root/.targetcli/prefs.bin.
targetcli shell version 2.1.53
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> ls
o- / ..................................................................................................... [...]
  o- backstores .......................................................................................... [...]
  | o- block .............................................................................. [Storage Objects: 0]
  | o- fileio ............................................................................. [Storage Objects: 0]
  | o- pscsi .............................................................................. [Storage Objects: 0]
  | o- ramdisk ............................................................................ [Storage Objects: 0]
  o- iscsi ........................................................................................ [Targets: 0]
  o- loopback ..................................................................................... [Targets: 0]
/>
```
backstores 설정 (공유할 장치 지정)
```bash
/> backstores/block create block01 /dev/vg0/iscsi
Created block storage object block01 using /dev/vg0/iscsi.
```

iscsi 설정 (target의 IQN 설정)
```bash
/> iscsi/ create iqn.2022-04.com.example.nobreak:target
Created target iqn.2022-04.com.example.nobreak:target.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
```
tpg 와 portal 자동 생성됨
IQN은 네트워크 상에서 구분이 가능한 고유값으로 지정

iscsi 설정2 (tpg 설정)
```bash
/> iscsi/iqn.2022-04.com.example.nobreak:target/tpg1/acls create iqn.2022-04.com.example.nobreak:initiator
Created Node ACL for iqn.2022-04.com.example.nobreak:initiator
/> iscsi/iqn.2022-04.com.example.nobreak:target/tpg1/luns create /backstores/block/block01
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2022-04.com.example.nobreak:initiator
```

확인
```bash
/> ls
o- / ..................................................................................................... [...]
  o- backstores .......................................................................................... [...]
  | o- block .............................................................................. [Storage Objects: 1]
  | | o- block01 ................................................ [/dev/vg0/iscsi (2.0GiB) write-thru activated]
  | |   o- alua ............................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ................................................... [ALUA state: Active/optimized]
  | o- fileio ............................................................................. [Storage Objects: 0]
  | o- pscsi .............................................................................. [Storage Objects: 0]
  | o- ramdisk ............................................................................ [Storage Objects: 0]
  o- iscsi ........................................................................................ [Targets: 1]
  | o- iqn.2022-04.com.example.nobreak:target ........................................................ [TPGs: 1]
  |   o- tpg1 ........................................................................... [no-gen-acls, no-auth]
  |     o- acls ...................................................................................... [ACLs: 1]
  |     | o- iqn.2022-04.com.example.nobreak:initiator ........................................ [Mapped LUNs: 1]
  |     |   o- mapped_lun0 ........................................................... [lun0 block/block01 (rw)]
  |     o- luns ...................................................................................... [LUNs: 1]
  |     | o- lun0 .......................................... [block/block01 (/dev/vg0/iscsi) (default_tg_pt_gp)]
  |     o- portals ................................................................................ [Portals: 1]
  |       o- 0.0.0.0:3260 ................................................................................. [OK]
  o- loopback ..................................................................................... [Targets: 0]
/>
```
현재 설정은 모든 인터페이스에 대한 설정
특정 인터페이스를 전용으로 사용하고 싶을 경우 portal 항목을 제거하고 새로 추가
```bash
/> iscsi/iqn.2022-04.com.example.nobreak:target/tpg1/portals/ delete 0.0.0.0 3260
Deleted network portal 0.0.0.0:3260
/> iscsi/iqn.2022-04.com.example.nobreak:target/tpg1/portals/ create 10.0.2.10 3260
Using default IP port 3260
Created network portal 10.0.2.10:3260.
/> exit
```
exit 입력 시 설정 상태 자동 저장

#### 3) 서비스 활성화
```bash
[root@target ~]# systemctl status target
● target.service - Restore LIO kernel target configuration
   Loaded: loaded (/usr/lib/systemd/system/target.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
[root@target ~]# systemctl enable target
Created symlink /etc/systemd/system/multi-user.target.wants/target.service → /usr/lib/systemd/system/target.service.
[root@target ~]# systemctl status target
● target.service - Restore LIO kernel target configuration
   Loaded: loaded (/usr/lib/systemd/system/target.service; enabled; vendor preset: disabled)
   Active: inactive (dead)
```
이 서비스는 이전 작업에서 자동 저장한 데이터를 불러오기 하는 서비스
활성화해야 해당 데이터를 그대로 사용 (활성화하지 않고 재부팅 시 targetcli 를 실행하면 초기화됨)

#### 4) 방화벽 설정
```bash
[root@target ~]# firewall-cmd --add-service=iscsi-target
success
[root@target ~]# firewall-cmd --add-service=iscsi-target --permanent
success
```
만약 기본 포트가 아닌 다른 포트로 portal 항목 설정 시에는 포트 번호 직접 지정
(기본 3260/tcp)

### 2. initiator 설정
#### 1) 패키지 설치
```bash
[root@initiator ~]# dnf install -y iscsi-initiator-utils
```
운영체제 설치 시 기본설치 (지웠을 경우에만 설치)

#### 2) IQN 설정
```bash
[root@initiator ~]# vim /etc/iscsi/initiatorname.iscsi
[root@initiator ~]# cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.2022-04.com.example.nobreak:initiator
```
타겟에서 ACL 항목 설정 시 지정한 이름과 통일
일치하지 않으면 연결 시 오류 발생 (검색은 됨)

#### 3) 검색
```bash
[root@initiator ~]# iscsiadm -m discovery -t st -p 10.0.2.10
10.0.2.10:3260,1 iqn.2022-04.com.example.nobreak:target
```
방화벽 / 포털 설정 문제 시 오류 발생

#### 4) 연결
```bash
[root@initiator ~]# iscsiadm -m node -T iqn.2022-04.com.example.nobreak:target -l
Logging in to [iface: default, target: iqn.2022-04.com.example.nobreak:target, portal: 10.0.2.10,3260]
Login to [iface: default, target: iqn.2022-04.com.example.nobreak:target, portal: 10.0.2.10,3260] successful.
```
연결 시 -T 옵션으로는 검색된 타겟의 IQN 이름 지정
-l 옵션을 사용해야 연결 (안쓰면 정보만 출력)

#### 5) 연결 오류
##### 오류 내용
```bash
[root@initiator ~]# cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.2022-04.com.example.nobreak:error
[root@initiator ~]# iscsiadm -m discovery -t st -p 10.0.2.10
10.0.2.10:3260,1 iqn.2022-04.com.example.nobreak:target
[root@initiator ~]# iscsiadm -m node -T iqn.2022-04.com.example.nobreak:target -l
Logging in to [iface: default, target: iqn.2022-04.com.example.nobreak:target, portal: 10.0.2.10,3260]
iscsiadm: Could not login to [iface: default, target: iqn.2022-04.com.example.nobreak:target, portal: 10.0.2.10,3260].
iscsiadm: initiator reported error (8 - connection timed out)
iscsiadm: Could not log into all portals
```
##### 해결과정 - 1. 서비스 중지
```bash
[root@initiator ~]# systemctl status iscsid
● iscsid.service - Open-iSCSI
   Loaded: loaded (/usr/lib/systemd/system/iscsid.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-04-27 11:53:18 KST; 3min 15s ago
     Docs: man:iscsid(8)
           man:iscsiuio(8)
           man:iscsiadm(8)
 Main PID: 4191 (iscsid)
   Status: "Ready to process requests"
    Tasks: 1 (limit: 23545)
   Memory: 2.1M
   CGroup: /system.slice/iscsid.service
           └─4191 /usr/sbin/iscsid -f

Apr 27 11:55:12 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:14 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:16 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:18 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:20 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:22 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:24 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:26 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:28 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:30 initiator.nobreak.example.com iscsid[4191]: iscsid: Connection4:0 to [target: iqn.2022-04.com.e>
[root@initiator ~]# systemctl stop iscsid
Warning: Stopping iscsid.service, but it can still be activated by:
  iscsid.socket
[root@initiator ~]# systemctl status iscsid
● iscsid.service - Open-iSCSI
   Loaded: loaded (/usr/lib/systemd/system/iscsid.service; disabled; vendor preset: disabled)
   Active: inactive (dead) since Wed 2022-04-27 11:56:43 KST; 9s ago
     Docs: man:iscsid(8)
           man:iscsiuio(8)
           man:iscsiadm(8)
  Process: 4191 ExecStart=/usr/sbin/iscsid -f (code=exited, status=0/SUCCESS)
 Main PID: 4191 (code=exited, status=0/SUCCESS)
   Status: "Ready to process requests"

Apr 27 11:55:20 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:22 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:24 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:26 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:28 initiator.nobreak.example.com iscsid[4191]: iscsid: Login I/O error, failed to send a PDU
Apr 27 11:55:30 initiator.nobreak.example.com iscsid[4191]: iscsid: Connection4:0 to [target: iqn.2022-04.com.e>
Apr 27 11:56:43 initiator.nobreak.example.com systemd[1]: Stopping Open-iSCSI...
Apr 27 11:56:43 initiator.nobreak.example.com iscsid[4191]: iscsid shutting down.
Apr 27 11:56:43 initiator.nobreak.example.com systemd[1]: iscsid.service: Succeeded.
Apr 27 11:56:43 initiator.nobreak.example.com systemd[1]: Stopped Open-iSCSI.
[root@initiator ~]# systemctl status iscsid.socket
● iscsid.socket - Open-iSCSI iscsid Socket
   Loaded: loaded (/usr/lib/systemd/system/iscsid.socket; enabled; vendor preset: disabled)
   Active: active (listening) since Wed 2022-04-27 11:53:17 KST; 3min 46s ago
     Docs: man:iscsid(8)
           man:iscsiadm(8)
   Listen: @ISCSIADM_ABSTRACT_NAMESPACE (Stream)
    Tasks: 0 (limit: 23545)
   Memory: 0B
   CGroup: /system.slice/iscsid.socket

Apr 27 11:53:17 initiator.nobreak.example.com systemd[1]: Listening on Open-iSCSI iscsid Socket.
[root@initiator ~]# systemctl stop iscsid.socket
[root@initiator ~]# systemctl status iscsid.socket
● iscsid.socket - Open-iSCSI iscsid Socket
   Loaded: loaded (/usr/lib/systemd/system/iscsid.socket; enabled; vendor preset: disabled)
   Active: inactive (dead) since Wed 2022-04-27 11:57:09 KST; 2s ago
     Docs: man:iscsid(8)
           man:iscsiadm(8)
   Listen: @ISCSIADM_ABSTRACT_NAMESPACE (Stream)

Apr 27 11:53:17 initiator.nobreak.example.com systemd[1]: Listening on Open-iSCSI iscsid Socket.
Apr 27 11:57:09 initiator.nobreak.example.com systemd[1]: iscsid.socket: Succeeded.
Apr 27 11:57:09 initiator.nobreak.example.com systemd[1]: Closed Open-iSCSI iscsid Socket.
```
실행 중인 서비스 데몬(iscsid.service)과 연결된 소켓유닛(iscsid.socket)을 함께 중지

##### 해결과정 - 2. 검색 기록 제거
```bash
[root@initiator ~]# iscsiadm -m node -T iqn.2022-04.com.example.nobreak:target -o delete
```

##### 해결과정 - 3. 이름 재설정
##### 해결과정 - 4. 재검색 및 연결
##### 위 내용과 동일해서 3/4는 생략. 서비스 및 소켓은 iscsiadm 명령어 사용 시 자동 실행.

#### 볼륨 사용
```bash
[root@initiator ~]# iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 6.2.1.4-1
Target: iqn.2022-04.com.example.nobreak:target (non-flash)
        Current Portal: 10.0.2.10:3260,1
        Persistent Portal: 10.0.2.10:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2022-04.com.example.nobreak:initiator
                Iface IPaddress: 10.0.2.20
                Iface HWaddress: default
                Iface Netdev: default
                SID: 5
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 3  State: running
                scsi3 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdb          State: running
[root@initiator ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                   8:0    0  100G  0 disk
├─sda1                8:1    0    1G  0 part /boot
└─sda2                8:2    0   99G  0 part
  ├─rl_nobreak-root 253:0    0 63.9G  0 lvm  /
  ├─rl_nobreak-swap 253:1    0    4G  0 lvm  [SWAP]
  └─rl_nobreak-home 253:2    0 31.2G  0 lvm  /home
sdb                   8:16   0    2G  0 disk
sr0                  11:0    1 1024M  0 rom
```
추가된 장치 확인 가능
해당 장치에 대한 파티셔닝 및 마운트 작업 후 사용 (생략)

## 볼륨 확장 기능
### target 설정
#### 논리볼륨 크기 확장 (생략)
#### initiator 설정
```bash
[root@initiator ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                   8:0    0  100G  0 disk
├─sda1                8:1    0    1G  0 part /boot
└─sda2                8:2    0   99G  0 part
  ├─rl_nobreak-root 253:0    0 63.9G  0 lvm  /
  ├─rl_nobreak-swap 253:1    0    4G  0 lvm  [SWAP]
  └─rl_nobreak-home 253:2    0 31.2G  0 lvm  /home
sdb                   8:16   0    2G  0 disk /mnt
sr0                  11:0    1 1024M  0 rom
[root@initiator ~]# df -h
Filesystem                   Size  Used Avail Use% Mounted on
devtmpfs                     1.8G     0  1.8G   0% /dev
tmpfs                        1.9G     0  1.9G   0% /dev/shm
tmpfs                        1.9G  9.2M  1.9G   1% /run
tmpfs                        1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/rl_nobreak-root   64G  5.4G   59G   9% /
/dev/mapper/rl_nobreak-home   32G  255M   31G   1% /home
/dev/sda1                   1014M  258M  757M  26% /boot
tmpfs                        374M   32K  374M   1% /run/user/0
/dev/sdb                     2.0G   70M  2.0G   2% /mnt
[root@initiator ~]# iscsiadm -m node -T iqn.2022-04.com.example.nobreak:target -R
Rescanning session [sid: 8, target: iqn.2022-04.com.example.nobreak:target, portal: 10.0.2.10,3260]
[root@initiator ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                   8:0    0  100G  0 disk
├─sda1                8:1    0    1G  0 part /boot
└─sda2                8:2    0   99G  0 part
  ├─rl_nobreak-root 253:0    0 63.9G  0 lvm  /
  ├─rl_nobreak-swap 253:1    0    4G  0 lvm  [SWAP]
  └─rl_nobreak-home 253:2    0 31.2G  0 lvm  /home
sdb                   8:16   0    3G  0 disk /mnt
sr0                  11:0    1 1024M  0 rom
[root@initiator ~]# df -h
Filesystem                   Size  Used Avail Use% Mounted on
devtmpfs                     1.8G     0  1.8G   0% /dev
tmpfs                        1.9G     0  1.9G   0% /dev/shm
tmpfs                        1.9G  9.2M  1.9G   1% /run
tmpfs                        1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/rl_nobreak-root   64G  5.4G   59G   9% /
/dev/mapper/rl_nobreak-home   32G  255M   31G   1% /home
/dev/sda1                   1014M  258M  757M  26% /boot
tmpfs                        374M   32K  374M   1% /run/user/0
/dev/sdb                     2.0G   70M  2.0G   2% /mnt
[root@initiator ~]# xfs_growfs /dev/sdb
meta-data=/dev/sdb               isize=512    agcount=11, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=1336320, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1336320 to 1572864
[root@initiator ~]# df -h
Filesystem                   Size  Used Avail Use% Mounted on
devtmpfs                     1.8G     0  1.8G   0% /dev
tmpfs                        1.9G     0  1.9G   0% /dev/shm
tmpfs                        1.9G  9.2M  1.9G   1% /run
tmpfs                        1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/rl_nobreak-root   64G  5.4G   59G   9% /
/dev/mapper/rl_nobreak-home   32G  255M   31G   1% /home
/dev/sda1                   1014M  258M  757M  26% /boot
tmpfs                        374M   32K  374M   1% /run/user/0
/dev/sdb                     3.0G   76M  3.0G   2% /mnt
```
iscsiadm 명령어의 -R 옵션으로 다시 스캔하면 블록 자체의 크기 감지 가능
하지만 이미 마운트하고 사용 중이었다면 파일시스템 크기는 그대로
파일시스템 확장 명령어를 추가로 사용하면 실제 사이즈 인식

