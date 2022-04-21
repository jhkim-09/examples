# FreeIPA 구성 예제

## 0. 사전 준비
### 1) 정적 IP 주소 설정
```bash
[root@localhost ~]# nmcli con add type ethernet ifname enp0s3 con-name ipa ipv4.addresses 10.0.2.10/24 ipv4.gateway 10.0.2.1 ipv4.dns 8.8.8.8 ipv4.method manual
Connection 'ipa' (ca5363e6-a9e7-41b5-a0cf-e4c5dfa2a595) successfully added.
[root@localhost ~]# nmcli con up ipa
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
```

### 2) 호스트네임 설정
```bash
[root@localhost ~]# hostnamectl set-hostname ipa.nobreak.example.com
```

### 3) /etc/hosts 파일 설정
```bash
[root@ipa ~]# echo "10.0.2.10 ipa.nobreak.example.com" >> /etc/hosts
[root@ipa ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.0.2.10 ipa.nobreak.example.com
```

## 1. IPA 서버 구성 (로컬 DNS 없을 때 -> DNS까지 함께 구성)
### 1) 패키지 설치
```bash
[root@ipa ~]# dnf module -y install idm:DL1/dns
```

### 2) 설치작업
```bash
[root@ipa ~]# ipa-server-install --setup-dns

The log file for this installation can be found in /var/log/ipaserver-install.log
==============================================================================
This program will set up the IPA Server.
Version 4.9.6

This includes:
  * Configure a stand-alone CA (dogtag) for certificate management
  * Configure the NTP client (chronyd)
  * Create and configure an instance of Directory Server
  * Create and configure a Kerberos Key Distribution Center (KDC)
  * Configure Apache (httpd)
  * Configure DNS (bind)
  * Configure SID generation
  * Configure the KDC to enable PKINIT

To accept the default shown in brackets, press the Enter key.

Enter the fully qualified domain name of the computer
on which you're setting up server software. Using the form
<hostname>.<domainname>
Example: master.example.com.


Server host name [ipa.nobreak.example.com]:

Warning: skipping DNS resolution of host ipa.nobreak.example.com
The domain name has been determined based on the host name.

Please confirm the domain name [nobreak.example.com]:

The kerberos protocol requires a Realm name to be defined.
This is typically the domain name converted to uppercase.

Please provide a realm name [NOBREAK.EXAMPLE.COM]:
Certain directory server operations require an administrative user.
This user is referred to as the Directory Manager and has full access
to the Directory for system management tasks and will be added to the
instance of directory server created for IPA.
The password must be at least 8 characters long.

Directory Manager password:
Password (confirm):

The IPA server requires an administrative user, named 'admin'.
This user is a regular system account used for IPA server administration.

IPA admin password:
Password (confirm):

Checking DNS domain nobreak.example.com., please wait ...
Do you want to configure DNS forwarders? [yes]:
Following DNS servers are configured in /etc/resolv.conf: 8.8.8.8
Do you want to configure these servers as DNS forwarders? [yes]:
All detected DNS servers were added. You can enter additional addresses now:
Enter an IP address for a DNS forwarder, or press Enter to skip:
DNS forwarders: 8.8.8.8
Checking DNS forwarders, please wait ...
Do you want to search for missing reverse zones? [yes]:
Checking DNS domain 2.0.10.in-addr.arpa., please wait ...
Do you want to create reverse zone for IP 10.0.2.10 [yes]:
Please specify the reverse zone name [2.0.10.in-addr.arpa.]:
Checking DNS domain 2.0.10.in-addr.arpa., please wait ...
Using reverse zone(s) 2.0.10.in-addr.arpa.
Trust is configured but no NetBIOS domain name found, setting it now.
Enter the NetBIOS name for the IPA domain.
Only up to 15 uppercase ASCII letters, digits and dashes are allowed.
Example: EXAMPLE.


NetBIOS domain name [NOBREAK]:

Do you want to configure chrony with NTP server or pool address? [no]:

The IPA Master Server will be configured with:
Hostname:       ipa.nobreak.example.com
IP address(es): 10.0.2.10
Domain name:    nobreak.example.com
Realm name:     NOBREAK.EXAMPLE.COM

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=NOBREAK.EXAMPLE.COM
Subject base: O=NOBREAK.EXAMPLE.COM
Chaining:     self-signed

BIND DNS server will be configured to serve IPA domain with:
Forwarders:       8.8.8.8
Forward policy:   only
Reverse zone(s):  2.0.10.in-addr.arpa.

Continue to configure the system with these values? [no]: yes
```
패스워드 설정 및 마지막 확인 항목만 yes
나머지는 다 기본값 선택 (Enter)
약 5분 소요?

