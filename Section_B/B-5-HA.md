# **High Availability Cluster with DRBD and Pacemaker/Corosync**

In this tutorial we are going to learn how to use DRBD (Distributed Redundant Block Device) and Pacemaker/Corosync for server synchronization under High Availability. We will also configure Squid for that purpose later in it.

**Step-1: Establishing connection**

We have two  **CentOS**  machines, Hanzala and Ali . First we will resolve them for DNS in order to communicate with each other.

On both machines type:

# vim /etc/hosts

Add this entry in the table:

192.168.223.128 Hanzala 192.168.223.129 Ali

Now save the file and type below command on the terminal of both machines. # ping Ali

Output will be:

PING Ali (192.168.223.129) 56(84) bytes of data. 64 bytes from Ali (192.168.223.129): icmp\_seq=1 ttl=64 time=0.062 ms 64 bytes from Ali (192.168.223.129): icmp\_seq=2 ttl=64 time=0.108 ms 64 bytes from Ali (192.168.223.129): icmp\_seq=3 ttl=64 time=0.102 ms bytes from Ali (192.168.223.129): icmp\_seq=4 ttl=64 time=0.105 ms

**Step-2: Stopping unnecessary services**

In this step we have to stop unnecessary services like Sendmail and Bluetooth But since my both server doesn&#39;t have Sendmail service so I will only stop Bluetooth services. To stop it type below command.

# systemctl stop bluetooth

Now to check if it is really stopped type below command.

# systemctl status bluetooth

Output will be this.

● bluetooth.service - Bluetooth service Loaded: loaded (/usr/lib/systemd/system/bluetooth.service; enabled; vendor preset: enabled) Active: inactive (dead) since Mon 2022-01-24 12:50:13 PKT; 7s ago Docs: man:bluetoothd(8) Process: 921 ExecStart=/usr/libexec/bluetooth/bluetoothd (code=exited, status=0/SUCCESS) Main PID: 921 (code=exited, status=0/SUCCESS) Status: &quot;Powering down&quot; Jan 24 12:35:30 Hanzala bluetoothd[921]: Endpoint registered: sender=:1.295 path=/MediaEndpoint/A2DPSink/sbc Jan 24 12:35:30 Hanzala bluetoothd[921]: Endpoint registered: sender=:1.295 path=/MediaEndpoint/A2DPSource/sbc Jan 24 12:50:12 Hanzala systemd[1]: Stopping Bluetooth service... Jan 24 12:50:12 Hanzala bluetoothd[921]: Terminating Jan 24 12:50:13 Hanzala bluetoothd[921]: Endpoint unregistered: sender=:1.295 path=/MediaEndpoint/A2DPSink/sbc Jan 24 12:50:13 Hanzala bluetoothd[921]: Endpoint unregistered: sender=:1.295 path=/MediaEndpoint/A2DPSource/sbc Jan 24 12:50:13 Hanzala bluetoothd[921]: Stopping SDP server Jan 24 12:50:13 Hanzala bluetoothd[921]: Exit Jan 24 12:50:13 Hanzala systemd[1]: bluetooth.service: Succeeded. Jan 24 12:50:13 Hanzala systemd[1]: Stopped Bluetooth service.

Run both above commands on Alimachine as well.

**Step-3: Flushing the IP-Tables**

We will installed our required services without ip tables in the firewall so we will flush the ip-tables on both machines by typing the below command.

# iptables -F

To check the output type below command.

# iptables -L

Output will be

Chain INPUT (policy ACCEPT) target prot opt source destination
 Chain FORWARD (policy ACCEPT) target prot opt source destination
 Chain OUTPUT (policy ACCEPT) target prot opt source destination
 Chain LIBVIRT\_INP (0 references) target prot opt source destination
 Chain LIBVIRT\_OUT (0 references) target prot opt source destination
 Chain LIBVIRT\_FWO (0 references) target prot opt source destination
 Chain LIBVIRT\_FWI (0 references) target prot opt source destination
 Chain LIBVIRT\_FWX (0 references) target prot opt source destination

Now save this IP table by typing below command.

# service iptables save

Output will be

iptables: Saving firewall rules to /etc/sysconfig/iptables:[OK]

Now repeat this same step in Ali machine.

**Step-3: Checking NTP service**

Now we will check if NTP(Network time Protocol) is installed and working properly on both machines or not because it is very important that both servers shall be synchronized with each other.

