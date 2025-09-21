# Apache CloudStack Installation Guide

This README provides a comprehensive, step-by-step guide for installing Apache CloudStack on a single machine, which acts as both the **Management Server** and the **KVM Host**. This "all-in-one" setup is perfect for testing, learning, and small-scale deployments. The guide covers two different versions: **CloudStack 4.20 on Ubuntu 24.04** and **CloudStack 4.18 on Ubuntu 22.04**.

## 📖 Table of Contents

  * [Apache CloudStack 4.20 on Ubuntu 24.04](#Apache-CloudStack-420-on-Ubuntu-2404)
      * [System and Network Configuration](#System-and-Network-Configuration)
      * [CloudStack Management Server Installation](#CloudStack-Management-Server-Installation)
      * [CloudStack KVM Host Configuration](#CloudStack-KVM-Host-Configuration)
      * [Finalizing the Installation](#Finalizing-the-Installation)
      * [Dashboard & Next Steps](#Dashboard--Next-Steps)
  * [Apache CloudStack 4.18 on Ubuntu 22.04](#Apache-CloudStack-418-on-Ubuntu-2204)
      * [System and Network Configuration ACS 4.18](#System-and-Network-Configuration-ACS-418)
      * [CloudStack Management Server Installation ACS 4.18](#CloudStack-Management-Server-Installation-ACS-418)
      * [CloudStack KVM Host Configuration ACS 4.18](https://www.google.com/search?q=%23cloudstack-kvm-host-configuration-1)
      * [Finalizing the Installation ACS 4.18](#Finalizing-the-Installation-ACS-418)
      * [Dashboard & Next Steps ACS 4.18](#Dashboard--Next-Steps-ACS-418)
  * [Adding an Additional KVM Host ACS 4.18](#Adding-an-Additional-KVM-Host-ACS-418)

-----

## Apache CloudStack 4.20 on Ubuntu 24.04

This section details the installation of Apache CloudStack 4.20 on a single Ubuntu 24.04 machine.

### System and Network Configuration

First, you need to configure your system and network settings. The example uses a home network of **192.168.101.0/24** with a gateway at **192.168.101.1**.

**Network Details:**  
- **Home Network:** 192.168.101.0/24
- **Gateway:** 192.168.101.1
- **Management IP:** 192.168.101.4
- **System IPs:** 192.168.101.51-70
- **Public IPs:** 192.168.101.71-90

**1. Set a Static IP Address**
The Management Server needs a static IP address. This guide uses `netplan` for network configuration.

  * Rename any existing configuration files in `/etc/netplan/` by adding a `.bak` extension.
  * Create a new configuration file, for example, `/etc/netplan/01-netcfg.yaml`, and add the following content. This configuration sets up a bridge (`cloudbr0`) with a static IP and connects it to the physical interface (`eno1`).
    ```yaml
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
        addresses: [192.168.101.4/24]
        routes:
         - to: default
           via: 192.168.101.1
        nameservers:
          addresses: [1.1.1.1, 8.8.8.8]
        interfaces: [eno1]
        dhcp4: false
        dhcp6: false
        parameters:
          stp: false
          forward-delay: 0
    ```
    
  * Apply the new configuration and reboot the system.
    ```bash
    sudo -i
    netplan generate
    netplan apply
    reboot
    ```

**2. Update System & Install Tools**
Update your system and install essential packages, including `bridge-utils` for network bridging and an SSH server for remote access.

```bash
apt update & upgrade
apt install htop lynx duf -y
apt install bridge-utils
```

**3. Configure LVM (Optional)**
This step is only necessary if you are using Logical Volume Management and need to extend the disk space.

```bash
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

**4. Install SSH Server and Others Tools**
These commands install an SSH server for remote access terminal and other tools for monitoring and configuring.

```bash
apt-get install openntpd openssh-server sudo vim htop tar -y
apt-get install intel-microcode -y
passwd root
# Set a new password for the root user.
```

**5. Enable Root Login via SSH**
For easier management, enable root login. This is not recommended for production environments.
This is required by cloudstack management when adding a new host.

```bash
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
service ssh restart
```

**6. Set Timezone**
Ensure your system time is synchronized, which is critical for a well-functioning server environment.

```bash
timedatectl set-timezone Asia/Jakarta
```

-----

### CloudStack Management Server Installation

This section covers the setup of the Apache CloudStack Management Server and its dependencies.

**1. Add CloudStack Repository**
Add the ShapeBlue repository for CloudStack 4.20 to your system's package list.

```bash
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.20 / > /etc/apt/sources.list.d/cloudstack.list
apt-get update -y
```

**2. Install Management Server & MySQL**
Install the core CloudStack and MySQL server packages. This may take a considerable amount of time.

```bash
apt-get install cloudstack-management mysql-server
```

**3. Configure MySQL Database**
Modify the MySQL configuration file to optimize it for CloudStack. You can either use `nano` or the provided `sed` command.

```bash
# Using nano
nano /etc/mysql/mysql.conf.d/mysqld.cnf

# Paste the following lines under the [mysqld] block:
[mysqld]
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'

# Restart the MySQL service
systemctl restart mysql
```

**4. Deploy CloudStack Database**
Initialize the CloudStack database, creating a `cloud` user with the password `cloud`. Replace `root:password` with your MySQL root credentials.

```bash
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:password -i 192.168.101.4
```

**5. Configure Primary & Secondary Storage**
Set up NFS (Network File System) to handle primary and secondary storage. This is where your virtual machine disks and templates will be stored.

```bash
apt-get install nfs-kernel-server quota
echo "/export *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

**6. Configure NFS Server**
To configure the NFS server with specific ports for RPC services, modify the relevant configuration files located in `/etc/default`. You can either edit these files directly or replace their entire contents with the provided configuration shown below.

Paste below configuration to `/etc/default/nfs-kernel-server`
```bash
# Configuration for nfs-kernel-server
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
```

Paste below configuration to `/etc/default/nfs-common`
```bash
# Configuration for nfs-commom
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
```

Paste below configuration to `/etc/default/quota`
```bash
# Configuration for quota
# Set to "true" if warnquota should be run in cron.daily
run_warnquota=

# Add options to rpc.rquotad here
RPCRQUOTADOPTS="-p 875"
```

**7. CloudStack Usage and Billing (Optional)**
The `cloudstack-usage` package in Apache CloudStack installs the Usage Server, which is an optional but powerful component designed to track and report resource consumption across your cloud environment.

```bash
apt-get install cloudstack-usage 
```

-----

### CloudStack KVM Host Configuration

This section prepares the machine to function as a KVM hypervisor host for CloudStack.

**1. Install KVM Host & CloudStack Agent**
Install the `qemu-kvm` hypervisor and the `cloudstack-agent` package that allows CloudStack to manage this host.

```bash
apt-get install qemu-kvm cloudstack-agent
```

**2. Configure Libvirt**
Edit the libvirt configuration to allow it to listen for connections from the Management Server.

```bash
# Allow VNC connections from all IPs
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# Add the following lines to libvirtd.conf to enable TCP listening
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf
echo 'remote_mode="legacy"' >> /etc/libvirt/libvirt.conf
systemctl restart libvirtd

# Mask libvirt sockets and restart the service to apply changes
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd

# Enable Remote Libvirt Access for CloudStack KVM Integration (Remote Access)
echo LIBVIRTD_ARGS=\"--listen\" >> /etc/default/libvirtd
systemctl restart libvirtd
```

**3. Docker & Service Compatibility**
Add bridge-related settings to `sysctl.conf` to prevent conflicts with other services like Docker.

```bash
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

**4. Generate a Unique Host ID**
Install the `uuid` package and generate a unique ID for the host.

```bash
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

**5. Disable AppArmor**
Disable AppArmor profiles for libvirt to prevent potential permission issues.

```bash
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

**6. Configure the Firewall with iptables (Optional)**
This step is critical if you have a firewall enabled. It opens the necessary ports for CloudStack. 

```bash
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
# Answer with yes two times
```

-----

### Finalizing the Installation

**1. Launch the Management Server**
Run the final command to set up the Management Server and start the service.
This command completes the CloudStack setup and starts the management server. The tail command lets you monitor the log file to see when the services are fully up and running.

```bash
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
# Wait for all services to start successfully
```

**2. Open the CloudStack Dashboard**
Once the services are running, you can access the CloudStack dashboard in your web browser. The default credentials are `username: admin` and `password: password`.

http://192.168.101.4:8080/client

## Dashboard & Next Steps

Coming soon...


-----

      * [System and Network Configuration ACS 4.18](#System-and-Network-Configuration-ACS-418)
      * [CloudStack Management Server Installation ACS 4.18](#CloudStack-Management-Server-Installation-ACS-418)
      * [CloudStack KVM Host Configuration ACS 4.18](https://www.google.com/search?q=%23cloudstack-kvm-host-configuration-1)
      * [Finalizing the Installation ACS 4.18](#Finalizing-the-Installation-ACS-418)
      * [Dashboard & Next Steps ACS 4.18](#Dashboard--Next-Steps-ACS-418)
      

# Apache CloudStack 4.18 Installation on Ubuntu 22.04

==============================================================
## System and Network Configuration ACS 4.18
ALL in ONE - Cloudstack Management and Host in One Machine
NETWORK/SYSTEM CONFIGURATION AND ADDITIONAL TOOLS

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
## CloudStack Management Server Installation ACS 4.18
APACHE CLOUDSTACK INSTALLATION  
CloudStack Management Server and Everything in one machine

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
## CloudStack KVM Host Configuration ACS 4.18
Configure Cloudstack Host with KVM Hypervisor  
Install KVM Host and Cloudstack Agent  

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

## Finalizing the Installation ACS 4.18
Launch Management Server and Start Your Cloud

```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
#wait until all services (components) running successfully
```

Open CloudStack Dashboard 
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
## Dashboard & Next Steps ACS 4.18

## CONTINUE WITH INSTALATION (DASHBOARD)

[![Cloudstack 4.18 Installation on Dashboard](https://img.youtube.com/vi/8ZZeU3vbbl4/0.jpg)](https://www.youtube.com/watch?v=8ZZeU3vbbl4)

## REGISTER ISO AND ADD INSTANCE

[![Register ISO and Add Instance](https://img.youtube.com/vi/y_S3x3tJvCg/0.jpg)](https://www.youtube.com/watch?v=y_S3x3tJvCg)

## Adding an Additional KVM Host ACS 4.18
Install Additional KVM Host
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

## Research & Development {#research-dev}


**6. Continue on the CloudStack Dashboard**
Once the new host is prepared, you must add it to your CloudStack environment through the web UI. This process involves adding the host to a zone, cluster, and finally, adding the primary and secondary storage.

This video demonstrates how to add an additional KVM host and storage to your CloudStack environment.
