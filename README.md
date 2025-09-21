# Comprehensive Guide to Apache CloudStack Installation
This document provides a detailed, step-by-step guide for installing Apache CloudStack in an "All-in-One" configuration, where the CloudStack Management Server and the KVM host reside on the same machine. This guide is tailored for a general reader and includes explanations for each command and configuration change.

## Table of Contents
[Apache CloudStack 4.20 on Ubuntu 24.04](#apache-cloudstack-4.20-on-ubuntu-24.04)
1. [Network and System Configuration](#1.-network-and-system-configuration)
2. [Apache CloudStack Management Server Installation](#2.-apache-cloudstack-management-server-installation)
3. KVM Hypervisor and CloudStack Agent Configuration
4. Launching the Cloud and Final Steps
Apache CloudStack 4.18 on Ubuntu 22.04
1. Network and System Configuration
2. Apache CloudStack Management Server Installation
3. KVM Hypervisor and CloudStack Agent Configuration
4. Launching the Cloud and Final Steps

## Apache CloudStack 4.20 on Ubuntu 24.04
This section details the installation for Apache CloudStack version 4.20 on a single Ubuntu 24.04 machine, serving as both the management server and the virtualization host.

### 1. Network and System Configuration
This first set of steps prepares the host machine by setting up a static network configuration and installing necessary tools.

**Network Details:**
**Home Network:** 192.168.101.0/24
**Gateway:** 192.168.101.1
**Management IP:** 192.168.101.4
**System IPs:** 192.168.101.51-70
**Public IPs:** 192.168.101.71-90

**Step 1.1:** Set a Static IP Address with Netplan
netplan is a utility for easily configuring networking on Ubuntu systems. This configuration creates a cloudbr0 bridge, which is essential for CloudStack to manage virtual machine networks.
First, rename any existing configuration files by adding a .bak extension.
Then, edit the 01-netcfg.yaml file and add the following configuration. This sets up VLANs and the bridge with a static IP and gateway.





## ALL in ONE - Cloudstack Management and Host in One Machine

==============================================================
### NETWORK/SYSTEM CONFIGURATION AND ADDITIONAL TOOLS

Home network: 192.168.101.0/24
Gateway: 192.168.101.1
Subnet Mask: 255.255.255.0

IP Address for management: 192.168.101.4
IP Address for system: 192.168.101.51-70
IP Address for public: 192.168.101.71-90

### SET STATIC IP ADDRESS FOR MANAGEMENT SERVER
#### Network configuration with netplan
#### Rename all existing configuration by adding .bak
#### ****:~$ cat /etc/netplan/01-netcfg.yaml
```
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
      optional: true
  vlans:
    vlan.2101:
      id: 2101
      link: "eno1"
      dhcp4: false
      dhcp6: false
    vlan.262:
      id: 262
      link: "eno1"
      dhcp4: false
      dhcp6: false
      addresses: [***.***.101.4/24]
      nameservers:
        addresses: [8.8.8.8,8.8.4.4]
  bridges:
    cloudbr0:
      interfaces: [vlan.2101]
      addresses: [192.168.101.4/24]
      routes:
        - to: default
          via: 192.168.101.1
      mtu: 1500
      nameservers:
        addresses: [8.8.8.8,8.8.4.4]
      parameters:
        stp: false
        forward-delay: 0
```
#### Apply your network configuration
```
sudo -i
netplan generate
netplan apply
reboot
```
#### Update Your System and Install Some Useful Tools
```
apt update & upgrade
apt install htop lynx duf -y
apt install bridge-utils
```
#### Configure LVM (OPTIONAL)
```
#not required unless logical volume is used
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```
#### Install SSH Server and Others Tools 
```
apt-get install openntpd openssh-server sudo vim htop tar -y
apt-get install intel-microcode -y
passwd root
#change it to password
```
#### Enable Root Login (PermitRootLogin)
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
```
#### Set Timezone 
```
timedatectl set-timezone Asia/Jakarta
```
==============================================================
## APACHE CLOUDSTACK INSTALLATION
### CloudStack Management Server and Everything in one machine

#### Reference
```
https://rohityadav.cloud/blog/cloudstack-kvm/
```

#### CloudStack Management Server Setup from SHAPEBLUE ACS 4.20
```
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null

echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.20 / > /etc/apt/sources.list.d/cloudstack.list
apt-get update -y

#update and install cloudstack management and mysql
#it takes very long time

apt-get update -y
apt-get install cloudstack-management mysql-server
```
#### CloudStack Usage and Billing (OPTIONAL)
```
apt-get install cloudstack-usage 
```
#### Configure Database --> next time using sed 
```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
#### Paste below code to mysqld.cnf under [mysqld] block section
```
[mysqld]
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'
```
#### Restart mysql service
```
systemctl restart mysql
```
#### Deploy Database as Root and Then Create "cloud" User with Password "cloud" too
```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:password -i 192.168.101.4
```
#### Configure Primary and Secondary Storage
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```
#### Configure NFS server
```
#sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
#sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
#echo "NEED_STATD=yes" >> /etc/default/nfs-common
#sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
echo 'RPCMOUNTDOPTS="-p 892 --manage-gids"' >> /etc/default/nfs-kernel-server
echo 'STATDOPTS="--port 662 --outgoing-port 2020"' >> /etc/default/nfs-common
echo 'NEED_STATD=yes' >> /etc/default/nfs-common
echo 'RPCRQUOTADOPTS="-p 875"' /etc/default/quota
service nfs-kernel-server restart
```
#### OR copy this to respective file
```
#nfs-kernel-server
# Number of servers to start up
RPCNFSDCOUNT=8

# Runtime priority of server (see nice(1))
RPCNFSDPRIORITY=0

# Options for rpc.mountd.
# If you have a port-based firewall, you might want to set up
# a fixed port here using the --port option. For more information,
# see rpc.mountd(8) or http://wiki.debian.org/SecuringNFS
# To disable NFSv4 on the server, specify '--no-nfs-version 4' here
RPCMOUNTDOPTS="-p 892 --manage-gids"

# Do you want to start the svcgssd daemon? It is only required for Kerberos
# exports. Valid alternatives are "yes" and "no"; the default is "no".
NEED_SVCGSSD=""

# Options for rpc.svcgssd.
RPCSVCGSSDOPTS=""


#nfs-commom
# If you do not set values for the NEED_ options, they will be attempted
# autodetected; this should be sufficient for most people. Valid alternatives
# for the NEED_ options are "yes" and "no".

# Do you want to start the statd daemon? It is not needed for NFSv4.
NEED_STATD=

# Options for rpc.statd.
#   Should rpc.statd listen on a specific port? This is especially useful
#   when you have a port-based firewall. To use a fixed port, set this
#   this variable to a statd argument like: "--port 4000 --outgoing-port 4001".
#   For more information, see rpc.statd(8) or http://wiki.debian.org/SecuringNFS
STATDOPTS="--port 662 --outgoing-port 2020"

# Do you want to start the idmapd daemon? It is only needed for NFSv4.
NEED_IDMAPD=

# Do you want to start the gssd daemon? It is required for Kerberos mounts.
NEED_GSSD=
NEED_STATD=yes

#quota
# Set to "true" if warnquota should be run in cron.daily
run_warnquota=

# Add options to rpc.rquotad here
RPCRQUOTADOPTS="-p 875"
```
### Configure Cloudstack Host with KVM Hypervisor

#### Install KVM Host and Cloudstack Agent
```
apt-get install qemu-kvm cloudstack-agent
```
#### Configure Qemu KVM Virtualisation Management (libvirtd)
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf
```
#### configure default libvirtd configuration
```
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf
echo 'remote_mode="legacy"' >> /etc/libvirt/libvirt.conf
systemctl restart libvirtd
```
#### For Ubuntu 20.04/22.04/24.04 Socket Masking
```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
#### On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead. (ERROR???)
```
#sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
echo LIBVIRTD_ARGS=\"--listen\" >> /etc/default/libvirtd
systemctl restart libvirtd
```
#### More Configuration to Support Docker and Other Services
```
#on certain hosts where you may be running docker and other services, 
#you may need to add the following in /etc/sysctl.conf
#and then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```
#### Generate Unigue Host ID
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```
#### Configure Firewall iptables (OPTIONAL)
```
NETWORK=192.168.101.0/24
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT

apt-get install iptables-persistent
#just answer yes yes
```
#### Disable apparmour on libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```
#### Launch Management Server and Start Your Cloud
```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
#wait until all services (components) running successfully
```
#### Open CloudStack Dashboard 
```
#after management server is UP, proceed to http://192.168.104.21(i.e. the cloudbr0-IP):8080/client 
#and log in using the default credentials - username admin and password password.
#Microsoft Edge browser require secure https protocol thus it might reject the connection without ssl

http://192.168.104.10:8080/client
```
==============================================================
### Enable XRDP (OPTIONAL) ---> Not working for new UBUNTU
#### Reference
```
#https://www.digitalocean.com/community/tutorials/how-to-enable-remote-desktop-protocol-using-xrdp-on-ubuntu-22-04
```
#### Install Desktop XFCE environtment and Remote Desktop XRDP
```
apt update
apt install xfce4 xfce4-goodies -y
apt install xrdp -y
```
#### configure to allow tcp ipv4 listen to 3389. It's a bug only listen to tcp6 --> port=tcp://:3389 --> /etc/xrdp/xrdp.ini
```
netstat -tulpn | grep xrdp

sed -i.bak 's/^\(port=\).*/\1tcp:\/\/:3389/' /etc/xrdp/xrdp.ini
systemctl restart xrdp
systemctl status xrdp
```
## CONTINUE WITH INSTALATION (DASHBOARD)

[![Cloudstack 4.18 Installation on Dashboard](https://img.youtube.com/vi/8ZZeU3vbbl4/0.jpg)](https://www.youtube.com/watch?v=8ZZeU3vbbl4)

## REGISTER ISO AND ADD INSTANCE

[![Register ISO and Add Instance](https://img.youtube.com/vi/y_S3x3tJvCg/0.jpg)](https://www.youtube.com/watch?v=y_S3x3tJvCg)


### Install Additional KVM Host
```
sudo -i
apt update
apt upgrade

mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null

echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

apt-get update -y
```
#### Install KVM Host and Cloudstack Agent
```
apt-get install qemu-kvm cloudstack-agent
```
#### Configure Qemu KVM Virtualisation Management (libvirtd)
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

#configure default libvirtd configuration

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
#### More Configuration to Support Docker and Other Services
```
#on certain hosts where you may be running docker and other services, 
#you may need to add the following in /etc/sysctl.conf
#and then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```
#### Generate Unigue Host ID
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```
#### Firewall disabled (not active) for simplicity
```
ufw status
#make sure it's inactive
```
#### Disable apparmour on libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

#### STORAGE SETUP for Additional PRIMARY and SECONDARY
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```
#### Configure NFS server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```
#### Enable Root Login (PermitRootLogin)
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
```
#### Continue in the cloud management server by adding host

[![Additional KVM Host and Primary-Secondary Storage](https://img.youtube.com/vi/bE7z-D05aCs/0.jpg)](https://www.youtube.com/watch?v=bE7z-D05aCs)

```
#cpu memory increased
#primary and secondary storage increased
```

# HAPPY CLOUDSTACKING!




# Apache CloudStack 4.18 Installation on Ubuntu 22.04

## ALL in ONE - Cloudstack Management and Host in One Machine

==============================================================
### NETWORK/SYSTEM CONFIGURATION AND ADDITIONAL TOOLS

Home network: 192.168.104.0/24
Gateway: 192.168.104.1
Subnet Mask: 255.255.255.0

IP Address for management: 192.168.104.10
IP Address for system: 192.168.104.151-160
IP Address for public: 192.168.104.200-210

### SET STATIC IP ADDRESS FOR MANAGEMENT SERVER
#### Network configuration with netplan
#### Rename all existing configuration by adding .bak
#### maradens@dtecloud:~$ cat /etc/netplan/01-netcfg.yaml
```
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.104.10/24]
      routes:
        - to: default
          via: 192.168.104.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [eno1]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```
#### Apply your network configuration
```
sudo -i
netplan generate
netplan apply
reboot
```
#### Update Your System and Install Some Useful Tools
```
apt update & upgrade
apt install htop lynx duf -y
apt install bridge-utils
```
#### Configure LVM (OPTIONAL)
```
#not required unless logical volume is used
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```
#### Install SSH Server and Others Tools 
```
apt-get install openntpd openssh-server sudo vim htop tar -y
apt-get install intel-microcode -y
passwd root
#change it to Pa$$w0rd
```
#### Enable Root Login (PermitRootLogin)
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
```
#### Set Timezone 
```
timedatectl set-timezone Asia/Jakarta
```
==============================================================
## APACHE CLOUDSTACK INSTALLATION
### CloudStack Management Server and Everything in one machine

#### Reference
```
https://rohityadav.cloud/blog/cloudstack-kvm/
```

#### CloudStack Management Server Setup from SHAPEBLUE ACS 4.18
```
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

#update and install cloudstack management and mysql
#it takes very long time

apt-get update -y
apt-get install cloudstack-management mysql-server
```
#### CloudStack Usage and Billing (OPTIONAL)
```
apt-get install cloudstack-usage 
```
#### Configure Database --> next time using sed 
```
nano /etc/mysql/mysql.conf.d/mysqld.cnf

#paste below code to mysqld.cnf under [mysqld] block section

[mysqld]
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'

#restart mysql service

systemctl restart mysql
```
#### Deploy Database as Root and Then Create "cloud" User with Password "cloud" too
```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.104.10
```
#### Configure Primary and Secondary Storage
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```
#### Configure NFS server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```
### Configure Cloudstack Host with KVM Hypervisor

#### Install KVM Host and Cloudstack Agent
```
apt-get install qemu-kvm cloudstack-agent
```
#### Configure Qemu KVM Virtualisation Management (libvirtd)
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

#configure default libvirtd configuration

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
#### More Configuration to Support Docker and Other Services
```
#on certain hosts where you may be running docker and other services, 
#you may need to add the following in /etc/sysctl.conf
#and then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```
#### Generate Unigue Host ID
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```
#### Configure Firewall iptables (OPTIONAL)
```
NETWORK=192.168.101.0/24
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT

apt-get install iptables-persistent
#just answer yes yes
```
#### Disable apparmour on libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```
#### Launch Management Server and Start Your Cloud
```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
#wait until all services (components) running successfully
```
#### Open CloudStack Dashboard 
```
#after management server is UP, proceed to http://192.168.104.21(i.e. the cloudbr0-IP):8080/client 
#and log in using the default credentials - username admin and password password.
#Microsoft Edge browser require secure https protocol thus it might reject the connection without ssl

http://192.168.104.10:8080/client
```
==============================================================
### Enable XRDP (OPTIONAL) ---> Not working for new UBUNTU
#### Reference
```
#https://www.digitalocean.com/community/tutorials/how-to-enable-remote-desktop-protocol-using-xrdp-on-ubuntu-22-04
```
#### Install Desktop XFCE environtment and Remote Desktop XRDP
```
apt update
apt install xfce4 xfce4-goodies -y
apt install xrdp -y
```
#### configure to allow tcp ipv4 listen to 3389. It's a bug only listen to tcp6 --> port=tcp://:3389 --> /etc/xrdp/xrdp.ini
```
netstat -tulpn | grep xrdp

sed -i.bak 's/^\(port=\).*/\1tcp:\/\/:3389/' /etc/xrdp/xrdp.ini
systemctl restart xrdp
systemctl status xrdp
```
## CONTINUE WITH INSTALATION (DASHBOARD)

[![Cloudstack 4.18 Installation on Dashboard](https://img.youtube.com/vi/8ZZeU3vbbl4/0.jpg)](https://www.youtube.com/watch?v=8ZZeU3vbbl4)

## REGISTER ISO AND ADD INSTANCE

[![Register ISO and Add Instance](https://img.youtube.com/vi/y_S3x3tJvCg/0.jpg)](https://www.youtube.com/watch?v=y_S3x3tJvCg)


### Install Additional KVM Host
```
sudo -i
apt update
apt upgrade

mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null

echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

apt-get update -y
```
#### Install KVM Host and Cloudstack Agent
```
apt-get install qemu-kvm cloudstack-agent
```
#### Configure Qemu KVM Virtualisation Management (libvirtd)
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

#configure default libvirtd configuration

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
#### More Configuration to Support Docker and Other Services
```
#on certain hosts where you may be running docker and other services, 
#you may need to add the following in /etc/sysctl.conf
#and then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```
#### Generate Unigue Host ID
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```
#### Firewall disabled (not active) for simplicity
```
ufw status
#make sure it's inactive
```
#### Disable apparmour on libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

#### STORAGE SETUP for Additional PRIMARY and SECONDARY
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```
#### Configure NFS server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```
#### Enable Root Login (PermitRootLogin)
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
```
#### Continue in the cloud management server by adding host

[![Additional KVM Host and Primary-Secondary Storage](https://img.youtube.com/vi/bE7z-D05aCs/0.jpg)](https://www.youtube.com/watch?v=bE7z-D05aCs)

```
#cpu memory increased
#primary and secondary storage increased
```
# Introduction
# HAPPY CLOUDSTACKING!