### 3) 확인
```bash
[root@ipa ~]# kinit admin
Password for admin@NOBREAK.EXAMPLE.COM:
[root@ipa ~]# klist
Ticket cache: KCM:0
Default principal: admin@NOBREAK.EXAMPLE.COM

Valid starting       Expires              Service principal
04/14/2022 16:24:17  04/15/2022 15:26:40  krbtgt/NOBREAK.EXAMPLE.COM@NOBREAK.EXAMPLE.COM
```

### 4) 방화벽 설정
```bash
[root@ipa ~]# firewall-cmd --add-service={freeipa-ldap,freeipa-ldaps,dns,ntp}
success
[root@ipa ~]# firewall-cmd --add-service={freeipa-ldap,freeipa-ldaps,dns,ntp} --permanent
success
```

### 5) 서비스 상태 확인
```bash
[root@ipa ~]# systemctl status ipa
● ipa.service - Identity, Policy, Audit
   Loaded: loaded (/usr/lib/systemd/system/ipa.service; enabled; vendor preset: disabled)
   Active: active (exited) since Thu 2022-04-14 16:23:31 KST; 2min 25s ago
  Process: 40310 ExecStart=/usr/sbin/ipactl start (code=exited, status=0/SUCCESS)
 Main PID: 40310 (code=exited, status=0/SUCCESS)
    Tasks: 0 (limit: 23545)
   Memory: 0B
   CGroup: /system.slice/ipa.service

Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting Directory Service
Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting krb5kdc Service
Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting kadmin Service
Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting named Service
Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting httpd Service
Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting ipa-custodia Service
Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting pki-tomcatd Service
Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting ipa-otpd Service
Apr 14 16:23:31 ipa.nobreak.example.com ipactl[40310]: Starting ipa-dnskeysyncd Service
Apr 14 16:23:31 ipa.nobreak.example.com systemd[1]: Started Identity, Policy, Audit.
```
설치 시 기본 활성화

## 2. IPA 클라이언트 연결
### *) 설치 전 NTP 설정 단계
기본적으로 서비스 구성 시 서버와 클라이언트 시간 동기화 필요
동일한 NTP 서버로 동기화 혹은 IPA 서버를 NTP 서버로 구성 및 지정 가능
#### 서버쪽 구성
```bash
[root@ipa ~]# dnf -y install chrony
[root@ipa ~]# vi /etc/chrony.conf
# line 3 : change servers to synchronize (replace to your own timezone NTP server)
# need NTP server itself to sync time with other NTP server
#pool 2.centos.pool.ntp.org iburst
pool ntp.nict.jp iburst
# line 24 : add network range to allow to receive time synchronization requests from NTP Clients
# specify your local network and so on
# if not specified, only localhost is allowed
allow 10.0.2.0/24
[root@ipa ~]# systemctl enable --now chronyd
[2]	If Firewalld is running, allow NTP service. NTP uses [123/UDP].
[root@ipa ~]# firewall-cmd --add-service=ntp
success
[root@ipa ~]# firewall-cmd --runtime-to-permanent
success
```
Server with GUI 로 운영체제 설치 시 chrony 패키지는 기본 설치됨
서버가 사용할 NTP 서버 지정 및 NTP 서버로써 동작할 IP대역을 지정
설정파일 수정 후 서비스 활성화(재시작) 및 방화벽(ntp) 오픈

