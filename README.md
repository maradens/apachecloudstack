# Apache CloudStack 4.18 Installation
## Installation CloudStack 4.18 on Ubuntu 22.04
Home network 192.168.101.0/24

## SET STATIC IP ADDRESS
  
## Network configuration with netplan
#rename all existing configuration by adding .bak
maradens@dtecloud:~$ cat /etc/netplan/01-netcfg.yaml
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
      addresses: [192.168.104.21/24]
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

## Apply your network configuration
```
sudo -i
netplan generate
netplan apply
reboot
```
## Update your system and install some useful tools
```
apt update & upgrade
apt install htop lynx duf -y
apt install bridge-utils
```
## CONFIGURE LVM +100%FREE (OPTIONAL)
```
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

## Install ssh server and others tool if not yet present
```
apt-get install openntpd openssh-server sudo vim htop tar -y
apt-get install intel-microcode -y
passwd root
```
#change to Pa$$w0rd

## SET TIME
timedatectl set-timezone Asia/Jakarta

#enable Root Login
#PermitRootLogin yes
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
```

## APACHE CLOUDSTACK INSTALLATION
## CloudStack Management Server and Everything in one machine
https://rohityadav.cloud/blog/cloudstack-kvm/

## CloudStack Management Server Setup from SHAPEBLUE ACS 4.18
```
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null

echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
```
```
apt-get update -y
apt-get install cloudstack-management mysql-server
```
#CloudStack usage and billing (optional)
```
apt-get install cloudstack-usage 
```
#Configure database --> next time using sed 
```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
paste
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
restart
```
systemctl restart mysql
```
#deploy database as root and then create cloud user with password cloud too
```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.104.21
```
#Storage Setup
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

#Configure NFS server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

## Setup KVM
#Setup KVM host and cloudstack agent
```
apt-get install qemu-kvm cloudstack-agent
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf
```
#On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.
```
sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
```

#Configure default libvirtd config:
```
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

#On certain hosts where you may be running docker and other services, you may need to add the following in /etc/sysctl.conf and then run sysctl -p: --> /etc/sysctl.conf
```
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```
#generate host id
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

#configure firewall (OPTIONAL)
# configure iptables
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
```
#yes yes

# Disable apparmour on libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

#Enable XRDP --> OPTIONAL
#https://www.digitalocean.com/community/tutorials/how-to-enable-remote-desktop-protocol-using-xrdp-on-ubuntu-22-04
```
apt update
apt install xfce4 xfce4-goodies -y
apt install xrdp -y
```

#configure to allow tcp ipv4 listen to 3389. It's a bug only listen to tcp6 --> port=tcp://:3389 --> /etc/xrdp/xrdp.ini

```
netstat -tulpn | grep xrdp

sed -i.bak 's/^\(port=\).*/\1tcp:\/\/:3389/' /etc/xrdp/xrdp.ini
systemctl restart xrdp
systemctl status xrdp
```

#Launch Management Server
#Start your cloud:
```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
```

#After management server is UP, proceed to http://192.168.101.11(i.e. the cloudbr0-IP):8080/client and log in using the default credentials - username admin and password password.
```
http://192.168.101.11:8080/client
```
## CONTINUE WITH INSTALATION

![CloudStack](/images/cloudstack.png?raw=true "Apache CloudStack 4.18")

# ADDITIONAL KVM HOST
```
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null

echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

apt-get update -y
```

## INSTALL KVM and CLOUDSTACK AGENT
### not sure whether cloudstack agent is needed but I installed anyway
### may next time I will confirm by installing only qemu-kvm
```
apt-get install qemu-kvm cloudstack-agent
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf
```
## On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.
```
sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
```
## Configure default libvirtd config:
```
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

## On certain hosts where you may be running docker and other services, you may need to add the following in /etc/sysctl.conf and then run sysctl -p: --> /etc/sysctl.conf
```
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

## Generate unique host id with uuidenerate host id
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

## Firewall disabled (not active)

## Disable AppArmor
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

## CONTINUE IN MANAGEMENT SERVER BY ADDING HOST
## cpu memory increased

# STORAGE SETUP for Additional PRIMARY and SECONDARY
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

## Configure NFS server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```
## Primary and Secondary Storage increased.

# HAPPY CLOUDSTACKING!
