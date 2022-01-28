


**High Availability Cluster with DRBD, Pacemaker and Squid**

We are going to use DRBD (Distributed Redundant Block Device), Pacemaker and Squid Server for Synchronization and High Availability Cluster will also be created. The role of DRBD is to establish synchronization between two systems and allows the replication of data These are the steps we are going to follow to achieve synchronization:

1.  Establishing Connection between Machines
2.  Stopping unnecessary services and running iptables command
3.  Checking NTP services
4.  Making LVM for DRBD

4a) Creating partitions 4b) Creating physical volume 4c) Creating volume group 4d) Create logical volume

5.  Including entries in sysctl config file
6.  Installing and Configuring DRBD
7.  Installing and Configuring Pacemaker
8.  Setting up Squid Server

**1: ESTABLISHING CONNECTION BETWEEN MACHINES**

First we need to check IP on both Centos machines,  `szabist`  and  `szabist1`  by typing command:

```
ifconfig

```

IP's of both machines:  `192.168.117.138`  szabist1  `192.168.117.128`  szabist Now we have to add them in the hosts file the command will be

```
vim /etc/hosts

```

Add these entries in the file:

```
192.168.117.138 szabist1
192.168.117.128 szabist

```

Now save the files and type below commands on each machine to check if they are connected

```
   ping szabist
   ping szabist1

```

**2: STOPPING UNNECESSARY SERVICES AND RUNNING IP TABLES COMMAND**

In this step first we need to disable services like cups.path. The command below will be used for disabling:

```
systemctl disable cups.path

```

Now we have to check the command is actually disabled.

```
systemctl status cups.path

```

Now we have to flush ip tables on both machines by typing below command:

```
iptables -F

```

To check output type the below command.

```
iptables -L

```

**3: CHECKING NTP SERVICE**

Now we have to check if NTP(Network Time Protocol) is installed and working on both machines or not because it is important to check if both servers are synchronized with each other. Type below command to check NTP service is installed or not.

```
rpm -qa | grep 'chrony'

```

Now we have to enable our machine ip for NTP request by changing the entry in the configuration file of chrony.

```
vim /etc/chrony.conf

```

Now, we have to allow both our machine's IPs.

```
allow 192.168.117.128
allow 192.168.117.138

```

Restart the chrony service.

```
systemctl restart chronyd

```

Now, type this command to allow the NTP incoming requests.

```
firewall-cmd --permanent --add-service=ntp
firewall-cmd --reload

```

**4: MAKING LVM FOR DRBD**

**4a) Creating partitions for LVM**  First we have to create LVM partitions on both machines.

To check details of disk we have to type command:

```
fdisk -l

```

The second step will be to create partitions in sdb so we will do that by typing below command.

```
fdisk /dev/sdb

```

Now press n for creating new partitions After that, press p for primary. Now press enter by selecting all options as default. A new partition will be created. For converting type from Linux to Linux LVM we have to change HEX code to 8e. for that we have type t then provide the code 8e. Now save all the changes by pressing w.

**4b) Creating physical volume**  For creating physical volume type the following command.

```
pvcreate /dev/sdb1

```

**4c) Creating Volume Group**  for creating volume group type the following command

```
vgcreate vgdrbd /dev/sdb1

```

**4d) Creating Logical Volume**  Now for creating logical volume we have type following command.

```
lvcreate -n lvdrbd /dev/mapper/vgdrbd -L +3000M

```

Now make the filesystem by typing the command:

```
mkfs.ext3 /dev/mapper/vgdrbd-lvdrbd

```

**5: INCLUDING ENTRIES IN SYSCTL CONFIG FILE**

We will add few entries in the sysctl config file of both the machines by typing below command.

```
vi etc/sysctl.conf

net.ipv4.conf.ens33.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.ens33.arp_announce = 2

```

Now look at the output by typing below command:

```
sysctl -p

```

**6: INSTALLING AND CONFIGURING DRBD**

By the help of the following command, you can install drbd on both machines:  `dnf -y install https://download-ib01.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/d/drbd-9.17.0-1.el8.x86_64.rpm`

By the help of the following command, you can install drbd-utils:  `dnf -y install https://download-ib01.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/d/drbd-utils-9.17.0-1.el8.x86_64.rpm`

By the help of the following command, you can install kernel module for drbd:  `dnf -y install https://mirror.rackspace.com/elrepo/elrepo/el8/x86_64/RPMS/kmod-drbd90-9.1.5-1.el8_5.elrepo.x86_64.rpm`