#### 클라이언트 구성
```bash
[root@client ~]# dnf -y install chrony
[root@client ~]# vi /etc/chrony.conf
# line 3 : change to your own NTP server or others in your timezone
#pool 2.centos.pool.ntp.org iburst
pool ipa.nobreak.example.com iburst
[root@client ~]# systemctl enable --now chronyd
# verify status
[root@client ~]# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^? ipa.nobreak.example.com                 2   6     3     0   +665us[ +665us] +/-   10ms
```

#### 여기에서는 따로 구성 없이 기본 NTP 서버로 진행



### 0) 호스트네임 설정
```bash
[root@localhost ~]# hostnamectl set-hostname client.nobreak.example.com
```

### 1) DNS entry 추가 (IPA 서버에서 실행)
```bash
[root@ipa ~]# ipa dnsrecord-add nobreak.example.com client --a-rec 10.0.2.11
  Record name: client
  A record: 10.0.2.11
```

### 2) 패키지 설치
```bash
[root@client ~]# dnf module -y install idm:DL1/client
```

### 3) IP주소 및 DNS 주소 변경
```bash
[root@client ~]# nmcli con add type ethernet ifname enp0s3 con-name client ipv4.addresses 10.0.2.11/24 ipv4.gateway 10.0.2.1 ipv4.dns 10.0.2.10 ipv4.method manual
Connection 'client' (9c3df0b0-b37d-4300-9e95-645ad152157c) successfully added.
[root@client ~]# nmcli con up client
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/7)
```
DNS 주소는 IPA 서버 구성 시 DNS까지 함께 구성했으므로 IPA 서버 주소로 지정
(반드시 패키지 설치 후 변경)

### 4) 설치
```bash
[root@client ~]# ipa-client-install
This program will set up IPA client.
Version 4.9.6

Discovery was successful!
Do you want to configure chrony with NTP server or pool address? [no]:
Client hostname: client.nobreak.example.com
Realm: NOBREAK.EXAMPLE.COM
DNS Domain: nobreak.example.com
IPA Server: ipa.nobreak.example.com
BaseDN: dc=nobreak,dc=example,dc=com

Continue to configure the system with these values? [no]: yes
Synchronizing time
No SRV records of NTP servers found and no NTP server or pool address was provided.
Using default chrony configuration.
Attempting to sync time with chronyc.
Time synchronization was successful.
User authorized to enroll computers: admin
Password for admin@NOBREAK.EXAMPLE.COM:
Successfully retrieved CA cert
    Subject:     CN=Certificate Authority,O=NOBREAK.EXAMPLE.COM
    Issuer:      CN=Certificate Authority,O=NOBREAK.EXAMPLE.COM
    Valid From:  2022-04-14 07:19:04
    Valid Until: 2042-04-14 07:19:04

Enrolled in IPA realm NOBREAK.EXAMPLE.COM
Created /etc/ipa/default.conf
Configured sudoers in /etc/authselect/user-nsswitch.conf
Configured /etc/sssd/sssd.conf
Configured /etc/krb5.conf for IPA realm NOBREAK.EXAMPLE.COM
Systemwide CA database updated.
Adding SSH public key from /etc/ssh/ssh_host_ecdsa_key.pub
Adding SSH public key from /etc/ssh/ssh_host_ed25519_key.pub
Adding SSH public key from /etc/ssh/ssh_host_rsa_key.pub
SSSD enabled
Configured /etc/openldap/ldap.conf
Configured /etc/ssh/ssh_config
Configured /etc/ssh/sshd_config
Configuring nobreak.example.com as NIS domain.
Client configuration complete.
The ipa-client-install command was successful
```


