# Group 4 - HA Cluster (FailOver)

- Create 2 CentOS 7 Machines on VMware
- Setup User on Each Machines
- Basic machine setup configuration

**machine 1**

*Go to root user*

*find ip of machines using;* 

```sh
ip a 
```

*centos7.0 192.168.241.128/24*
*centos7.2 192.168.241.129/24*

```sh
vi /etc/host
```

*Type ip and machine name in vim editor:*
```sh
192.168.241.128 node1
192.168.241.129 node2
```

*connecting machine 1 to machine 2;*
```sh
scp/etc/hosts 192.168.241.129:/etc/
```

*ping machine 2*
```sh
ping 192.168.241.129
```

*ping machine 1*
```sh
ping 192.168.241.128
```

*we can go from machine 1 to machine 2 by;*
###### ssh machine Name or ip
```sh
ssh 192.168.241.129
```
  
*check services on machine;*
```sh
chkconfig --list
```

*save ip tables firewall rules on machine 1;*
```sh
iptables-save
```

*ip firewall flush;*
```sh
iptables -F
iptables -L
```

*save ip tables firewall rules on machine 2;*
```sh
iptables-save
```

*ip firewall flush;*
```sh
iptables -F
iptables -L
```

======================================================================================

**machine 1**

```sh
yum install ntp -y
```

*checking ntp*
```sh
rpm -qa : grep ntp
```

*install ntp on 2nd machine as well;*
```sh
yum install ntp -y
rpm -qa : grep ntp
```

*configure ntp in frist machine to make it server machine*


```sh
vi /etc/ntp.conf
```

*put this line in vi editor;*

```sh
server 127.127.1.0
```

*ntpd service restart & enable*

```sh
systemctl retart ntpd
```
```sh
enable ntpd
```

*checking if ntp is syncronizing or not;*

```sh
watch ntpq -p -n
```

*now we will check other machine's ntp*

*configure ntp in second machine to make it client machine*
```sh
vi /etc/ntp.conf
```
```sh
server 192.168.241.128
```
```sh
systemctl retart ntpd
```
```sh
enable ntpd
```
```sh
watch ntpq -p -n
```

======================================================================================

**machine 1**

*creating partitions;*


```sh
fdisk -l
```

```sh
fdisk /dev/sdb1
```
 
- enter n for new 
- p for primary
- enter 1 default value
- enter
- enter
- enter t *for type prompt*
- 8e *change type of partition*
- w *save and write partition in the list*


```sh partprobe ``` *restart partition to check* 

*partiton will be now visible in fdisk -l*

*now we will perform same thing in machine 2;*


```sh
fdisk -l
```
```sh
fdisk /dev/sdb1
```

*enter*
-n  
-p 
-enter 1 
-enter
-enter
-enter t 
-8e 
-w 

```sh
partprobe
```
```sh
fdisk -l
```

======================================================================================

**machine 1**

*physical volume*
```sh
pvcreate /dev/sdb1
```

lvm partition
```sh
vgcreate vgdrbd /dev/sdb1
```

```sh
lvcreate -n lvdrbd /dev/mapper/vgdrbd -L +4000M
```

```sh
lvdisplay | more
```
```sh
vgdisplay | more
```

**machine 2**

```sh
pvcreate /dev/sdb1
```
```sh
lvm partition
```
```sh
vgcreate vgdrbd /dev/sdb1
```
```sh
lvcreate -n lvdrbd /dev/mapper/vgdrbd -L +4000M
```
```sh
lvdisplay | more
```
```sh
vgdisplay | more
```

======================================================================================

**machine 1**

```sh
vi /etc/sysctl.conf
```

*control source route verification*

*type this in editor:*

```sh
net.ipv4.conf.ens33.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.ens33.arp_announce = 2
```


```sh
sysctl -p
```

**machine 2**


```sh
vi /etc/sysctl.conf
```

```sh
net.ipv4.conf.ens33.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.ens33.arp_announce = 2
```

```sh
sysctl -p
```

======================================================================================

**machine 1**

*add repl repository*
```sh
rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
```

