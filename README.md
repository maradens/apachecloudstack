I can definitely help you improve your README. The revised version below is more detailed and user-friendly, providing a clearer step-by-step guide for a general audience. It includes a table of contents, descriptive headings, and explanations for each command to make the installation process more understandable.

# Apache CloudStack Installation Guide

This README provides a comprehensive, step-by-step guide for installing Apache CloudStack on a single machine, which acts as both the **Management Server** and the **KVM Host**. This "all-in-one" setup is perfect for testing, learning, and small-scale deployments. The guide covers two different versions: **CloudStack 4.20 on Ubuntu 24.04** and **CloudStack 4.18 on Ubuntu 22.04**.

## 📖 Table of Contents

  * [Apache CloudStack 4.20 on Ubuntu 24.04](#Apache-CloudStack-420-on-Ubuntu-2404)
      * [System and Network Configuration](System-and-Network-Configuration)
      * [CloudStack Management Server Installation](https://www.google.com/search?q=%23cloudstack-management-server-installation)
      * [CloudStack KVM Host Configuration](https://www.google.com/search?q=%23cloudstack-kvm-host-configuration)
      * [Finalizing the Installation](https://www.google.com/search?q=%23finalizing-the-installation)
      * [Dashboard & Next Steps](https://www.google.com/search?q=%23dashboard--next-steps)
  * [Apache CloudStack 4.18 on Ubuntu 22.04](https://www.google.com/search?q=%23apache-cloudstack-418-on-ubuntu-2204)
      * [System and Network Configuration](https://www.google.com/search?q=%23system-and-network-configuration-1)
      * [CloudStack Management Server Installation](https://www.google.com/search?q=%23cloudstack-management-server-installation-1)
      * [CloudStack KVM Host Configuration](https://www.google.com/search?q=%23cloudstack-kvm-host-configuration-1)
      * [Finalizing the Installation](https://www.google.com/search?q=%23finalizing-the-installation-1)
      * [Dashboard & Next Steps](https://www.google.com/search?q=%23dashboard--next-steps-1)
  * [Adding an Additional KVM Host](https://www.google.com/search?q=%23adding-an-additional-kvm-host)

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
  * Create a new configuration file, for example, `/etc/netplan/01-netcfg.yaml`, and add the following content. This configuration sets up a bridge (`cloudbr0`) with a static IP and connects it to the physical interface (`eno1`). It also defines two VLANs (`vlan.2101` and `vlan.262`).
    ```yaml
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
apt update && apt upgrade
apt install htop lynx duf -y
apt install bridge-utils openntpd openssh-server sudo vim htop tar intel-microcode -y
```

**3. Enable Root Login via SSH**
For easier management, enable root login. This is not recommended for production environments.

```bash
passwd root
# Enter your desired root password here
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
service ssh restart
```

**4. Set Timezone**
Ensure your system time is synchronized.

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
Modify the NFS configuration to set specific ports for RPC services. This can be done by editing the files directly or by appending the provided commands.

```bash
echo 'RPCMOUNTDOPTS="-p 892 --manage-gids"' >> /etc/default/nfs-kernel-server
echo 'STATDOPTS="--port 662 --outgoing-port 2020"' >> /etc/default/nfs-common
echo 'NEED_STATD=yes' >> /etc/default/nfs-common
echo 'RPCRQUOTADOPTS="-p 875"' /etc/default/quota
service nfs-kernel-server restart
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

# Mask libvirt sockets and restart the service to apply changes
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
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

-----

### Finalizing the Installation

**1. Launch the Management Server**
Run the final command to set up the Management Server and start the service.

```bash
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
# Wait for all services to start successfully
```

**2. Open the CloudStack Dashboard**
Once the services are running, you can access the CloudStack dashboard in your web browser. The default credentials are `username: admin` and `password: password`.

```
http://192.168.104.10:8080/client
```

-----

## Apache CloudStack 4.18 on Ubuntu 22.04

This section outlines the installation process for Apache CloudStack 4.18 on a single Ubuntu 22.04 machine. The steps are similar to the 4.20 guide but with specific version commands and configurations.

### System and Network Configuration

The example uses a home network of **192.168.104.0/24** with a gateway at **192.168.104.1**.

**1. Set a Static IP Address**
Configure a static IP for the Management Server using `netplan`.

  * Create `/etc/netplan/01-netcfg.yaml` with the following content:
    ```yaml
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
  * Apply the configuration and reboot.
    ```bash
    sudo -i
    netplan generate
    netplan apply
    reboot
    ```

**2. Update System & Install Tools**

```bash
apt update && upgrade
apt install htop lynx duf -y
apt install bridge-utils openntpd openssh-server sudo vim htop tar intel-microcode -y
```

**3. Enable Root Login**

```bash
passwd root
# Enter your desired root password here
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
service ssh restart
```

**4. Set Timezone**

```bash
timedatectl set-timezone Asia/Jakarta
```

-----

### CloudStack Management Server Installation

**1. Add CloudStack Repository**
Add the repository for CloudStack 4.18.

```bash
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
apt-get update -y
```

**2. Install Management Server & MySQL**

```bash
apt-get install cloudstack-management mysql-server
```

**3. Configure MySQL Database**
Modify the MySQL configuration file.

```bash
# Add the provided lines to the [mysqld] section
nano /etc/mysql/mysql.conf.d/mysqld.cnf

# Restart the MySQL service
systemctl restart mysql
```

**4. Deploy CloudStack Database**
Initialize the database for version 4.18.

```bash
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.104.10
```

**5. Configure Primary & Secondary Storage**

```bash
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

**6. Configure NFS Server**

```bash
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

-----

### CloudStack KVM Host Configuration

**1. Install KVM Host & CloudStack Agent**

```bash
apt-get install qemu-kvm cloudstack-agent
```

**2. Configure Libvirt**

```bash
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf
sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd

echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

**3. Docker & Service Compatibility**

```bash
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

**4. Generate a Unique Host ID**

```bash
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

**5. Disable AppArmor**

```bash
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

-----

### Finalizing the Installation

**1. Launch the Management Server**

```bash
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
```

**2. Open the CloudStack Dashboard**

```
http://192.168.104.10:8080/client
```

-----

## Adding an Additional KVM Host

This section explains how to add another KVM host to an existing CloudStack environment. These steps should be performed on the **new host machine**.

**1. Prepare the New Host**

  * Update the system and install necessary tools.
    ```bash
    sudo -i
    apt update && upgrade
    ```
  * Add the CloudStack repository for the appropriate version (e.g., 4.18).
    ```bash
    mkdir -p /etc/apt/keyrings
    wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
    echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
    apt-get update -y
    ```
  * Install the KVM host and CloudStack agent.
    ```bash
    apt-get install qemu-kvm cloudstack-agent
    ```

**2. Configure Libvirt on the New Host**
Follow the same libvirt configuration steps as for the initial all-in-one setup.

```bash
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf
sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

**3. Enable Root Login**

```bash
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
service ssh restart
```

**4. Add Storage from the Management Server**
Configure the NFS shares on the new host to connect to the primary and secondary storage volumes located on the Management Server.

```bash
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

**5. Disable AppArmor**

```bash
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

**6. Continue on the CloudStack Dashboard**
Once the new host is prepared, you must add it to your CloudStack environment through the web UI. This process involves adding the host to a zone, cluster, and finally, adding the primary and secondary storage.

This video demonstrates how to add an additional KVM host and storage to your CloudStack environment.