## 3. IPA 사용자 설정 및 확인
### 1) 기본 생성 및 확인
```bash
[root@ipa ~]# ipa user-add ipauser01
First name: Ipa
Last name: User01
----------------------
Added user "ipauser01"
----------------------
  User login: ipauser01
  First name: Ipa
  Last name: User01
  Full name: Ipa User01
  Display name: Ipa User01
  Initials: IU
  Home directory: /home/ipauser01
  GECOS: Ipa User01
  Login shell: /bin/sh
  Principal name: ipauser01@NOBREAK.EXAMPLE.COM
  Principal alias: ipauser01@NOBREAK.EXAMPLE.COM
  Email address: ipauser01@nobreak.example.com
  UID: 968400003
  GID: 968400003
  Password: False
  Member of groups: ipausers
  Kerberos keys available: False
```
사용자 생성 (이름만 지정)
옵션으로 First / Last name 지정 및 패스워드 설정 가능

```bash
[root@client ~]# getent passwd ipauser01
ipauser01:*:968400003:968400003:Ipa User01:/home/ipauser01:/bin/sh
[root@client ~]# ssh ipauser01@localhost
The authenticity of host 'localhost (<no hostip for proxy command>)' can't be established.
ECDSA key fingerprint is SHA256:hBb7l10yLqgKykhOMkqce3KM+qcZIddMquKSikXTygo.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
Password:
```
확인은 가능하지만 패스워드를 설정하지 않아서 사용불가

### 2) 패스워드 설정
```bash
[root@ipa ~]# ipa passwd ipauser01
New Password:
Enter New Password again to verify:
----------------------------------------------------
Changed password for "ipauser01@NOBREAK.EXAMPLE.COM"
----------------------------------------------------
```
사용자 이름 미지정 시 admin 사용자 패스워드 변경 가능

```bash
[root@client ~]# ssh ipauser01@localhost
Password:
Password expired. Change your password now.
Current Password:
New password:
Retype new password:
Password change failed. Server message: Password is too short

Password not changed.

Password:
Password expired. Change your password now.
Current Password:
New password:
Retype new password:
Activate the web console with: systemctl enable --now cockpit.socket

This system is not registered to Red Hat Insights. See https://cloud.redhat.com/
To register this system, run: insights-client --register

Last failed login: Thu Apr 14 17:45:15 KST 2022 from ::1 on ssh:notty
There were 2 failed login attempts since the last successful login.
Last login: Thu Apr 14 17:44:24 2022 from ::1
Could not chdir to home directory /home/ipauser01: No such file or directory
[ipauser01@client /]$
[ipauser01@client /]$ pwd
/
[ipauser01@client /]$ id
uid=968400003(ipauser01) gid=968400003(ipauser01) groups=968400003(ipauser01) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[ipauser01@client /]$ grep ipauser01 /etc/passwd
[ipauser01@client /]$ getent passwd ipauser01
ipauser01:*:968400003:968400003:Ipa User01:/home/ipauser01:/bin/sh
```
접속 시 패스워드 변경 요구 
접속한 사용자는 로컬사용자가 아니라 IPA 사용자
접속은 했으나 홈디렉토리가 없어서 / 디렉토리로 접속됨

### 3) 홈디렉토리 설정
```bash
[root@client ~]# authselect enable-feature with-mkhomedir
Make sure that SSSD service is configured and enabled. See SSSD documentation for more information.

- with-mkhomedir is selected, make sure pam_oddjob_mkhomedir module
  is present and oddjobd service is enabled and active
  - systemctl enable --now oddjobd.service

[root@client ~]# systemctl enable --now oddjobd.service
Created symlink /etc/systemd/system/multi-user.target.wants/oddjobd.service → /usr/lib/systemd/system/oddjobd.service.
[root@client ~]# ssh ipauser01@localhost
Password:
Activate the web console with: systemctl enable --now cockpit.socket

This system is not registered to Red Hat Insights. See https://cloud.redhat.com/
To register this system, run: insights-client --register

Last login: Thu Apr 14 17:45:28 2022 from ::1
[ipauser01@client ~]$ pwd
/home/ipauser01
```
이 작업은 한 번만 수행하면 모든 사용자에 대해 적용
각 사용자로 접속 시 홈디렉토리 자동 생성 (처음 한 번)

## 4. 서비스 키탭파일 생성 및 복사 (NFS)