After the complete installation of drbd, make a new file, and make changes.

```
nano /etc/drbd.d/resource0.res

resource resource0 {
protocol C;
on szabist {
device /dev/drbd0;
disk /dev/vgdrbd/lvdrbd;
address 192.168.117.128:7788;
meta-disk internal;
}
on szabist1 {
device /dev/drbd0;
disk /dev/vgdrbd/lvdrbd;
address 192.168.117.138:7788;
meta-disk internal;
}
}

```

Allow the port  `7788`  on the firewall to avoid any kind of issue.  `firewall-cmd --zone=public --permanent --add-port 7788/tcp`

Reload using this command:  `firewall-cmd --reload`

Start DRBD by using the following command:

```
modprobe drbd

```

Now echo modprobe drd in rc.local by the following command:

```
echo "modprobe drbd" >> /etc/rc.local

```

Initialize the DRBD resource by this command:

```
drbdadm create-md resource0

```

Now, enable the resource:

```
drbdadm up resource0

```

Check the status of the drbd:

```
drbdadm status resource0

```

On your first machine, start the initial full synchronization:

```
drbdadm primary --force resource0

```

Now, check the status:

```
drbdadm status resource0

```

Your output will change from the previous one.

Now create the filesystem:

```
mkfs.ext4 /dev/drbd0

```

Mount the DRBD device on any directory, we are using /opt:

```
mount /dev/drbd0 /opt/

```

Create some files inside /opt directory by using command:

```
cd /opt
touch file1 file2 file3

```

Now, on the first machine, unmount the DRBD and make it secondary:

```
cd
umount /opt
drbdadm secondary resource0

```

On the second machine, make the second node primary with this command:

```
drbdadm primary resource0

```

Now, mount the DRBD device on the /opt directory with the following command:

```
mount /dev/drbd0 /opt

```

Type and run this command to print the list of files in the /opt directory:

```
ls /opt

```

All the files stored in the DRBD device will be shown there. Here, the installation and configuration of DRBD is successful.

**7: INSTALLING AND CONFIGURING PACEMAKER**

Now, enable the High Availability Repo and install pacemaker:

```
dnf --enablerepo=ha -y install pacemaker pcs

```

Now, enable pcs:

```
systemctl enable --now pcsd

```

Set the password for the cluster admin:

```
passwd hacluster

```

Allow the HA service:

```
firewall-cmd --add-service=high-availability --permanent
firewall-cmd --reload

```

Create authorization among nodes and configure basic cluster settings:

```
pcs host auth szabist szabist1

pcs cluster setup ha_cluster szabist szabist1

```

Start services for cluster:

```
pcs cluster start --all

```

Make it auto-start:

```
pcs cluster enable --all

```

Now check status:

```
pcs cluster status

```

The output will be following:

```
Cluster Status:
 Cluster Summary:
   * Stack: corosync
   * Current DC: szabist1 (version 2.1.0-8.el8-7c3f660707) - partition with quorum
   * Last updated: Wed Jan 26 15:30:29 2022
   * Last change:  Tue Jan 25 05:44:32 2022 by hacluster via crmd on szabist
   * 2 nodes configured
   * 0 resource instances configured
 Node List:
   * Online: [ szabist szabist1 ]

PCSD Status:
  szabist: Online
  szabist1: Online

```

Also check the status for corosync:

```
pcs status corosync

```

The output will be following:

```
Membership information
----------------------
    Nodeid      Votes Name
         1          1 szabist
         2          1 szabist1 (local)

```

**8: SETTING UP SQUID**

Install squid using the following command:

```
dnf install squid -y

```

Back up the configuration file:

```
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.ori

```

Access the configuration file:

```
vim /etc/squid/squid.conf

```

Allow squid on the firewall:

```
firewall-cmd --add-service=squid --permanent 
firewall-cmd --reload

```

Now it's time for the configuration of CentOS client:

```
sudo vim /etc/profile.d/proxyserver.sh

```

Add these settings:

```
MY_PROXY_URL="szabist:3128" 
HTTP_PROXY=$MY_PROXY_URL
HTTPS_PROXY=$MY_PROXY_URL
FTP_PROXY=$MY_PROXY_URL
http_proxy=$MY_PROXY_URL
https_proxy=$MY_PROXY_URL
ftp_proxy=$MY_PROXY_URL

```

Then finally source the file:

```
source /etc/profile.d/proxyserver.sh

```

The squid proxy has successfully been installed.