*importing key for encryption*
```sh
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
```

*check which drbd version is supported by kernel*
```sh
yum info *drbd* | grep name
```

*install the desired drbd version*
```sh
yum -y install drbd84-utils kmod-drbd84
```

**machine 2**

```sh
rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
```
```sh
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
```
```sh
yum info *drbd* | grep name
```
```sh
yum -y install drbd84-utils kmod-drbd84
```

======================================================================================

**machine 1**

*configuring drbd file*


```sh
vi /etc/drbd.conf
```

*add # at start of both lines with include*
*and type:*


```sh
global{
usage-count yes;
}
common{
syncer{
rate 10M;
}
}
resource r0{
protocol C;
handlers{
pri-on-incon-degr"echo o > /proc/sysrq-trigger ; halt -f ";
pri-lost-after-sb "echo o > /proc/sysrq-trigger; halt -f ";
local-io-error "echo o > /proc/sysrq-trigger; halt -f ";
outdate-peer "/usr/lib/heartbeat/drbd-peer-outdater -t 5";
}
startup{
}
disk{
on-io-error detach;
}
net{
after-sb-0pri disconnect;
after-sb-1pri disconnect;
after-sb-2pri disconnect;
rr-conflict disconnect;
}
syncer{
rate 10M;
al-extents 257;
}
on node1{
device /dev/drbd0;
disk /dev/vgdrbd/lvdrbd;
address 192.168.241.128:7788
meta-disk internal;
}
on node2{
device /dev/drbd0;
disk /dev/vgdrbd/lvdrbd;
address 192.168.241.129:7788
meta-disk internal;
}
}

```

*now same work for machine 2 as well;*

```sh
vi /etc/drbd.conf
```


```sh
global{
usage-count yes;
}
common{
syncer{
rate 10M;
}
}
resource r0{
protocol C;
handlers{
pri-on-incon-degr"echo o > /proc/sysrq-trigger ; halt -f ";
pri-lost-after-sb "echo o > /proc/sysrq-trigger; halt -f ";
local-io-error "echo o > /proc/sysrq-trigger; halt -f ";
outdate-peer "/usr/lib/heartbeat/drbd-peer-outdater -t 5";
}
startup{
}
disk{
on-io-error detach;
}
net{
after-sb-0pri disconnect;
after-sb-1pri disconnect;
after-sb-2pri disconnect;
rr-conflict disconnect;
}
syncer{
rate 10M;
al-extents 257;
}
on node1{
device /dev/drbd0;
disk /dev/vgdrbd/lvdrbd;
address 192.168.241.128:7788
meta-disk internal;
}
on node2{
device /dev/drbd0;
disk /dev/vgdrbd/lvdrbd;
address 192.168.241.129:7788
meta-disk internal;
}
}
```

======================================================================================

**machine 1**

```sh
modprobe drbd
```
```sh
echo drbd
```

*putting drbd module in kernel;*
```sh
lsmod | grep -i drbd
```
```sh
echo "modprobe drbd" >> /etc/rc.local
```

======================================================================================

**machine 1**

```sh
drbdadm create-md r0
```

```sh
groupadd haclient
```
```sh
chgrp haclient /lib/drbd/drbdsetup-84
```
```sh
chmod o-x /lib/drbd/drbdsetup-84
```
```sh
chmod u+s /lib/drbd/drbdsetup-84
```
```sh
chgrp haclient /usr/sbin/drbmeta
```
```sh
chmod o-x /usr/sbin/drbmeta
```
```sh
chmod u+s /usr/sbin/drbmeta
```


*Attach the drbd0 device to sdb1 disk;*
```sh
drbdadm up r0
```
```sh
lsblk
```

*Assign Ports in Firewall;*
```sh
firewall-cmd --zone=internal --add-port=7788-7799/tcp --permanent
```
```sh
firewall-cmd --zone=internal --add-port=7788-7799/udp --permanent
```
```sh
firewall-cmd --reload
```

*Start and Enable DRBD:*
```sh
systemctl start drbd
```
```sh
systemctl enable drbd
```










