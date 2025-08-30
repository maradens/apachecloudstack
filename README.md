# Apache CloudStack 4.20 Installation on Ubuntu 24.04

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

# HAPPY CLOUDSTACKING!
