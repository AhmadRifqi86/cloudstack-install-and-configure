# Install and Configure Apache Cloudstack Private Cloud
![image](https://github.com/AhmadRifqi86/cloudstack-install-and-configure/assets/111260795/99bef561-43e0-400d-8d90-7dba43054c20)

## Introduction

### What is Cloudstack

Apache CloudStack is an open-source cloud computing software designed to deploy and manage large networks of virtual machines (VMs). As an Infrastructure-as-a-Service (IaaS) platform, CloudStack provides a comprehensive set of features to build and manage scalable cloud environments. It supports multiple hypervisors, extensive API capabilities, and a rich set of tools for cloud administrators.

### Architecture Reference

![image](https://github.com/AhmadRifqi86/cloudstack-install-and-configure/assets/111260795/68ea9d69-1118-4500-8825-8c099f69c4ce)

### Dependencies

#### Mysql

MySQL is an open-source relational database management system (RDBMS) that uses Structured Query Language (SQL) for accessing, adding, and managing data in a database. Developed by Oracle Corporation, MySQL is widely used for web applications and embedded database solutions due to its reliability, scalability, and ease of use.

Apache CloudStack is an open-source cloud computing software for creating, managing, and deploying infrastructure cloud services. MySQL plays a crucial role in the functioning of CloudStack by serving as the primary database backend for storing critical information such as users, computing node information and storage array information

#### KVM




## Computing Environment Setup

### Hardware Requirement

```
CPU : Intel Core i5 gen 8
RAM : 24 GB
Storage : 250GB
Network : Ethernet 100GB/s
Operating System : Ubuntu Server 22.04
```

### Network Address

```
Network Address : 192.168.104.0/24
Host IP address : 192.168.104.24/24
Gateway : 192.168.104.1
Management IP :
System IP :
Public IP :
```

## Configure Network 

### Edit the network configuration file under /netplan directory

```
cd /etc/netplan
sudo nano ./0*.yaml
```

The opened file will look like this:
```
# This is the network config written by 'subiquity'
network:
  version: 2
  ethernets:
    enp0s3:                             # Interface Name
      addresses: [10.1.1.8/24]          # Ip Address
      gateway4: 10.1.1.1                # Gateway
      nameservers:
        addresses: [10.1.1.1,8.8.8.8]   # DNS Server
```

Edit the file

```
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.104.24/24]  #Your host IP address
      routes:
        - to: default
          via: 192.168.104.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [enp0s3]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

### Apply Network Configuration

```
sudo -i
netplan generate
netplan apply
reboot
```

> Note : You may encounter some error during this step, make sure you use "space" instead of "tab" when modifying network configuration file

### Test the Network, make sure the configuration applied

```
ifconfig     #check the ip address and existing interface
ping -c 20 google.com  #make sure you could connect to the internet
```

> Note : if you can't ping the google.com, try to ping your gateway and 8.8.8.8
> This step will tell you the connection problem between your computer and internet, make sure there's no problem with the internet as you will install some package from internet

### Login to the system as root user

```
su -
```

### Installing Hardware resource monitoring tools

```
apt update -y
apt upgrade -y
apt install htop lynx duf -y
apt install bridge-utils
```

* htop is CPU usage monitoring tool
* duf is Disk usage monitoring tool
* lynx is CLI based web browser

### Configure LVM (optional)

```
#not required unless logical volume is used
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

### Installing Network services and text editor

```
apt-get install openntpd openssh-server sudo vim tar -y
apt-get install intel-microcode -y
passwd root
#change it to Pa$$w0rd
```

* openntpd is NTP client to synchronize time between host and entire internet
* openssh-server is an SSH server to enable remote access
* vim is a text editor
* tar is a tool to compress and decompress file. Tar is used to decompress most of the downloaded file
* intel-microcode is a set of procedures written in x86 assembly to increase low level modularity

### Enable SSH root login

```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
#or
systemctl restart sshd.service
```

## Cloudstack Installation (Controller and Compute Node at the Same Host)


### Importing cloudstack repositories key

```
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list

```

### Installing Cloudstack and Mysql Server

```
apt-get update -y
apt-get install cloudstack-management mysql-server
```

> Note : This process will take a long time

### Configure mysql

#### Open mysql config file

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

#### Copy these lines under [mysqld] section

```
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

### Deploy Database as Root and Then Create "cloud" User with Password "cloud" too

```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.104.24
```

### Configure Primary and Secondary Storage

```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

### Configure NFS Server

```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

## Configure Cloudstack Host with KVM Hypervisor

### Install KVM and Cloudstack Agent

```
apt-get install qemu-kvm cloudstack-agent -y
```

### Configure KVM Virtualization Management

#### Change some lines

```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
```

#### Add some lines

```
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf
```

#### Restart libvirtd

```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

libvirtd is a daemon that provides management of virtual machines (VMs), virtual networks, and storage for various virtualization technologies, such as KVM, QEMU, Xen, and others. It is part of the libvirt project, which offers a toolkit for managing virtualization platforms. The primary purpose of libvirtd is to provide a consistent and secure API for managing VMs and associated resources, regardless of the underlying virtualization technology.


### Configuration to Support Docker and Other Services

```
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

### Generate Unique Host ID

```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

### Configure Iptables Firewall and Make it persistent

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

This step ensuring all service port used by cloudstack didn't blocked by firewall and accessible by network

### Disable apparmour on libvirtd

```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

### Launch management Server

```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log #if you want to troubleshoot 
#wait until all services (components) running successfully
```

### Open web browser and type

```
http://<YOUR_IP_ADDRESS>:8080
```

Example:

```
http://192.168.104.24:8080
```

### You should see the cloudstack dashboard

![image](https://github.com/AhmadRifqi86/cloudstack-install-and-configure/assets/111260795/1d99f652-efec-4d4d-857e-dec1122ed865)