To check if NTP service is installed or not type below command.

# rpm -qa | grep &#39;chrony&#39;

Output.

chrony-4.1-1.el8.x86\_64

Chrony is a different form of NTP server and according to this output NTP service is installed so we will just configure it according to our requirements.

We will enable our machine ip for NTP request by changing the entry in the chrony configuration file.

# vi /etc/chrony.conf

Uncomment the entry of allow 192.168.223.128 (there should be your server IP) then restart the service by typing below command.

# systemctl restart chronyd

Now, open the firewall port to permit the NTP incoming requests by typing below command.

# firewall-cmd --permanent --add-service=ntp # firewall-cmd --reload

Output will be:

success

Now, To check if the NTP services is running properly, type below command.

# timedatectl

Output should be:

Local time: Mon 2022-01-24 14:06:54 PKT Universal time: Mon 2022-01-24 09:06:54 UTC RTC time: Mon 2022-01-24 09:06:53 Time zone: Asia/Karachi (PKT, +0500) System clock synchronized: yes NTP service: active RTC in local TZ: no

Now repeat this step in Ali machine.

**Step-4: Making LVM for DRBD**

**4.1: Creating Partitions for LVM**

For this, first we need to create LVM partition on both machines.

To check disk details type # fdisk -l command and output should be like this.

Disk /dev/nvme0n1p1: 20 GiB, 21474836480 bytes, 41943040 sectors Units: sectors of 1 \* 512 = 512 bytes Sector size (logical/physical): 512 bytes / 512 bytes I/O size (minimum/optimal): 512 bytes / 512 bytes Disklabel type: dos Disk identifier: 0xcfb5a6d8

Device Boot Start End Sectors Size Id Type /dev/nvme0n1p11 \*
 2048 2099199 2097152 1G 83 Linux /dev/nvme0n1p12 2099200 41943039 39843840 19G 8e Linux LVM

Disk /dev/nvme0n1p2: 5 GiB, 5368709120 bytes, 10485760 sectors Units: sectors of 1 \* 512 = 512 bytes Sector size (logical/physical): 512 bytes / 512 bytes I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/mapper/cl-root: 17 GiB, 18249416704 bytes, 35643392 sectors Units: sectors of 1 \* 512 = 512 bytes Sector size (logical/physical): 512 bytes / 512 bytes I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/mapper/cl-swap: 2 GiB, 2147483648 bytes, 4194304 sectors Units: sectors of 1 \* 512 = 512 bytes Sector size (logical/physical): 512 bytes / 512 bytes I/O size (minimum/optimal): 512 bytes / 512 bytes

In the above output we can see that disk nvme0n1p2 is unpartitioned so we will create partition of it. Type following commands.

# fdisk /dev/nvme0n1p2

Now type n for new partitions. See the output.

Command (m for help): n Partition type p primary (0 primary, 0 extended, 4 free) e extended (container for logical partitions)

Now type p for primary. Now type 1 for creating only one partition. Now select all option as default for this, press enter at all options. After that your partition is created and to check type p.

Output will be:

Device Boot Start End Sectors Size Id Type /dev/nvme0n2p1
 2048 10485759 10483712 5G 83 Linux

Now for using this disk for LVM we will change its HEX code to 8e. So type t then provide the code which is 8e to this disk.

The output will be:

Command (m for help): t Selected partition 1 Hex code (type L to list all codes): 8e Changed type of partition &#39;Linux&#39; to &#39;Linux LVM&#39;.

Now to save all these changes into the just type w for write.

Output:

Command (m for help): w The partition table has been altered. Calling ioctl() to re-read partition table. Syncing disks.

Now our desired partition is created.

Disk /dev/nvme0n1p2: 5 GiB, 5368709120 bytes, 10485760 sectors Units: sectors of 1 \* 512 = 512 bytes Sector size (logical/physical): 512 bytes / 512 bytes I/O size (minimum/optimal): 512 bytes / 512 bytes Disklabel type: dos Disk identifier: 0x0fee0b2a

Device Boot Start End Sectors Size Id Type /dev/nvme0n2p1
 2048 10485759 10483712 5G 8e Linux LVM

Do this same step in Ali machine.

**4.2: Creating Physical Volume**

For creating physical volume type following command.

# pvcreate /dev/nvme0n2p1

Output:

Physical volume &quot;/dev/nvme0n2p1&quot; successfully created.

**4.3: Creating Volume Group**

For volume group type:

# vgcreate vgdrbd /dev/nvme0n2p1

Output:

Volume group &quot;vgdrbd&quot; successfully created

**4.4: Creating Logical Volume**

Now for creating logical volume type:

# lvcreate -n lvdrbd /dev/mapper/vgdrbd -L +5000M

Output:

Logical volume &quot;lvdrbd&quot; created.

Now type lvdisplay | more to see the details

--- Logical volume --- LV Path /dev/vgdrbd/lvdrbd LV Name lvdrbd VG Name vgdrbd LV UUID SdZcKj-97l0-8Efx-5h7C-edm1-7w2s-GpwbWP LV Write Access read/write LV Creation host, time Hanzala, 2022-01-24 17:33:35 +0500 LV Status available #open 0 LV Size 4.88 GiB Current LE 1250 Segments 1 Allocation inherit Read ahead sectors auto -currently set to 8192 Block device 253:2

Do these same steps in Ali machine.

**Step-5: Adding entries in System control file**

We will now add some entries in the system control file by typing the below command.

# vi /etc/sysctl.conf

Insert following entries into the file.

net.ipv4.conf.ens33.arp\_ignore = 1

net.ipv4.conf.all.arp\_announce = 2

net.ipv4.conf.ens33.arp\_announce = 2

Now run this file by typing # sysctl -p

Output:

net.ipv4.conf.ens33.arp\_ignore = 1 net.ipv4.conf.all.arp\_announce = 2 net.ipv4.conf.ens33.arp\_announce = 2

Do this same step in Ali machine.

**Step-6: Setting Up DRBD**

To install DRBD type the following commands.

# dnf -y install https://download-ib01.fedoraproject.org/pub/epel/8/Everything/x86\_64/Packages/d/drbd-9.17.0-1.el8.x86\_64.rpm

# dnf -y install https://download-ib01.fedoraproject.org/pub/epel/8/Everything/x86\_64/Packages/d/drbd-utils-9.17.0-1.el8.x86\_64.rpm

# dnf -y install https://mirror.rackspace.com/elrepo/elrepo/el8/x86\_64/RPMS/kmod-drbd90-9.1.5-1.el8\_5.elrepo.x86\_64.rpm

After that create a config file of DRBD by typing # nano /etc/drbd.d/resource0.res and make following entries.

resource resource0 { protocol C; on Hanzala { device /dev/drbd0; disk /dev/vgdrbd/lvdrbd; address 192.168.223.128:7788; meta-disk internal; } on Ali { device /dev/drbd0; disk /dev/vgdrbd/lvdrbd; address 192.168.223.129:7788; meta-disk internal; } }

Now allow the port 7788 on the firewall to avoid any kind of issue.

firewall-cmd --zone=public --permanent --add-port 7788/tcp firewall-cmd --reload

Starting DRBD module by typing following command. # modprobe drbd

If you want to start this services at system startup then type the following command.

# echo &quot;modprobe drbd&quot; \&gt;\&gt; /etc/rc.local

Now create DRBD resource by typing: # drbdadm create-md resource0

Now up the created resource by:

# drbdadm up resource0

Do these above all step on the Ali as well otherwise connection will not be established.

Now run this below command on the node that you want to make it a primary. I will run this command on my Hanzala machine.

# drbdadm primary --force resource0

After that check the status by typing:

# drbdadm status resource0

The output should be look like this.

esource0 role:Primary disk:UpToDate Ali role:Secondary peer-disk:UpToDate

On the primary node make file system of drbd0 by typing following commands.

mkfs.ext3 /dev/drbd0

Output:

mke2fs 1.45.6 (20-Mar-2020) Creating filesystem with 1279951 4k blocks and 320000 inodes Filesystem UUID: 936a0e13-c312-4a88-b293-f72a776eba5b Superblock backups stored on blocks: 32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
 Writing inode tables: done
 Creating journal (16384 blocks): done Writing superblocks and filesystem accounting information: done

Now we will create a directory on both machines in which we will mount our file system: mkdir /data/

Now mount the filesystem in primary node Hanzala which we have just created.

mount /dev/drbd0 /data/

Now we have established the connection we will install Pacemaker/Corosync next.

**Step-7: Setting Up Pacemaker/Corosync**

First we need to enable our High Availability Repository, for that type:

# yum repolist all | grep -i HighAvailability

Output will be:

ha CentOS Linux 8 - HighAvailability disabled

