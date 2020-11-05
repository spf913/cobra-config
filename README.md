# cobra-config
```bash

# DISABLE SELINUX FIREWALL ON ALL MASTER AND SLAVE NODE

sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config

# Setup Master and Slave Node Hostname

############################  /etc/hosts #########################################
# set hostname (from master)
hostnamectl --static set-hostname <master_ip>
# set hostname (from slave1)
hostnamectl --static set-hostname <slave1_ip>
# set hostname (from slave2)
hostnamectl --static set-hostname <slave2_ip>

# REBOOT ALL INSTENCES
reboot
sestatus



# ALL CHANGES ONLY NEED TO APPLY ON MASTER NODES
cat /var/lib/zookeeper/myid             (1) (for single master)  # for multi master master1(1), master2(2),  master3(3)
cat /etc/zookeeper/conf/zoo.cfg       server.1=<master_IP>:2888:3888 (for single master)   for multi master: server.1=, server.2=, server.3=
cat /etc/mesos/zk                     # for single master   zk://<master_IP>:2181/mesos   ,,,,,,,  #for multi master    zk=zk://<master1_IP>:2181,<master2_IP>:2181,<master3_IP>:2181/mesos
cat /etc/mesos-master/quorum          #for single master (1)  # for multimaster the value is (2)


# FOR SLAVE NODES SAME CONFIG WILL APPLY ALL THE TIME
# SEE "WORKER01" CONFIG SECTION

# for all mesos master install (master01)
mesos-master,mesos-slave,marathon,zookeeper
# enable mesos-master,mesos-slave,marathon,zookeeper on all master node

# for all mesos slave install (slave01, slave02, slave03)
mesos-master,mesos-slave,marathon,zookeeper
# enable mesos-slave,marathon,zookeeper on all slave node, disable mesos-master



###########
### settings for (SINGLE) MASTER
## NODE
########################

# Apply to All nodes (master and slave) of the Cluster

cat <<EOF | sudo tee /etc/hosts > /dev/null
# localhost
127.0.0.1 localhost localhost.localdomain

# Master and Slave IP's
104.248.236.22 104.248.236.22
142.93.249.68 142.93.249.68
142.93.7.31 142.93.7.31
EOF


cat /etc/hosts


# Fix IPROUTE

cat <<EOF | sudo tee /etc/rc.local > /dev/null
defrt=`ip route | grep "^default" | head -1`
ip route change $defrt initcwnd 10
EOF


# Restart Network Manager
sudo systemctl restart NetworkManager.service

# Fix grub for Docker

cat <<EOF | sudo tee /etc/default/grub > /dev/null
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="console=ttyS0,115200n8 no_timer_check net.ifnames=0 crashkernel=auto cgroup_enable=memory swapaccount=1"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
EOF

# Reload GRUB
grub2-mkconfig -o "$(readlink -e /etc/default/grub)"


# TWEAK sysctl.conf (apply this to all MASTER and SLAVE nodes)

cat <<EOF | sudo tee /etc/sysctl.d/00-sysctl.conf > /dev/null
#############################################################################################
# Tweak virtual memory
#############################################################################################

# Default: 30
# 0 - Never swap under any circumstances.
# 1 - Do not swap unless there is an out-of-memory (OOM) condition.
# "Performance Scalability of a Multi-Core Web Server", Nov 2007
# Bryan Veal and Annie Foong, Intel Corporation, Page 4/10
# /etc/sysctl.conf
# enable bbr from google
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
vm.swappiness = 1

# vm.dirty_background_ratio is used to adjust how the kernel handles dirty pages that must be flushed to disk.
# Default value is 10.
# The value is a percentage of the total amount of system memory, and setting this value to 5 is appropriate in many situations.
# This setting should not be set to zero.
vm.dirty_background_ratio = 5

# The total number of dirty pages that are allowed before the kernel forces synchronous operations to flush them to disk
# can also be increased by changing the value of vm.dirty_ratio, increasing it to above the default of 30 (also a percentage of total system memory)
# vm.dirty_ratio value in-between 60 and 80 is a reasonable number.
vm.dirty_ratio = 60

# vm.max_map_count will calculate the current number of memory mapped files.
# The minimum value for mmap limit (vm.max_map_count) is the number of open files ulimit (cat /proc/sys/fs/file-max).
# map_count should be around 1 per 128 KB of system memory. Therefore, max_map_count will be 262144 on a 32 GB system.
# Default: 65530
vm.max_map_count = 2097152

#############################################################################################
# Tweak file handles
#############################################################################################

# Increases the size of file handles and inode cache and restricts core dumps.
fs.file-max = 2097152
fs.suid_dumpable = 0

#############################################################################################
# Tweak network settings
#############################################################################################

# Default amount of memory allocated for the send and receive buffers for each socket.
# This will significantly increase performance for large transfers.
net.core.wmem_default = 25165824
net.core.rmem_default = 25165824

# Maximum amount of memory allocated for the send and receive buffers for each socket.
# This will significantly increase performance for large transfers.
net.core.wmem_max = 25165824
net.core.rmem_max = 25165824

# In addition to the socket settings, the send and receive buffer sizes for
# TCP sockets must be set separately using the net.ipv4.tcp_wmem and net.ipv4.tcp_rmem parameters.
# These are set using three space-separated integers that specify the minimum, default, and maximum sizes, respectively.
# The maximum size cannot be larger than the values specified for all sockets using net.core.wmem_max and net.core.rmem_max.
# A reasonable setting is a 4 KiB minimum, 64 KiB default, and 2 MiB maximum buffer.
net.ipv4.tcp_wmem = 20480 12582912 25165824
net.ipv4.tcp_rmem = 20480 12582912 25165824

# Increase the maximum total buffer-space allocatable
# This is measured in units of pages (4096 bytes)
net.ipv4.tcp_mem = 65536 25165824 262144
net.ipv4.udp_mem = 65536 25165824 262144

# Minimum amount of memory allocated for the send and receive buffers for each socket.
net.ipv4.udp_wmem_min = 16384
net.ipv4.udp_rmem_min = 16384

# Enabling TCP window scaling by setting net.ipv4.tcp_window_scaling to 1 will allow
# clients to transfer data more efficiently, and allow that data to be buffered on the broker side.
net.ipv4.tcp_window_scaling = 1

# Increasing the value of net.ipv4.tcp_max_syn_backlog above the default of 1024 will allow
# a greater number of simultaneous connections to be accepted.
net.ipv4.tcp_max_syn_backlog = 10240

# Increasing the value of net.core.netdev_max_backlog to greater than the default of 1000
# can assist with bursts of network traffic, specifically when using multigigabit network connection speeds,
# by allowing more packets to be queued for the kernel to process them.
net.core.netdev_max_backlog = 65536

# Increase the maximum amount of option memory buffers
net.core.optmem_max = 25165824

# Number of times SYNACKs for passive TCP connection.
net.ipv4.tcp_synack_retries = 2

# Allowed local port range.
net.ipv4.ip_local_port_range = 2048 65535

# Protect Against TCP Time-Wait
# Default: net.ipv4.tcp_rfc1337 = 0
net.ipv4.tcp_rfc1337 = 1

# Decrease the time default value for tcp_fin_timeout connection
net.ipv4.tcp_fin_timeout = 15

# The maximum number of backlogged sockets.
# Default is 128.
net.core.somaxconn = 4096

# Turn on syncookies for SYN flood attack protection.
net.ipv4.tcp_syncookies = 1

# Avoid a smurf attack
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Turn on protection for bad icmp error messages
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Enable automatic window scaling.
# This will allow the TCP buffer to grow beyond its usual maximum of 64K if the latency justifies it.
net.ipv4.tcp_window_scaling = 1

# Turn on and log spoofed, source routed, and redirect packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Tells the kernel how many TCP sockets that are not attached to any
# user file handle to maintain. In case this number is exceeded,
# orphaned connections are immediately reset and a warning is printed.
# Default: net.ipv4.tcp_max_orphans = 65536
net.ipv4.tcp_max_orphans = 65536

# Do not cache metrics on closing connections
net.ipv4.tcp_no_metrics_save = 1

# Enable timestamps as defined in RFC1323:
# Default: net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_timestamps = 1

# Enable select acknowledgments.
# Default: net.ipv4.tcp_sack = 1
net.ipv4.tcp_sack = 1

# Increase the tcp-time-wait buckets pool size to prevent simple DOS attacks.
# net.ipv4.tcp_tw_recycle has been removed from Linux 4.12. Use net.ipv4.tcp_tw_reuse instead.
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_reuse = 1

# The accept_source_route option causes network interfaces to accept packets with the Strict Source Route (SSR) or Loose Source Routing (LSR) option set. 
# The following setting will drop packets with the SSR or LSR option set.
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Turn on reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable ICMP redirect acceptance
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# Disables sending of all IPv4 ICMP redirected packets.
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Disable IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

#############################################################################################
# Kubernetes related settings
#############################################################################################

# Enable IP forwarding.
# IP forwarding is the ability for an operating system to accept incoming network packets on one interface,
# recognize that it is not meant for the system itself, but that it should be passed on to another network, and then forwards it accordingly.
net.ipv4.ip_forward = 1

# These settings control whether packets traversing a network bridge are processed by iptables rules on the host system.
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# To prevent Linux conntrack table is out of space, increase the conntrack table size.
# This setting is for Calico networking.
net.netfilter.nf_conntrack_max = 1000000

#############################################################################################
# Tweak kernel parameters
#############################################################################################

# Address Space Layout Randomization (ASLR) is a memory-protection process for operating systems that guards against buffer-overflow attacks.
# It helps to ensure that the memory addresses associated with running processes on systems are not predictable,
# thus flaws or vulnerabilities associated with these processes will be more difficult to exploit.
# Accepted values: 0 = Disabled, 1 = Conservative Randomization, 2 = Full Randomization
kernel.randomize_va_space = 2

# Allow for more PIDs (to reduce rollover problems)
kernel.pid_max = 65536
EOF






sudo sysctl --system
swapoff -a

# REBOOT

########################### SPF913 DEPLOYING STACK CORE #################
cd /root
git clone https://github.com/spf913/spf913-depend.git
cd spf913-depend
curl -L https://www.dropbox.com/s/4tweyucolu83i2v/marathon.tar.gz -o marathon.tar.gz && \
curl -L https://www.dropbox.com/s/wk5jcmfs2zzkn31/mesos.tar.gz -o mesos.tar.gz && \
tar -zxvf marathon.tar.gz && \
tar -zxvf mesos.tar.gz


yum install -y --cacheonly --disablerepo=* /root/spf913-depend/se/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/yum/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/ip/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/cgroup/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/libtool/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/lvm2/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/unzip/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/wget/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/iproute/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/devicemapper/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/ebtables/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/vim/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/net-tools/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/chrony/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/ntpstat/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/keepalived/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/nginx/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/policycoreutils-python-utils/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/kernel-headers/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/kernel-devel/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/libssl/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/compat/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/mesos/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/marathon/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/zookeeper/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/unzip/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/java/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/subversion/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/cyrus/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/libevent/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/dnf/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/gcc/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/socat/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/conntrack/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/libcgroup/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/container-selinux/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/iptables/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/nftables/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/docker1806/*.rpm && \
yum install -y --cacheonly --disablerepo=* /root/spf913-depend/kubernetesv1.15.0/*.rpm


















##########################  SPF913 CORE DEPLOYED            ##################


# post deployment setup


systemctl enable irqbalance && \
systemctl start irqbalance

iptables -P FORWARD ACCEPT && \
sed -i '/ swap / s/^/#/' /etc/fstab && \
update-alternatives --set iptables /usr/sbin/iptables-legacy && \
sed -i '/^pool/c\pool time.google.com iburst' /etc/chrony.conf && \
timedatectl set-timezone Asia/Colombo && \
timedatectl set-ntp true && \
systemctl enable --now chronyd && systemctl status chronyd

ntpstat && \
chronyc sourcestats -v

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf > /dev/null
overlay
br_netfilter
EOF

modprobe overlay && \
modprobe br_netfilter && \
sysctl --system


# file limit optimization

vi /etc/systemd/user.conf
DefaultLimitNOFILE=100000

vi /etc/pam.d/login
session required pam_limits.so

vi /etc/systemd/system.conf
DefaultLimitNOFILE=100000

vi /etc/security/limits.conf
root soft nofile 100000
root hard nofile 100000

# sysctl --system



## Docker post Setup
mkdir -p /etc/docker && \
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

sudo systemctl enable docker && \
systemctl start docker && \
usermod -aG docker ${USER} && \
su ${USER} && \
id -nG

REBOOT


# Verify
docker info | grep -i cgroup
output:
Cgroup Driver: systemd


## check installed service
docker --version
kubeadm version
mesos-master --version
mesos-slave --version
systemctl start zookeeper
systemctl status zookeeper



```
