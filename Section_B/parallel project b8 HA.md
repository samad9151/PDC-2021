Go to root user
_find ip of machines using;_
ip a

vim /etc/host

_Type ip and machine name in vim editor:_

192.168.241.131 machine 1
192.168.241.130 machine 2

_connecting machine 1 to machine 2;_

scp/etc/hosts 192.168.241.130:/etc/

_ping machine 2_

ping 192.168.241.130

_ping machine 1_

ping 192.168.241.131

_we can go from machine 1 to machine 2 by;_



ssh 192.168.241.131

_check services on machine;_

chkconfig --list

_save ip tables firewall rules on machine 1;_

iptables-save

_ip firewall flush;_

iptables -F
iptables -L

_save ip tables firewall rules on machine 2;_

iptables-save

_ip firewall flush;_

iptables -F
iptables -L

_ping machine 2_

ping 192.168.241.129

_ping machine 1_

ping 192.168.241.128

_we can go from machine 1 to machine 2 by;_

###### [](https://github.com/asmryz/PDC-2021/blob/main/Section_A/A-4-HA.md#ssh-machine-name-or-ip)ssh machine Name or ip

ssh 192.168.241.129

_check services on machine;_

chkconfig --list

_save ip tables firewall rules on machine 1;_

iptables -F
iptables -L

iptables-save

_save ip tables firewall rules on machine 2;_

iptables -F
iptables -L

iptables-save

machine 1

yum install ntp -y

_checking ntp_

rpm -qa : grep ntp

_install ntp on 2nd machine as well;_

yum install ntp -y

rpm -qa : grep ntp

configure ntp in first machine to make it server machine_

vim /etc/ntp.conf

put this line in vi editor;
server 127.127.1.0
ntpd service restart & enable_
systemctl restart ntpd
enable ntpd
watch ntpq -p -n
configure ntp in second machine to make it client machine
vim /etc/ntp.conf
server 192.168.241.131
systemctl retart ntpd
enable ntpd
watch ntpq -p -n
_creating partitions;_
fdisk -l
fdisk /dev/sdb1
fdisk -l
fdisk /dev/sdb1
_enter_  -n  
-p -enter 1 -enter -enter -enter t -8e -w
partprobe
fdisk -l
machine 1

_physical volume_
pvcreate /dev/sdb1
lvm partition
vgcreate vgdrbd /dev/sdb1
lvcreate -n lvdrbd /dev/mapper/vgdrbd -L +4000M
lvdisplay | more
vgdisplay | more
vim /etc/sysctl.conf

control source route verification
type this in editor:

net.ipv4.conf.ens33.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.ens33.arp_announce = 2

sysctl -p
machine 2

pvcreate /dev/sdb1

lvm partition

vgcreate vgdrbd /dev/sdb1

lvcreate -n lvdrbd /dev/mapper/vgdrbd -L +4000M
lvdisplay | more
vgdisplay | more
vim /etc/sysctl.conf

net.ipv4.conf.ens33.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.ens33.arp_announce = 2

sysctl -p

machine 1

_add repl repository_
rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
importing key for encryption
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
check which drbd version is supported by kernel
yum info *drbd* | grep name
install the desired drbd version
yum -y install drbd84-utils kmod-drbd84

machine 2

rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
yum info *drbd* | grep name
yum -y install drbd84-utils kmod-drbd84

machine 1

configuring drbd file
vim /etc/drbd.conf
 machine 2 
vim /etc/drbd.conf
machine 1
modprobe drbd
echo drbd
putting drbd module in kernel;
lsmod | grep -i drbd
echo "modprobe drbd" >> /etc/rc.local
drbdadm create-md r0
groupadd haclient
chgrp haclient /lib/drbd/drbdsetup-84
drbdadm up r0
lsblk
firewall-cmd --zone=internal --add-port=7788-7799/tcp --permanent
firewall-cmd --zone=internal --add-port=7788-7799/udp --permanent
firewall-cmd --reload
_Start and Enable DRBD:_

systemctl start drbd

systemctl enable drbd

-   [](https://github.com/ "GitHub")