As you can see is disabled so we will enable it by typing following command.

# yum config-manager --set-enabled ha

After that install the following package for HA cluster by typing following command.

# yum install --assumeyes pacemaker corosync pcs

Now start this service by typing:

# systemctl start pcsd

Now to enable the service at startup, create a symlink by typing the following command:

# systemctl enable pacemaker corosync pcsd

Now allow this service on the firewall by: # firewall-cmd --permanent --add-service=high-availability # firewall-cmd --reload

Now enable authorization among nodes by following command. These should be done on a primary node.

# pcs host auth Hanzala Ali

Now configure the cluster by.

# pcs cluster setup ha\_cluster Hanzala Ali

Now start and enable the pcs service by # pcs cluster start --all # pcs cluster enable --all

Check the pcs status by typing.

# pcs cluster status

Output:

Cluster Status: Cluster Summary: \* Stack: corosync \* Current DC: Hanzala (version 2.1.0-8.el8-7c3f660707) - partition with quorum

- Last updated: Tue Jan 25 00:39:45 2022
- Last change: Tue Jan 25 00:39:33 2022 by hacluster via crmd on Hanzala
- 2 nodes configured
- 0 resource instances configured Node List: \* Online: [Hanzala Ali]

PCSD Status: Ali: Online Hanzala: Online

Now check corosync status by

# pcs status corosync

Output:

**Membership information**

Nodeid Votes Name

1 1 Hanzala (local)

2 1 Ali

Now create a configuration file for HA by:

# vi /etc/ha.d/ha.cf

And paste the following entries:

logfacility local0 keepalive 2 #deadtime 30 # USE THIS!!! deadtime 10 #we use two heartbeat links, eth2 and serial 0 bcast ens33 ####### We can use eth1 instead of eth0 it&#39;s better option ######## #serial /dev/ttyS0 baud 19200 auto\_failback on ################## Active Active state ################# node Hanzala node Ali

Save the file and exit. Now on the primary node create another configuration file by typing:

# vi /etc/ha.d/haresources

And paste the following entries.

Hanzala IPaddr::192.168.1.104/24/eth0 drbddisk::resource0 Filesystem::/dev/drbd0::/data::ext3 squid

Now on the second machine which is Ali create the same config file and paste below given entries.

Ali IPaddr::192.168.1.104/24/eth0 drbddisk::resource0 Filesystem::/dev/drbd0::/data::ext3 squid

Now create Auth keys on both servers by first creating the configuration file then inserting the values..

# vi /etc/ha.d/authkeys

Make the following entry in it.

auth 3 3 md5 redhat

Now change its permission type by:

# chmod 600 /etc/ha.d/authkeys

**Step-8: Setting Up Squid Server**

Before installing squid on both machines we will enable IP packet forwarding on both machines by adding an entry in the system control configuration file.

# vim /etc/sysctl.conf

then add following entry in it.

net.ipv4.ip\_forward = 1

Now run the command # sysctl -p for executing configurations.

Now install by typing the following command.

# yum install -y squid

Now open squid configuration file by typing

# vim /etc/squid/squid.conf

and insert the following entries.

acl our\_networks src 192.168.1.0/24 192.168.2.0/24 http\_access allow our\_networks http\_port 3128 transparent visible\_hostname squid.ha-cluster.com cache\_dir ufs /data/squid 1000 32 256

Do this step on the second machine as well.

Now we have to create a squid cache directory. For this create a directory named squid in /data/ directory by typing command.

# cd /data/ # mkdir squid

Then change its mode by # chown squid:squid squid

Now create swap directories by typing:

# squid -z

Now for redirecting client port type the following command.

# iptables -t nat -A PREROUTING -p tcp -i eth0 --dport 80 -j REDIRECT --to-port 3128

**Step-9: Configuring CentOS client on proxy**

First open the server proxy file by:

# vi /etc/profile.d/proxyserver.sh

Insert the following entries in it.

MY\_PROXY\_URL=&quot;192.168.223.128:3128&quot; HTTP\_PROXY=$ MY\_PROXY\_URL HTTPS\_PROXY=$ MY\_PROXY\_URL FTP\_PROXY=$ MY\_PROXY\_URL http\_proxy=$ MY\_PROXY\_URL https proxy=$ MY\_PROXY\_URL ftp\_proxy=$MY\_PROXY\_URL

Now source the file typing the below command.

# source /etc/profile.d/proxyserver.sh

THE END