# OpenStack HA Manual Deploy

----------

## 整体架构

----------

![](architecture-1.jpg "High level architecture-1")

----------

![](architecture-2.jpg "High level architecture-2")

----------

## 控制节点

----------

### 环境说明
```
############################################################################################

# 节点名称：
#                controller-1      controller-2      controller-3

# 配置信息：
#                4核8G/3*300G      4核8G/3*300G      4核8G/3*300G

# 网络信息：
#         vip    192.168.0.130
#         eth0   192.168.0.131     192.168.0.132     192.168.0.133     API & Mgmt
#         eth1   access            access            access            L3
#         eth2   trunk             trunk             trunk             vlan data
#         eth3   192.168.103.131   192.168.103.132   192.168.103.133   vxlan data
#         eth4   192.168.104.131   192.168.104.132   192.168.104.133   ceph public
#         eth5   192.168.105.131   192.168.105.132   192.168.105.133   ceph cluster

# 操作系统：
#                CentOS 7

# 服务器角色：
#                Controller nodes
#                Load balancer nodes
#                Database nodes
#                Network nodes
#                Storage nodes
#                #Compute nodes

############################################################################################
```

----------

### 环境准备
```
# PXE 安装操作系统（CentOS 7 最小化安装）
## 1. swap : 120G | / : 150G
## 2. 关闭 SELinux、NetworkManager、firewalld、tuned
## 3. root/root1234
## 4. 安装 bash-completion、chrony、tcpdump、telnet、vim、wget
## 5. 配置本地源
## 6. yum clean all && yum update -y
## 7. 安装 nmon
## 8. 重启服务器

echo -e "\n192.168.0.130\tcontroller-vip" >> /etc/hosts
echo -e "192.168.0.131\tcontroller-1" >> /etc/hosts
echo -e "192.168.0.132\tcontroller-2" >> /etc/hosts
echo -e "192.168.0.133\tcontroller-3" >> /etc/hosts

echo -e "\tStrictHostKeyChecking=no" >> /etc/ssh/ssh_config

ssh-keygen


ssh-copy-id controller-1


ssh-copy-id controller-2

ssh-copy-id controller-3

#controller-1
#---------------------------------------------------------------------

sed -i -e "s/^server .*/#&/" /etc/chrony.conf
sed -i -e "s/^allow .*/#&/" /etc/chrony.conf

cat >> /etc/chrony.conf << EOF

server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst
server controller-1 iburst

allow 192.168/16
EOF

#---------------------------------------------------------------------

#controller-2
#---------------------------------------------------------------------

sed -i -e "s/^server .*/#&/" /etc/chrony.conf
sed -i -e "s/^allow .*/#&/" /etc/chrony.conf

cat >> /etc/chrony.conf << EOF

server controller-1 iburst
server controller-2 iburst

allow 192.168/16
EOF

#---------------------------------------------------------------------

#controller-3
#---------------------------------------------------------------------

sed -i -e "s/^server .*/#&/" /etc/chrony.conf
sed -i -e "s/^allow .*/#&/" /etc/chrony.conf

cat >> /etc/chrony.conf << EOF

server controller-1 iburst
server controller-2 iburst
server controller-3 iburst

allow 192.168/16
EOF

#---------------------------------------------------------------------

systemctl restart chronyd
sleep 10
chronyc sources -v


```

----------

### 安装配置 Ceph
```
cat > /etc/yum.repos.d/ceph-infernalis.repo << EOF
[ceph-infernalis-x86_64]
name=Ceph Infernalis X86_64
baseurl=http://192.168.0.100/centos/7/ceph/x86_64/rpm-infernalis/
gpgcheck=0

[ceph-infernalis-noarch]
name=Ceph Infernalis Noarch
baseurl=http://192.168.0.100/centos/7/ceph/noarch/rpm-infernalis/
gpgcheck=0
EOF

cat > /etc/yum.repos.d/epel.repo << EOF
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
baseurl=http://mirrors.ustc.edu.cn/epel/7/x86_64/
failovermethod=priority
enabled=1
gpgcheck=1
gpgkey=http://mirrors.ustc.edu.cn/epel/RPM-GPG-KEY-EPEL-7
EOF

useradd -d /home/ceph -m ceph

echo ceph1234 \
| passwd --stdin ceph

cat << EOF > /etc/sudoers.d/ceph
Defaults:ceph !requiretty

ceph ALL = (root) NOPASSWD: ALL
EOF

chmod 0440 /etc/sudoers.d/ceph

#controller-1
#---------------------------------------------------------------------

yum install -y ceph-deploy
ceph-deploy --version

su - ceph
pwd
ssh-keygen


ssh-copy-id controller-2


ssh-copy-id controller-3


mkdir my-cluster
cd my-cluster/
ceph-deploy install --no-adjust-repos controller-1 controller-2 controller-3


cat << EOF > ceph.conf
[global]
fsid = $(uuidgen)
mon_initial_members = controller-1,controller-2,controller-3
mon_host = 192.168.104.131,192.168.104.132,192.168.104.133
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
filestore_xattr_use_omap = true

public_network = 192.168.104.0/24
cluster_network = 192.168.105.0/24

mon osd full ratio = .80
mon osd nearfull ratio = .70
filestore max sync interval = 10

osd_mkfs_type = xfs
osd_mkfs_options_xfs = -f -i size=2048
osd_mount_options_xfs = rw,noatime,inode64,logbsize=256k,delaylog

#osd_pool_default_size = 1
#osd_crush_chooseleaf_type = 0

[osd]
osd_journal_size = 20480

[osd.0]
host = controller-1
public_addr = 192.168.104.131
cluster_addr = 192.168.105.131

[osd.1]
host = controller-2
public_addr = 192.168.104.132
cluster_addr = 192.168.105.132

[osd.2]
host = controller-3
public_addr = 192.168.104.133
cluster_addr = 192.168.105.133
EOF

ceph-deploy mon create-initial


ceph-deploy osd prepare controller-1:sdb
ceph-deploy osd activate controller-1:sdb1:sdb2
ceph-deploy osd prepare controller-2:sdb
ceph-deploy osd activate controller-2:sdb1:sdb2
ceph-deploy osd prepare controller-3:sdb
ceph-deploy osd activate controller-3:sdb1:sdb2


sudo ceph osd pool delete rbd rbd --yes-i-really-really-mean-it
sudo ceph osd pool create volumes 32
sudo ceph osd pool create images 32
sudo ceph osd pool create backups 32
sudo ceph osd pool create vms 32
sudo ceph osd lspools

exit
sleep 10
ceph -s

#---------------------------------------------------------------------

mount | grep 'sdb1' | awk '{print $1" "$3" "$5" "$6}' | awk -F '(' '{print $1$2}' | awk -F ')' '{print $1}' >> /etc/fstab
sed -i -e "s/\/dev\/sdb1/$(blkid | grep sdb1 | awk '{print $2}' | awk -F '"' '{print $1$2}')/" /etc/fstab

yum clean all
mv /etc/yum.repos.d/ceph-infernalis.repo /etc/yum.repos.d/epel.repo /etc/yum.repos.d/bak/


```

----------

### 安装配置 Pacemaker
```
cat > /etc/yum.repos.d/openstack-liberty.repo << EOF
[openstack-liberty]
name=OpenStack Liberty
baseurl=http://192.168.0.100/centos/7/cloud/x86_64/openstack-liberty/
gpgcheck=0
EOF

yum install -y pcs fence-agents-all fence-agents-virsh

systemctl start pcsd
systemctl enable pcsd

echo hacluster1234 \
| passwd --stdin hacluster

#controller-1
#---------------------------------------------------------------------

pcs cluster auth controller-1 controller-2 controller-3 \
-u hacluster -p hacluster1234 --force

pcs cluster setup --force --name openstack-cluster \
controller-1 controller-2 controller-3

pcs cluster start --all
pcs cluster enable --all

#pcs property set stonith-enabled=false
#pcs property set no-quorum-policy=ignore

pcs property set pe-warn-series-max=1000 \
pe-input-series-max=1000 \
pe-error-series-max=1000 \
cluster-recheck-interval=1min

pcs stonith create fence-server fence_virsh ipaddr=192.168.0.53 login=root passwd=root1234 pcmk_host_map="controller-1:test1;controller-2:test2;controller-3:test3" action=off --force

pcs stonith level add 1 controller-1 fence-server
pcs stonith level add 1 controller-2 fence-server
pcs stonith level add 1 controller-3 fence-server

mkdir -p /etc/pacemaker/
dd if=/dev/urandom of=/etc/pacemaker/authkey bs=4096 count=1
for i in 2 3; do ssh controller-$i "mkdir -p /etc/pacemaker/"; scp /etc/pacemaker/authkey controller-$i:/etc/pacemaker/; done

#---------------------------------------------------------------------
```

----------

### 安装配置 HAProxy
```
yum install -y haproxy

echo 'net.ipv4.ip_nonlocal_bind=1' >> /etc/sysctl.d/haproxy.conf
sysctl -p /etc/sysctl.d/haproxy.conf

cat > /etc/sysctl.d/tcp_keepalive.conf << EOF
net.ipv4.tcp_keepalive_intvl = 1
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_time = 5
EOF
sysctl -p /etc/sysctl.d/tcp_keepalive.conf


cat > /etc/haproxy/haproxy.cfg << EOF
global
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode tcp
    maxconn 10000
    timeout connect 5s
    timeout client 30s
    timeout server 30s

listen monitor
    bind 192.168.0.130:9300
    mode http
    monitor-uri /status
    stats enable
    stats uri /admin
    stats realm Haproxy\ Statistics
    stats auth root:root1234
    stats refresh 5s

frontend vip-db
    bind 192.168.0.130:3306
    timeout client 90m
    default_backend db-vms-galera
backend db-vms-galera
    option httpchk
    stick-table type ip size 1000
    stick on dst
    timeout server 90m
    server controller-1 192.168.0.131:3306 check inter 1s port 9200 backup on-marked-down shutdown-sessions
    server controller-2 192.168.0.132:3306 check inter 1s port 9200 backup on-marked-down shutdown-sessions
    server controller-3 192.168.0.133:3306 check inter 1s port 9200 backup on-marked-down shutdown-sessions

frontend vip-keystone-admin
    bind 192.168.0.130:35357
    default_backend keystone-admin-vms
    timeout client 600s
backend keystone-admin-vms
    balance roundrobin
    timeout server 600s
    server controller-1 192.168.0.131:35357 check inter 1s on-marked-down shutdown-sessions
    server controller-2 192.168.0.132:35357 check inter 1s on-marked-down shutdown-sessions
    server controller-3 192.168.0.133:35357 check inter 1s on-marked-down shutdown-sessions

frontend vip-keystone-public
    bind 192.168.0.130:5000
    default_backend keystone-public-vms
    timeout client 600s
backend keystone-public-vms
    balance roundrobin
    timeout server 600s
    server controller-1 192.168.0.131:5000 check inter 1s on-marked-down shutdown-sessions
    server controller-2 192.168.0.132:5000 check inter 1s on-marked-down shutdown-sessions
    server controller-3 192.168.0.133:5000 check inter 1s on-marked-down shutdown-sessions

frontend vip-glance-api
    bind 192.168.0.130:9191
    default_backend glance-api-vms
backend glance-api-vms
    balance roundrobin
    server controller-1 192.168.0.131:9191 check inter 1s
    server controller-2 192.168.0.132:9191 check inter 1s
    server controller-3 192.168.0.133:9191 check inter 1s

frontend vip-glance-registry
    bind 192.168.0.130:9292
    default_backend glance-registry-vms
backend glance-registry-vms
    balance roundrobin
    server controller-1 192.168.0.131:9292 check inter 1s
    server controller-2 192.168.0.132:9292 check inter 1s
    server controller-3 192.168.0.133:9292 check inter 1s

frontend vip-cinder
    bind 192.168.0.130:8776
    default_backend cinder-vms
backend cinder-vms
    balance roundrobin
    server controller-1 192.168.0.131:8776 check inter 1s
    server controller-2 192.168.0.132:8776 check inter 1s
    server controller-3 192.168.0.133:8776 check inter 1s

frontend vip-neutron
    bind 192.168.0.130:9696
    default_backend neutron-vms
backend neutron-vms
    balance roundrobin
    server controller-1 192.168.0.131:9696 check inter 1s
    server controller-2 192.168.0.132:9696 check inter 1s
    server controller-3 192.168.0.133:9696 check inter 1s

frontend vip-nova-vnc-novncproxy
    bind 192.168.0.130:6080
    default_backend nova-vnc-novncproxy-vms
backend nova-vnc-novncproxy-vms
    balance roundrobin
    timeout tunnel 1h
    server controller-1 192.168.0.131:6080 check inter 1s
    server controller-2 192.168.0.132:6080 check inter 1s
    server controller-3 192.168.0.133:6080 check inter 1s

frontend nova-metadata-vms
    bind 192.168.0.130:8775
    default_backend nova-metadata-vms
backend nova-metadata-vms
    balance roundrobin
    server controller-1 192.168.0.131:8775 check inter 1s
    server controller-2 192.168.0.132:8775 check inter 1s
    server controller-3 192.168.0.133:8775 check inter 1s

frontend vip-nova-api
    bind 192.168.0.130:8774
    default_backend nova-api-vms
backend nova-api-vms
    balance roundrobin
    server controller-1 192.168.0.131:8774 check inter 1s
    server controller-2 192.168.0.132:8774 check inter 1s
    server controller-3 192.168.0.133:8774 check inter 1s

frontend vip-horizon
    bind 192.168.0.130:80
    timeout client 180s
    default_backend horizon-vms
backend horizon-vms
    balance roundrobin
    timeout server 180s
    mode http
    cookie SERVERID insert indirect nocache
    server controller-1 192.168.0.131:80 check inter 1s cookie controller-1 on-marked-down shutdown-sessions
    server controller-2 192.168.0.132:80 check inter 1s cookie controller-2 on-marked-down shutdown-sessions
    server controller-3 192.168.0.133:80 check inter 1s cookie controller-3 on-marked-down shutdown-sessions

frontend vip-ceilometer
    bind 192.168.0.130:8777
    timeout client 90s
    default_backend ceilometer-vms
backend ceilometer-vms
    balance roundrobin
    timeout server 90s
    server controller-1 192.168.0.131:8777 check inter 1s
    server controller-2 192.168.0.132:8777 check inter 1s
    server controller-3 192.168.0.133:8777 check inter 1s
EOF

#controller-1
#---------------------------------------------------------------------

pcs resource create controller-vip IPaddr2 \
ip=192.168.0.130 cidr_netmask=23 nic=eth0 \
op monitor interval=30s

pcs resource create haproxy systemd:haproxy --clone
pcs constraint order start controller-vip then haproxy-clone kind=Optional
pcs constraint colocation add haproxy-clone with controller-vip

#---------------------------------------------------------------------
```

----------

### 安装配置 Galera
```
yum install -y mariadb-galera-server xinetd rsync

#controller-1
#---------------------------------------------------------------------

pcs resource disable haproxy

#---------------------------------------------------------------------

cat > /etc/sysconfig/clustercheck << EOF
MYSQL_USERNAME="clustercheck"
MYSQL_PASSWORD="clustercheck1234"
MYSQL_HOST="localhost"
MYSQL_PORT="3306"
EOF

systemctl start mariadb
mysql -e "CREATE USER 'clustercheck'@'localhost' IDENTIFIED BY 'clustercheck1234';"
systemctl stop mariadb


cat > /etc/my.cnf.d/galera.cnf << EOF
[mysqld]
skip-name-resolve=1
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
innodb_locks_unsafe_for_binlog=1
max_connections=8192
query_cache_size=0
query_cache_type=0
bind_address=$(hostname -s)
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_name="galera_cluster"
wsrep_slave_threads=1
wsrep_certify_nonPK=1
wsrep_max_ws_rows=131072
wsrep_max_ws_size=1073741824
wsrep_debug=0
wsrep_convert_LOCK_to_trx=0
wsrep_retry_autocommit=1
wsrep_auto_increment_control=1
wsrep_drupal_282555_workaround=0
wsrep_causal_reads=0
wsrep_notify_cmd=
wsrep_sst_method=rsync
wsrep_cluster_address="gcomm://controller-1,controller-2,controller-3"
wsrep_on=ON
EOF

cat > /etc/xinetd.d/galera-monitor << EOF
service galera-monitor
{
    port = 9200
    disable = no
    socket_type = stream
    protocol = tcp
    wait = no
    user = root
    group = root
    groups = yes
    server = /usr/bin/clustercheck
    type = UNLISTED
    per_source = UNLIMITED
    log_on_success = 
    log_on_failure = HOST
    flags = REUSE
}
EOF

systemctl start xinetd
systemctl enable xinetd

#controller-1
#---------------------------------------------------------------------

pcs resource create galera galera enable_creation=true wsrep_cluster_address="gcomm://controller-1,controller-2,controller-3" additional_parameters='--open-files-limit=16384' meta master-max=3 ordered=true op promote timeout=300s on-fail=block --master

pcs resource enable haproxy

sleep 45
mysql -e "show status like 'wsrep%';"


mysql -e "select user,host,password from mysql.user;"

mysql -e "drop user ''@'controller-1';"
mysql -e "drop user 'root'@'controller-1';"

cat > galera.setup << EOF
#GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED by 'root1234' WITH GRANT OPTION;
CREATE DATABASE keystone;
GRANT ALL ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone1234';
CREATE DATABASE glance;
GRANT ALL ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance1234';
CREATE DATABASE cinder;
GRANT ALL ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'cinder1234';
CREATE DATABASE neutron;
GRANT ALL ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron1234';
CREATE DATABASE nova;
GRANT ALL ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova1234';
FLUSH PRIVILEGES;
quit
EOF

mysql mysql < galera.setup
mysql -e "select user,host,password from mysql.user;"

mysqladmin flush-hosts
rm -f galera.setup

#---------------------------------------------------------------------
```

----------

### 安装配置 RabbitMQ
```
yum install -y rabbitmq-server

#controller-1
#---------------------------------------------------------------------

cat > /etc/rabbitmq/rabbitmq-env.conf << EOF
NODE_IP_ADDRESS=$(ip addr | grep 'eth0:' -A2 | tail -n1 | awk '{print $2}' | awk -F '/' '{print $1}')
EOF

systemctl start rabbitmq-server
systemctl stop rabbitmq-server
scp -p /var/lib/rabbitmq/.erlang.cookie controller-2:/var/lib/rabbitmq
scp -p /var/lib/rabbitmq/.erlang.cookie controller-3:/var/lib/rabbitmq

#---------------------------------------------------------------------

#controller-2 & controller-3
#---------------------------------------------------------------------

chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie

cat > /etc/rabbitmq/rabbitmq-env.conf << EOF
NODE_IP_ADDRESS=$(ip addr | grep 'eth0:' -A2 | tail -n1 | awk '{print $2}' | awk -F '/' '{print $1}')
EOF

#---------------------------------------------------------------------

#controller-1
#---------------------------------------------------------------------

pcs resource create rabbitmq rabbitmq-cluster set_policy='ha-all ^(?!amq\.).* {"ha-mode":"all"}' meta notify=true --clone ordered=true interleave=true

#---------------------------------------------------------------------
```

----------

### 安装配置 Memcached
```
yum install -y memcached

#controller-1
#---------------------------------------------------------------------

pcs resource create memcached systemd:memcached --clone interleave=true

#---------------------------------------------------------------------
```

----------

### 安装配置 MongoDB
```
yum install -y mongodb mongodb-server

sed -i \
	-e "s/.*bind_ip.*/bind_ip = 127.0.0.1,$(hostname -s)/g" \
	-e 's/.*replSet.*/replSet = ceilometer/g' \
	-e 's/.*smallfiles.*/smallfiles = true/g' \
	/etc/mongod.conf

#controller-1
#---------------------------------------------------------------------

pcs resource create mongod systemd:mongod op start timeout=300s --clone

sleep 10
cat > ~/mongo_replica_setup.js << EOF
rs.initiate()
sleep(10000)
rs.add("controller-2");
rs.add("controller-3");
EOF

mongo ~/mongo_replica_setup.js
rm -f ~/mongo_replica_setup.js

#---------------------------------------------------------------------
```

----------

### 安装配置 Redis
```
yum install -y redis

sed -i "s/bind 127.0.0.1/bind 127.0.0.1 $(hostname -s)/" /etc/redis.conf

#controller-2 & controller-3
#---------------------------------------------------------------------

echo slaveof ''controller-1'' 6379 >> /etc/redis.conf

#---------------------------------------------------------------------

cat > /etc/redis-sentinel.conf << EOF
sentinel monitor mymaster controller-1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1
min-slaves-to-write 1
min-slaves-max-lag 10
logfile /var/log/redis/sentinel.log
EOF

#controller-1
#---------------------------------------------------------------------

pcs resource create redis systemd:redis --clone interleave=true
pcs resource create redis-sentinel systemd:redis-sentinel --clone interleave=true

pcs constraint order start redis-clone then redis-sentinel-clone
pcs constraint colocation add redis-sentinel-clone with redis-clone

#---------------------------------------------------------------------
```

----------

### 安装配置 Keystone
```
yum install -y openstack-keystone openstack-utils python-openstackclient

#controller-1
#---------------------------------------------------------------------

openssl rand -hex 10 > ~/keystone_service_token
scp ~/keystone_service_token controller-2:~/
scp ~/keystone_service_token controller-3:~/

#---------------------------------------------------------------------

openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token $(cat ~/keystone_service_token)
openstack-config --set /etc/keystone/keystone.conf DEFAULT rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/keystone/keystone.conf DEFAULT rabbit_ha_queues true
openstack-config --set /etc/keystone/keystone.conf eventlet_server admin_endpoint 'http://controller-vip:%(admin_port)s/'
openstack-config --set /etc/keystone/keystone.conf eventlet_server public_endpoint 'http://controller-vip:%(public_port)s/'
openstack-config --set /etc/keystone/keystone.conf database connection mysql://keystone:keystone1234@controller-vip/keystone
openstack-config --set /etc/keystone/keystone.conf database max_retries -1
openstack-config --set /etc/keystone/keystone.conf DEFAULT public_bind_host $(hostname -s)
openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_bind_host $(hostname -s)
openstack-config --set /etc/keystone/keystone.conf token driver keystone.token.persistence.backends.sql.Token

#controller-1
#---------------------------------------------------------------------

keystone-manage pki_setup --keystone-user keystone --keystone-group keystone
chown -R keystone:keystone /var/log/keystone /etc/keystone/ssl/
su keystone -s /bin/sh -c "keystone-manage db_sync"
cd /etc/keystone/ssl
tar cvfz /tmp/keystone_ssl.tgz *
scp /tmp/keystone_ssl.tgz controller-2:/tmp
scp /tmp/keystone_ssl.tgz controller-3:/tmp
cd

#---------------------------------------------------------------------

#controller-2 & controller-3
#---------------------------------------------------------------------

mkdir -p /etc/keystone/ssl
tar xvfz /tmp/keystone_ssl.tgz -C /etc/keystone/ssl
chown -R keystone:keystone /var/log/keystone /etc/keystone/ssl/
restorecon -Rv /etc/keystone/ssl
touch /var/log/keystone/keystone.log
chown keystone:keystone /var/log/keystone/keystone.log

#---------------------------------------------------------------------

#controller-1
#---------------------------------------------------------------------

pcs resource create openstack-keystone systemd:openstack-keystone --clone interleave=true
#pcs constraint order start haproxy-clone then openstack-keystone-clone
pcs constraint order promote galera-master then openstack-keystone-clone
pcs constraint order start rabbitmq-clone then openstack-keystone-clone
pcs constraint order start memcached-clone then openstack-keystone-clone

sleep 10
export OS_TOKEN=$(cat ~/keystone_service_token)
export OS_URL=http://controller-vip:35357/v2.0
export OS_REGION_NAME=regionOne
# keystone
openstack service create --name=keystone --description="OpenStack Identity Service" identity
openstack endpoint create --publicurl 'http://controller-vip:5000/v2.0' --adminurl 'http://controller-vip:35357/v2.0' --internalurl 'http://controller-vip:5000/v2.0' --region regionOne keystone
openstack user create --password admin1234 admin
openstack role create admin
openstack project create admin
openstack role add --project admin --user admin admin
openstack user create --password demo1234 demo
openstack role create _member_
openstack project create demo
openstack role add --project demo --user demo _member_
openstack project create --description "Services Tenant" services
# glance
openstack user create --password glance1234 glance
openstack role add --project services --user glance admin
openstack service create --name=glance --description="OpenStack Image Service" image
openstack endpoint create --publicurl 'http://controller-vip:9292' --adminurl 'http://controller-vip:9292' --internalurl 'http://controller-vip:9292' --region regionOne glance
# cinder
openstack user create --password cinder1234 cinder
openstack role add --project services --user cinder admin
openstack service create --name=cinder --description="OpenStack Volume Service" volume
openstack endpoint create --publicurl "http://controller-vip:8776/v1/\$(tenant_id)s" --adminurl "http://controller-vip:8776/v1/\$(tenant_id)s" --internalurl "http://controller-vip:8776/v1/\$(tenant_id)s" --region regionOne cinder
openstack service create --name=cinderv2 --description="OpenStack Volumev2 Service" volumev2
openstack endpoint create --publicurl "http://controller-vip:8776/v2/\$(tenant_id)s" --adminurl "http://controller-vip:8776/v2/\$(tenant_id)s" --internalurl "http://controller-vip:8776/v2/\$(tenant_id)s" --region regionOne cinderv2


# neutron
openstack user create --password neutron1234 neutron
openstack role add --project services --user neutron admin
openstack service create --name=neutron --description="OpenStack Network Service" network
openstack endpoint create --publicurl "http://controller-vip:9696" --adminurl "http://controller-vip:9696" --internalurl "http://controller-vip:9696" --region regionOne neutron
# nova
openstack user create --password nova1234 nova
openstack role add --project services --user nova admin
openstack service create --name=nova --description="OpenStack Compute Service" compute
openstack endpoint create --publicurl "http://controller-vip:8774/v2/\$(tenant_id)s" --adminurl "http://controller-vip:8774/v2/\$(tenant_id)s" --internalurl "http://controller-vip:8774/v2/\$(tenant_id)s" --region regionOne nova
# ceilometer
openstack user create --password ceilometer1234 ceilometer
openstack role add --project services --user ceilometer admin
openstack role create ResellerAdmin
openstack role add --project services --user ceilometer ResellerAdmin
openstack service create --name=ceilometer --description="OpenStack Metering Service" metering
openstack endpoint create --publicurl "http://controller-vip:8777" --adminurl "http://controller-vip:8777" --internalurl "http://controller-vip:8777" --region regionOne ceilometer

#---------------------------------------------------------------------

cat > ~/keystonerc_admin << EOF
export OS_USERNAME=admin 
export OS_TENANT_NAME=admin
export OS_PROJECT_NAME=admin
export OS_REGION_NAME=regionOne
export OS_PASSWORD=admin1234
export OS_AUTH_URL=http://controller-vip:35357/v2.0/
export PS1='[\u@\h \W(keystone_admin)]\$ '
EOF

cat > ~/keystonerc_demo << EOF
export OS_USERNAME=demo
export OS_TENANT_NAME=demo
export OS_PROJECT_NAME=demo
export OS_REGION_NAME=regionOne
export OS_PASSWORD=demo1234
export OS_AUTH_URL=http://controller-vip:5000/v2.0/
export PS1='[\u@\h \W(keystone_user)]\$ '
EOF

rm -f ~/keystone_service_token


```

----------

### 安装配置 Glance
```
yum install -y openstack-glance openstack-utils python-openstackclient

openstack-config --set /etc/glance/glance-api.conf database connection mysql://glance:glance1234@controller-vip/glance
openstack-config --set /etc/glance/glance-api.conf database max_retries -1
openstack-config --set /etc/glance/glance-api.conf paste_deploy flavor keystone
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_uri http://controller-vip:5000/
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken username glance
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken password glance1234
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken project_name services
openstack-config --set /etc/glance/glance-api.conf DEFAULT notification_driver messaging
openstack-config --set /etc/glance/glance-api.conf DEFAULT bind_host $(hostname -s)
openstack-config --set /etc/glance/glance-api.conf DEFAULT registry_host controller-vip
openstack-config --set /etc/glance/glance-api.conf oslo_messaging_rabbit rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/glance/glance-api.conf oslo_messaging_rabbit rabbit_ha_queues true
openstack-config --set /etc/glance/glance-registry.conf database connection mysql://glance:glance1234@controller-vip/glance
openstack-config --set /etc/glance/glance-registry.conf database max_retries -1
openstack-config --set /etc/glance/glance-registry.conf paste_deploy flavor keystone
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_uri http://controller-vip:5000/
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken username glance
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken password glance1234
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken project_name services
openstack-config --set /etc/glance/glance-registry.conf DEFAULT bind_host $(hostname -s)

openstack-config --set /etc/glance/glance-api.conf DEFAULT show_image_direct_url true
openstack-config --set /etc/glance/glance-api.conf glance_store stores rbd,http
openstack-config --set /etc/glance/glance-api.conf glance_store default_store rbd
openstack-config --set /etc/glance/glance-api.conf glance_store rbd_store_chunk_size 8
openstack-config --set /etc/glance/glance-api.conf glance_store rbd_store_pool images
openstack-config --set /etc/glance/glance-api.conf glance_store rbd_store_user glance
openstack-config --set /etc/glance/glance-api.conf glance_store rbd_store_ceph_conf /etc/ceph/ceph.conf
openstack-config --set /etc/glance/glance-api.conf paste_deploy flavor keystone

ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images' | tee /etc/ceph/ceph.client.glance.keyring
chown glance:glance /etc/ceph/ceph.client.glance.keyring

#controller-1
#---------------------------------------------------------------------

su glance -s /bin/sh -c "glance-manage db_sync"

pcs resource create openstack-glance-registry systemd:openstack-glance-registry --clone interleave=true
pcs resource create openstack-glance-api systemd:openstack-glance-api --clone interleave=true

pcs constraint order start openstack-glance-registry-clone then openstack-glance-api-clone
pcs constraint colocation add openstack-glance-api-clone with openstack-glance-registry-clone

pcs constraint order start openstack-keystone-clone then openstack-glance-registry-clone

sleep 10
source ~/keystonerc_admin
curl http://192.168.0.100/images/cirros-0.3.4-x86_64-disk.raw | glance image-create --name='cirros image' --visibility=public --container-format=bare --disk-format=raw --protected=true --progress
glance image-list

#---------------------------------------------------------------------
```

----------

### 安装配置 Cinder
```
yum install -y openstack-cinder openstack-utils python-memcached python-keystonemiddleware python-openstackclient

openstack-config --set /etc/cinder/cinder.conf database connection mysql://cinder:cinder1234@controller-vip/cinder
openstack-config --set /etc/cinder/cinder.conf database max_retries -1
openstack-config --set /etc/cinder/cinder.conf DEFAULT auth_strategy keystone
openstack-config --set /etc/cinder/cinder.conf keystone_authtoken auth_uri http://controller-vip:5000/
openstack-config --set /etc/cinder/cinder.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/cinder/cinder.conf keystone_authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/cinder/cinder.conf keystone_authtoken username cinder
openstack-config --set /etc/cinder/cinder.conf keystone_authtoken password cinder1234
openstack-config --set /etc/cinder/cinder.conf keystone_authtoken project_name services
openstack-config --set /etc/cinder/cinder.conf DEFAULT notification_driver messaging
openstack-config --set /etc/cinder/cinder.conf DEFAULT control_exchange cinder
openstack-config --set /etc/cinder/cinder.conf DEFAULT glance_host controller-vip
openstack-config --set /etc/cinder/cinder.conf DEFAULT memcache_servers controller-1:11211,controller-2:11211,controller-3:11211
openstack-config --set /etc/cinder/cinder.conf DEFAULT host openstack-cinder
openstack-config --set /etc/cinder/cinder.conf DEFAULT osapi_volume_listen $(hostname -s)
openstack-config --set /etc/cinder/cinder.conf oslo_messaging_rabbit rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/cinder/cinder.conf oslo_messaging_rabbit rabbit_ha_queues true
openstack-config --set /etc/cinder/cinder.conf oslo_messaging_rabbit heartbeat_timeout_threshold 60
openstack-config --set /etc/cinder/cinder.conf keymgr encryption_auth_url http://controller-vip:5000/v3

#controller-1
#---------------------------------------------------------------------

uuidgen > ~/rbd_secret_uuid
scp ~/rbd_secret_uuid controller-2:~/
scp ~/rbd_secret_uuid controller-3:~/

#---------------------------------------------------------------------

openstack-config --set /etc/cinder/cinder.conf DEFAULT backup_ceph_conf /etc/ceph/ceph.conf
openstack-config --set /etc/cinder/cinder.conf DEFAULT backup_ceph_user cinder-backup
openstack-config --set /etc/cinder/cinder.conf DEFAULT backup_ceph_chunk_size 34217728
openstack-config --set /etc/cinder/cinder.conf DEFAULT backup_ceph_pool backups
openstack-config --set /etc/cinder/cinder.conf DEFAULT backup_ceph_stripe_unit 0
openstack-config --set /etc/cinder/cinder.conf DEFAULT backup_ceph_stripe_count 0
openstack-config --set /etc/cinder/cinder.conf DEFAULT restore_discard_excess_bytes true
openstack-config --set /etc/cinder/cinder.conf DEFAULT backup_driver cinder.backup.drivers.ceph
openstack-config --set /etc/cinder/cinder.conf DEFAULT glance_api_version 2
openstack-config --set /etc/cinder/cinder.conf DEFAULT default_volume_type rbd
openstack-config --set /etc/cinder/cinder.conf DEFAULT enabled_backends rbd
openstack-config --set /etc/cinder/cinder.conf rbd rbd_pool volumes
openstack-config --set /etc/cinder/cinder.conf rbd rbd_user cinder
openstack-config --set /etc/cinder/cinder.conf rbd rbd_ceph_conf /etc/ceph/ceph.conf
openstack-config --set /etc/cinder/cinder.conf rbd rbd_flatten_volume_from_snapshot false
openstack-config --set /etc/cinder/cinder.conf rbd rbd_secret_uuid $(cat ~/rbd_secret_uuid)
openstack-config --set /etc/cinder/cinder.conf rbd rbd_max_clone_depth 5
openstack-config --set /etc/cinder/cinder.conf rbd rbd_store_chunk_size 4
openstack-config --set /etc/cinder/cinder.conf rbd rados_connect_timeout -1
openstack-config --set /etc/cinder/cinder.conf rbd volume_group rbd
openstack-config --set /etc/cinder/cinder.conf rbd volume_driver cinder.volume.drivers.rbd.RBDDriver
openstack-config --set /etc/cinder/cinder.conf rbd volume_backend_name rbd

ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images' | tee /etc/ceph/ceph.client.cinder.keyring
chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups' | tee /etc/ceph/ceph.client.cinder-backup.keyring
chown cinder:cinder /etc/ceph/ceph.client.cinder-backup.keyring

#controller-1
#---------------------------------------------------------------------

su cinder -s /bin/sh -c "cinder-manage db sync"

pcs resource create openstack-cinder-api systemd:openstack-cinder-api --clone interleave=true
pcs resource create openstack-cinder-scheduler systemd:openstack-cinder-scheduler --clone interleave=true

pcs resource create openstack-cinder-volume systemd:openstack-cinder-volume

pcs resource create openstack-cinder-backup systemd:openstack-cinder-backup --clone interleave=true

pcs constraint order start openstack-cinder-api-clone then openstack-cinder-scheduler-clone
pcs constraint colocation add openstack-cinder-scheduler-clone with openstack-cinder-api-clone
pcs constraint order start openstack-cinder-scheduler-clone then openstack-cinder-volume
pcs constraint colocation add openstack-cinder-volume with openstack-cinder-scheduler-clone
pcs constraint order start openstack-cinder-scheduler-clone then openstack-cinder-backup-clone
pcs constraint colocation add openstack-cinder-backup-clone with openstack-cinder-scheduler-clone

pcs constraint order start openstack-keystone-clone then openstack-cinder-api-clone

sleep 10
source ~/keystonerc_admin
cinder type-create rbd
cinder type-key rbd set volume_backend_name=rbd
cinder extra-specs-list

#---------------------------------------------------------------------
```

----------

### 安装配置 Neutron
```
yum install -y openstack-neutron openstack-neutron-openvswitch openstack-neutron-ml2

#openstack-config --set /etc/neutron/neutron.conf DEFAULT host $(hostname -s)
openstack-config --set /etc/neutron/neutron.conf DEFAULT bind_host $(hostname -s)
openstack-config --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller-vip:5000/
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken username neutron
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken password neutron1234
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken project_name services
openstack-config --set /etc/neutron/neutron.conf database connection mysql://neutron:neutron1234@controller-vip:3306/neutron
openstack-config --set /etc/neutron/neutron.conf database max_retries -1
openstack-config --set /etc/neutron/neutron.conf DEFAULT notification_driver neutron.openstack.common.notifier.rpc_notifier
openstack-config --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_ha_queues true
openstack-config --set /etc/neutron/neutron.conf nova nova_region_name regionOne
openstack-config --set /etc/neutron/neutron.conf nova project_domain_id default
openstack-config --set /etc/neutron/neutron.conf nova project_name services
openstack-config --set /etc/neutron/neutron.conf nova user_domain_id default
openstack-config --set /etc/neutron/neutron.conf nova username nova
openstack-config --set /etc/neutron/neutron.conf nova password nova1234
openstack-config --set /etc/neutron/neutron.conf nova auth_url http://controller-vip:35357/
openstack-config --set /etc/neutron/neutron.conf nova auth_plugin password
openstack-config --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes True
openstack-config --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes True
openstack-config --set /etc/neutron/neutron.conf DEFAULT core_plugin neutron.plugins.ml2.plugin.Ml2Plugin
openstack-config --set /etc/neutron/neutron.conf DEFAULT service_plugins router
openstack-config --set /etc/neutron/neutron.conf DEFAULT router_scheduler_driver neutron.scheduler.l3_agent_scheduler.ChanceScheduler
openstack-config --set /etc/neutron/neutron.conf DEFAULT dhcp_agents_per_network 2
openstack-config --set /etc/neutron/neutron.conf DEFAULT api_workers 2
openstack-config --set /etc/neutron/neutron.conf DEFAULT rpc_workers 2
openstack-config --set /etc/neutron/neutron.conf DEFAULT l3_ha True
openstack-config --set /etc/neutron/neutron.conf DEFAULT min_l3_agents_per_router 2
openstack-config --set /etc/neutron/neutron.conf DEFAULT max_l3_agents_per_router 2


ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers local,gre,flat,vxlan,vlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vlan,vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers openvswitch
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks \*
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vlan network_vlan_ranges physnet1:100:1000
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_gre tunnel_id_ranges 100:1000
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 100:1000
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vxlan_group 224.0.0.1
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_security_group True
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

yum install -y openstack-neutron-lbaas
openstack-config --set /etc/neutron/neutron.conf DEFAULT service_plugins router,lbaas
openstack-config --set /etc/neutron/lbaas_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
openstack-config --set /etc/neutron/lbaas_agent.ini DEFAULT device_driver neutron_lbaas.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver
openstack-config --set /etc/neutron/lbaas_agent.ini haproxy user_group haproxy

yum install -y openstack-neutron-fwaas
openstack-config --set /etc/neutron/neutron.conf DEFAULT service_plugins router,firewall,lbaas
openstack-config --set /etc/neutron/fwaas_driver.ini fwaas enabled True
openstack-config --set /etc/neutron/fwaas_driver.ini fwaas driver neutron_fwaas.services.firewall.drivers.linux.iptables_fwaas.IptablesFwaasDriver

#controller-1
#---------------------------------------------------------------------

neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head

pcs resource create neutron-server systemd:neutron-server op start timeout=90 --clone interleave=true

pcs constraint order start openstack-keystone-clone then neutron-server-clone

pcs property set stonith-enabled=false

#---------------------------------------------------------------------

systemctl start openvswitch
systemctl enable openvswitch

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth1
NAME=eth1
DEVICE=eth1
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
BOOTPROTO=none
ONBOOT=yes
EOF

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-br-ex
NAME=br-ex
DEVICE=br-ex
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
BOOTPROTO=none
ONBOOT=yes
EOF

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth2
NAME=eth2
DEVICE=eth2
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-eth2
BOOTPROTO=none
ONBOOT=yes
EOF

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-br-eth2
NAME=br-eth2
DEVICE=br-eth2
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
BOOTPROTO=none
ONBOOT=yes
EOF

systemctl restart network


openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent tunnel_types vxlan
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent vxlan_udp_port 4789
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs local_ip \
$(ip addr | grep 'eth3:' -A2 | tail -n1 | awk '{print $2}' | awk -F '/' '{print $1}')
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs enable_tunneling True
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs integration_bridge br-int
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs tunnel_bridge br-tun
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings physnet1:br-eth2,extnet:br-ex
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent l2_population False

openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_strategy keystone
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_url http://controller-vip:35357/v2.0
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_host controller-vip
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_region regionOne
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT admin_tenant_name services
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT admin_user neutron
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT admin_password neutron1234
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_ip controller-vip
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_port 8775
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret metadata1234
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT metadata_workers 4
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT metadata_backlog 2048

openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT dnsmasq_config_file /etc/neutron/dnsmasq-neutron.conf
echo "dhcp-option-force=26,1400" > /etc/neutron/dnsmasq-neutron.conf
chown root:neutron /etc/neutron/dnsmasq-neutron.conf
chmod 644 /etc/neutron/dnsmasq-neutron.conf

openstack-config --set /etc/neutron/l3_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
openstack-config --set /etc/neutron/l3_agent.ini DEFAULT handle_internal_only_routers True
openstack-config --set /etc/neutron/l3_agent.ini DEFAULT send_arp_for_ha 3
openstack-config --set /etc/neutron/l3_agent.ini DEFAULT metadata_ip controller-vip
openstack-config --set /etc/neutron/l3_agent.ini DEFAULT external_network_bridge br-ex

mkdir /etc/systemd/system/neutron-server.service.d
cat > /etc/systemd/system/neutron-server.service.d/restart.conf << EOF
[Service]
Restart=on-failure
EOF
systemctl daemon-reload

#controller-1
#---------------------------------------------------------------------

pcs property set stonith-enabled=true

pcs resource cleanup

pcs resource create neutron-scale ocf:neutron:NeutronScale --clone globally-unique=true clone-max=3 interleave=true

pcs resource create neutron-ovs-cleanup ocf:neutron:OVSCleanup --clone interleave=true
pcs resource create neutron-netns-cleanup ocf:neutron:NetnsCleanup --clone interleave=true

pcs resource create neutron-openvswitch-agent  systemd:neutron-openvswitch-agent --clone interleave=true
pcs resource create neutron-dhcp-agent systemd:neutron-dhcp-agent --clone interleave=true
pcs resource create neutron-l3-agent systemd:neutron-l3-agent --clone interleave=true
pcs resource create neutron-metadata-agent systemd:neutron-metadata-agent  --clone interleave=true
pcs resource create neutron-lbaas-agent systemd:neutron-lbaas-agent  --clone interleave=true

pcs constraint order start neutron-scale-clone then neutron-ovs-cleanup-clone
pcs constraint colocation add neutron-ovs-cleanup-clone with neutron-scale-clone
pcs constraint order start neutron-ovs-cleanup-clone then neutron-netns-cleanup-clone
pcs constraint colocation add neutron-netns-cleanup-clone with neutron-ovs-cleanup-clone
pcs constraint order start neutron-netns-cleanup-clone then neutron-openvswitch-agent-clone
pcs constraint colocation add neutron-openvswitch-agent-clone with neutron-netns-cleanup-clone
pcs constraint order start neutron-openvswitch-agent-clone then neutron-dhcp-agent-clone
pcs constraint colocation add neutron-dhcp-agent-clone with neutron-openvswitch-agent-clone
pcs constraint order start neutron-dhcp-agent-clone then neutron-l3-agent-clone
pcs constraint colocation add neutron-l3-agent-clone with neutron-dhcp-agent-clone
pcs constraint order start neutron-l3-agent-clone then neutron-metadata-agent-clone
pcs constraint colocation add neutron-metadata-agent-clone with neutron-l3-agent-clone
pcs constraint order start neutron-metadata-agent-clone then neutron-lbaas-agent-clone
pcs constraint colocation add neutron-lbaas-agent-clone with neutron-metadata-agent-clone

pcs constraint order start neutron-server-clone then neutron-scale-clone

sleep 10
source ~/keystonerc_admin
neutron net-create external_network --provider:network_type flat --provider:physical_network extnet --router:external --shared
neutron subnet-create --name external_subnet --enable_dhcp=False --allocation-pool=start=192.168.0.140,end=192.168.0.145 --gateway=192.168.0.1 external_network 192.168.0.0/23
neutron router-create router1
neutron router-gateway-set router1 external_network
neutron net-create private_network1
neutron net-create private_network2 --provider:network_type vxlan
neutron subnet-create --name private_subnet1 private_network1 10.0.1.0/24
neutron subnet-create --name private_subnet2 private_network2 10.0.2.0/24
neutron router-interface-add router1 private_subnet1
neutron router-interface-add router1 private_subnet2

#---------------------------------------------------------------------
```

----------

### 安装配置 Nova
```
yum install -y openstack-nova-console openstack-nova-novncproxy openstack-utils openstack-nova-api openstack-nova-conductor openstack-nova-scheduler python-cinderclient python-memcached

openstack-config --set /etc/nova/nova.conf DEFAULT memcached_servers controller-1:11211,controller-2:11211,controller-3:11211
openstack-config --set /etc/nova/nova.conf DEFAULT novncproxy_host $(hostname -s)
openstack-config --set /etc/nova/nova.conf vnc novncproxy_base_url http://192.168.0.130:6080/vnc_auto.html
openstack-config --set /etc/nova/nova.conf vnc vncserver_proxyclient_address $(hostname -s)
openstack-config --set /etc/nova/nova.conf vnc vncserver_listen $(hostname -s)
openstack-config --set /etc/nova/nova.conf database connection mysql://nova:nova1234@controller-vip/nova
openstack-config --set /etc/nova/nova.conf database max_retries -1
openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
openstack-config --set /etc/nova/nova.conf DEFAULT osapi_compute_listen $(hostname -s)
openstack-config --set /etc/nova/nova.conf DEFAULT metadata_host \
$(ip addr | grep 'eth0:' -A2 | tail -n1 | awk '{print $2}' | awk -F '/' '{print $1}')
openstack-config --set /etc/nova/nova.conf DEFAULT metadata_listen \
$(ip addr | grep 'eth0:' -A2 | tail -n1 | awk '{print $2}' | awk -F '/' '{print $1}')
openstack-config --set /etc/nova/nova.conf DEFAULT metadata_listen_port 8775
openstack-config --set /etc/nova/nova.conf DEFAULT glance_host controller-vip
openstack-config --set /etc/nova/nova.conf DEFAULT network_api_class nova.network.neutronv2.api.API
openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
openstack-config --set /etc/nova/nova.conf libvirt vif_driver nova.virt.libvirt.vif.LibvirtGenericVIFDriver
openstack-config --set /etc/nova/nova.conf DEFAULT security_group_api neutron
openstack-config --set /etc/nova/nova.conf cinder cinder_catalog_info volume:cinder:internalURL
openstack-config --set /etc/nova/nova.conf conductor use_local false
openstack-config --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_ha_queues True
openstack-config --set /etc/nova/nova.conf neutron service_metadata_proxy True
openstack-config --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret metadata1234
openstack-config --set /etc/nova/nova.conf neutron url http://controller-vip:9696/
openstack-config --set /etc/nova/nova.conf neutron project_domain_id default
openstack-config --set /etc/nova/nova.conf neutron project_name services
openstack-config --set /etc/nova/nova.conf neutron user_domain_id default
openstack-config --set /etc/nova/nova.conf neutron username neutron
openstack-config --set /etc/nova/nova.conf neutron password neutron1234
openstack-config --set /etc/nova/nova.conf neutron auth_url http://controller-vip:35357/
openstack-config --set /etc/nova/nova.conf neutron auth_uri http://controller-vip:5000/
openstack-config --set /etc/nova/nova.conf neutron auth_plugin password
openstack-config --set /etc/nova/nova.conf neutron region_name regionOne

openstack-config --set /etc/nova/nova.conf DEFAULT scheduler_host_subset_size 30
openstack-config --set /etc/nova/api-paste.ini filter:authtoken auth_plugin password
openstack-config --set /etc/nova/api-paste.ini filter:authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/nova/api-paste.ini filter:authtoken username nova
openstack-config --set /etc/nova/api-paste.ini filter:authtoken password nova1234
openstack-config --set /etc/nova/api-paste.ini filter:authtoken project_name services
openstack-config --set /etc/nova/api-paste.ini filter:authtoken auth_uri http://controller-vip:5000/

openstack-config --set /etc/nova/nova.conf libvirt virt_type qemu

#controller-1
#---------------------------------------------------------------------

su nova -s /bin/sh -c "nova-manage db sync"

pcs resource create openstack-nova-consoleauth systemd:openstack-nova-consoleauth --clone interleave=true
pcs resource create openstack-nova-novncproxy systemd:openstack-nova-novncproxy --clone interleave=true
pcs resource create openstack-nova-api systemd:openstack-nova-api --clone interleave=true
pcs resource create openstack-nova-scheduler systemd:openstack-nova-scheduler --clone interleave=true
pcs resource create openstack-nova-conductor systemd:openstack-nova-conductor --clone interleave=true

pcs constraint order start openstack-nova-consoleauth-clone then openstack-nova-novncproxy-clone
pcs constraint colocation add openstack-nova-novncproxy-clone with openstack-nova-consoleauth-clone

pcs constraint order start openstack-nova-novncproxy-clone then openstack-nova-api-clone
pcs constraint colocation add openstack-nova-api-clone with openstack-nova-novncproxy-clone

pcs constraint order start openstack-nova-api-clone then openstack-nova-scheduler-clone
pcs constraint colocation add openstack-nova-scheduler-clone with openstack-nova-api-clone

pcs constraint order start openstack-nova-scheduler-clone then openstack-nova-conductor-clone
pcs constraint colocation add openstack-nova-conductor-clone with openstack-nova-scheduler-clone

pcs constraint order start openstack-keystone-clone then openstack-nova-consoleauth-clone

#---------------------------------------------------------------------
```

----------

### 安装配置 Ceilometer
```
yum install -y openstack-ceilometer-api openstack-ceilometer-central openstack-ceilometer-collector openstack-ceilometer-common openstack-ceilometer-alarm python-ceilometer python-ceilometerclient python-redis

openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken auth_uri http://controller-vip:5000/
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken username ceilometer
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken password ceilometer1234
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken project_name services
openstack-config --set /etc/ceilometer/ceilometer.conf DEFAULT memcache_servers controller-1:11211,controller-2:11211,controller-3:11211
openstack-config --set /etc/ceilometer/ceilometer.conf oslo_messaging_rabbit rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/ceilometer/ceilometer.conf oslo_messaging_rabbit rabbit_ha_queues true
openstack-config --set /etc/ceilometer/ceilometer.conf publisher telemetry_secret ceilometersecret
openstack-config --set /etc/ceilometer/ceilometer.conf service_credentials os_auth_url http://controller-vip:5000/v2.0 
openstack-config --set /etc/ceilometer/ceilometer.conf service_credentials os_tenant_name services
openstack-config --set /etc/ceilometer/ceilometer.conf service_credentials os_username ceilometer
openstack-config --set /etc/ceilometer/ceilometer.conf service_credentials os_password ceilometer1234
openstack-config --set /etc/ceilometer/ceilometer.conf database connection mongodb://controller-1,controller-2,controller-3:27017/ceilometer?replicaSet=ceilometer
openstack-config --set /etc/ceilometer/ceilometer.conf database max_retries -1
openstack-config --set /etc/ceilometer/ceilometer.conf database metering_time_to_live 432000
openstack-config --set /etc/ceilometer/ceilometer.conf api host $(hostname -s)
openstack-config --set /etc/ceilometer/ceilometer.conf coordination backend_url 'redis://controller-1:26379?sentinel=mymaster&sentinel_fallback=controller-2:26379&sentinel_fallback=controller-3:26379'

#controller-1
#---------------------------------------------------------------------

pcs resource create openstack-ceilometer-central systemd:openstack-ceilometer-central --clone interleave=true
pcs resource create openstack-ceilometer-collector systemd:openstack-ceilometer-collector --clone interleave=true
pcs resource create openstack-ceilometer-api systemd:openstack-ceilometer-api --clone interleave=true
pcs resource create openstack-ceilometer-delay Delay startdelay=10 --clone interleave=true
pcs resource create openstack-ceilometer-alarm-evaluator systemd:openstack-ceilometer-alarm-evaluator --clone interleave=true
pcs resource create openstack-ceilometer-alarm-notifier systemd:openstack-ceilometer-alarm-notifier --clone interleave=true
pcs resource create openstack-ceilometer-notification systemd:openstack-ceilometer-notification  --clone interleave=true

pcs constraint order start openstack-ceilometer-central-clone then openstack-ceilometer-collector-clone
pcs constraint order start openstack-ceilometer-collector-clone then openstack-ceilometer-api-clone
pcs constraint colocation add openstack-ceilometer-api-clone with openstack-ceilometer-collector-clone
pcs constraint order start openstack-ceilometer-api-clone then openstack-ceilometer-delay-clone
pcs constraint colocation add openstack-ceilometer-delay-clone with openstack-ceilometer-api-clone
pcs constraint order start openstack-ceilometer-delay-clone then openstack-ceilometer-alarm-evaluator-clone
pcs constraint colocation add openstack-ceilometer-alarm-evaluator-clone with openstack-ceilometer-delay-clone
pcs constraint order start openstack-ceilometer-alarm-evaluator-clone then openstack-ceilometer-alarm-notifier-clone
pcs constraint colocation add openstack-ceilometer-alarm-notifier-clone with openstack-ceilometer-alarm-evaluator-clone
pcs constraint order start openstack-ceilometer-alarm-notifier-clone then openstack-ceilometer-notification-clone
pcs constraint colocation add openstack-ceilometer-notification-clone with openstack-ceilometer-alarm-notifier-clone

pcs constraint order start mongod-clone then openstack-ceilometer-central-clone
pcs constraint order start openstack-keystone-clone then openstack-ceilometer-central-clone

#---------------------------------------------------------------------
```

----------

### 安装配置 Horizon
```
yum install -y mod_wsgi httpd mod_ssl python-memcached openstack-dashboard

#controller-1
#---------------------------------------------------------------------

openssl rand -hex 10 > ~/horizon_secret_key
scp ~/horizon_secret_key controller-2:~/
scp ~/horizon_secret_key controller-3:~/

#---------------------------------------------------------------------

sed -i -e "s#ALLOWED_HOSTS.*#ALLOWED_HOSTS = ['*',]#g" \
-e "s#^CACHES#SESSION_ENGINE = 'django.contrib.sessions.backends.cache'\nCACHES#g#" \
-e "s#locmem.LocMemCache'#memcached.MemcachedCache',\n\t'LOCATION' : [ 'controller-1:11211', 'controller-2:11211', 'controller-3:11211', ]#g" \
-e 's#OPENSTACK_HOST =.*#OPENSTACK_HOST = "controller-vip"#g' \
-e "s#^LOCAL_PATH.*#LOCAL_PATH = '/var/lib/openstack-dashboard'#g" \
-e "s#^SECRET_KEY.*#SECRET_KEY = '$(cat ~/horizon_secret_key)'#g#" \
/etc/openstack-dashboard/local_settings
rm -f ~/horizon_secret_key

#echo "COMPRESS_OFFLINE = True" >> /etc/openstack-dashboard/local_settings
#python /usr/share/openstack-dashboard/manage.py compress

sed -i -e 's/^Listen.*/Listen '$(hostname -s)':80/g' /etc/httpd/conf/httpd.conf
sed -i -e '/^Listen '$(hostname -s)':80/a\Listen localhost:80' /etc/httpd/conf/httpd.conf

cat > /etc/httpd/conf.d/server-status.conf << EOF
<Location /server-status>
    SetHandler server-status
    Order deny,allow
    Deny from all
    Allow from localhost
</Location>
EOF

#controller-1
#---------------------------------------------------------------------

pcs resource create httpd apache --clone interleave=true

pcs resource cleanup
sleep 45
pcs status

#---------------------------------------------------------------------
```

----------

## 计算节点

----------

### 环境说明
```
############################################################################################

# 节点名称：
#                compute-1         compute-2         compute-3

# 配置信息：
#                4核8G/3*300G      4核8G/3*300G      4核8G/3*300G

# 网络信息：
#         eth0   192.168.0.134     192.168.0.135     192.168.0.136     API & Mgmt
#         eth1   access            access            access            L3
#         eth2   trunk             trunk             trunk             vlan data
#         eth3   192.168.103.134   192.168.103.135   192.168.103.136   vxlan data
#         eth4   192.168.104.134   192.168.104.135   192.168.104.136   ceph public
#         eth5   192.168.105.134   192.168.105.135   192.168.105.136   ceph cluster

# 操作系统：
#                CentOS 7

# 服务器角色：
#                Compute nodes

############################################################################################
```

----------

### 环境准备
```
# PXE 安装操作系统（CentOS 7 最小化安装）
## 1. swap : 120G | / : 150G
## 2. 关闭 SELinux、NetworkManager、firewalld、tuned
## 3. root/root1234
## 4. 安装 bash-completion、chrony、tcpdump、telnet、vim、wget
## 5. 配置本地源
## 6. yum clean all && yum update -y
## 7. 安装 nmon
## 8. 重启服务器

#controller-1 & controller-2 & controller-3
#---------------------------------------------------------------------

echo -e "192.168.0.134\tcompute-1" >> /etc/hosts
echo -e "192.168.0.135\tcompute-2" >> /etc/hosts
echo -e "192.168.0.136\tcompute-3" >> /etc/hosts

ssh-copy-id compute-1


ssh-copy-id compute-2

ssh-copy-id compute-3

#---------------------------------------------------------------------

echo -e "\n192.168.0.130\tcontroller-vip" >> /etc/hosts
echo -e "192.168.0.131\tcontroller-1" >> /etc/hosts
echo -e "192.168.0.132\tcontroller-2" >> /etc/hosts
echo -e "192.168.0.133\tcontroller-3" >> /etc/hosts
echo -e "192.168.0.134\tcompute-1" >> /etc/hosts
echo -e "192.168.0.135\tcompute-2" >> /etc/hosts
echo -e "192.168.0.136\tcompute-3" >> /etc/hosts

echo -e "\tStrictHostKeyChecking=no" >> /etc/ssh/ssh_config


sed -i -e "s/^server .*/#&/" /etc/chrony.conf
sed -i -e "s/^allow .*/#&/" /etc/chrony.conf
cat >> /etc/chrony.conf << EOF

server controller-1 iburst
server controller-2 iburst
server controller-3 iburst
EOF

systemctl restart chronyd
sleep 10
chronyc sources -v


```

----------

### 安装配置 Ceph 客户端
```
cat > /etc/yum.repos.d/ceph-infernalis.repo << EOF
[ceph-infernalis-x86_64]
name=Ceph Infernalis X86_64
baseurl=http://192.168.0.100/centos/7/ceph/x86_64/rpm-infernalis/
gpgcheck=0

[ceph-infernalis-noarch]
name=Ceph Infernalis Noarch
baseurl=http://192.168.0.100/centos/7/ceph/noarch/rpm-infernalis/
gpgcheck=0
EOF

cat > /etc/yum.repos.d/epel.repo << EOF
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
baseurl=http://mirrors.ustc.edu.cn/epel/7/x86_64/
failovermethod=priority
enabled=1
gpgcheck=1
gpgkey=http://mirrors.ustc.edu.cn/epel/RPM-GPG-KEY-EPEL-7
EOF

useradd -d /home/ceph -m ceph

echo ceph1234 \
| passwd --stdin ceph

cat << EOF > /etc/sudoers.d/ceph
Defaults:ceph !requiretty

ceph ALL = (root) NOPASSWD: ALL
EOF

chmod 0440 /etc/sudoers.d/ceph

#controller-1
#---------------------------------------------------------------------

su - ceph

ssh-copy-id compute-1


ssh-copy-id compute-2

ssh-copy-id compute-3

cd my-cluster/
ceph-deploy install --no-adjust-repos compute-1 compute-2 compute-3

ceph-deploy --overwrite-conf admin compute-1 compute-2 compute-3

exit

#---------------------------------------------------------------------

yum clean all
mv /etc/yum.repos.d/ceph-infernalis.repo /etc/yum.repos.d/epel.repo /etc/yum.repos.d/bak/


```

----------

### 安装配置 OpenStack 组件
```
cat > /etc/yum.repos.d/openstack-liberty.repo << EOF
[openstack-liberty]
name=OpenStack Liberty
baseurl=http://192.168.0.100/centos/7/cloud/x86_64/openstack-liberty/
gpgcheck=0
EOF

yum install -y openstack-nova-compute openstack-utils python-cinder openstack-neutron-openvswitch openstack-ceilometer-compute python-memcached openstack-neutron

systemctl start openvswitch
systemctl enable openvswitch

systemctl stop libvirtd
systemctl disable libvirtd

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth2
NAME=eth2
DEVICE=eth2
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-eth2
BOOTPROTO=none
ONBOOT=yes
EOF

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-br-eth2
NAME=br-eth2
DEVICE=br-eth2
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
BOOTPROTO=none
ONBOOT=yes
EOF

systemctl restart network


openstack-config --set /etc/nova/nova.conf DEFAULT memcached_servers controller-1:11211,controller-2:11211,controller-3:11211
openstack-config --set /etc/nova/nova.conf vnc vncserver_proxyclient_address $(hostname -s)
openstack-config --set /etc/nova/nova.conf vnc vncserver_listen $(hostname -s)
openstack-config --set /etc/nova/nova.conf vnc novncproxy_base_url http://192.168.0.130:6080/vnc_auto.html
openstack-config --set /etc/nova/nova.conf database connection mysql://nova:nova1234@controller-vip/nova
openstack-config --set /etc/nova/nova.conf database max_retries -1
openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
openstack-config --set /etc/nova/nova.conf glance host controller-vip
openstack-config --set /etc/nova/nova.conf DEFAULT network_api_class nova.network.neutronv2.api.API
openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
openstack-config --set /etc/nova/nova.conf libvirt vif_driver nova.virt.libvirt.vif.LibvirtGenericVIFDriver
openstack-config --set /etc/nova/nova.conf DEFAULT security_group_api neutron
openstack-config --set /etc/nova/nova.conf cinder cinder_catalog_info volume:cinder:internalURL
openstack-config --set /etc/nova/nova.conf conductor use_local false
openstack-config --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_ha_queues True
openstack-config --set /etc/nova/nova.conf neutron service_metadata_proxy True
openstack-config --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret metadata1234
openstack-config --set /etc/nova/nova.conf neutron url http://controller-vip:9696/
openstack-config --set /etc/nova/nova.conf neutron project_domain_id default
openstack-config --set /etc/nova/nova.conf neutron project_name services
openstack-config --set /etc/nova/nova.conf neutron user_domain_id default
openstack-config --set /etc/nova/nova.conf neutron username neutron
openstack-config --set /etc/nova/nova.conf neutron password neutron1234
openstack-config --set /etc/nova/nova.conf neutron auth_url http://controller-vip:35357/
openstack-config --set /etc/nova/nova.conf neutron auth_uri http://controller-vip:5000/
openstack-config --set /etc/nova/nova.conf neutron auth_plugin password
openstack-config --set /etc/nova/nova.conf neutron region_name regionOne
openstack-config --set /etc/nova/api-paste.ini filter:authtoken auth_plugin password
openstack-config --set /etc/nova/api-paste.ini filter:authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/nova/api-paste.ini filter:authtoken username nova
openstack-config --set /etc/nova/api-paste.ini filter:authtoken password nova1234
openstack-config --set /etc/nova/api-paste.ini filter:authtoken project_name services
openstack-config --set /etc/nova/api-paste.ini filter:authtoken auth_uri http://controller-vip:5000/

openstack-config --set /etc/nova/nova.conf libvirt virt_type qemu


#controller-1
#---------------------------------------------------------------------

scp ~/rbd_secret_uuid compute-1:~/
scp ~/rbd_secret_uuid compute-2:~/
scp ~/rbd_secret_uuid compute-3:~/

#---------------------------------------------------------------------

openstack-config --set /etc/nova/nova.conf DEFAULT allow_resize_to_same_host true
openstack-config --set /etc/nova/nova.conf DEFAULT resume_guests_state_on_host_boot true
openstack-config --set /etc/nova/nova.conf libvirt inject_password false
openstack-config --set /etc/nova/nova.conf libvirt inject_key false
openstack-config --set /etc/nova/nova.conf libvirt inject_partition -2
openstack-config --set /etc/nova/nova.conf libvirt live_migration_flag \
"VIR_MIGRATE_UNDEFINE_SOURCE, VIR_MIGRATE_PEER2PEER, VIR_MIGRATE_LIVE, VIR_MIGRATE_PERSIST_DEST, VIR_MIGRATE_TUNNELLED"
openstack-config --set /etc/nova/nova.conf libvirt disk_cachemodes "network=writeback"
openstack-config --set /etc/nova/nova.conf libvirt images_type rbd
openstack-config --set /etc/nova/nova.conf libvirt images_rbd_pool vms
openstack-config --set /etc/nova/nova.conf libvirt images_rbd_ceph_conf /etc/ceph/ceph.conf
#openstack-config --set /etc/nova/nova.conf libvirt hw_disk_discard unmap
openstack-config --set /etc/nova/nova.conf libvirt rbd_user cinder
openstack-config --set /etc/nova/nova.conf libvirt rbd_secret_uuid $(cat ~/rbd_secret_uuid)

ceph auth get-or-create client.cinder | tee /etc/ceph/ceph.client.cinder.keyring

ceph auth get-key client.cinder | tee client.cinder.key

cat << EOF > secret.xml
<secret ephemeral='no' private='no'>
        <uuid>$(cat ~/rbd_secret_uuid)</uuid>
        <usage type='ceph'>
                <name>client.cinder secret</name>
        </usage>
</secret>
EOF

systemctl start libvirtd
virsh secret-define --file secret.xml
virsh secret-set-value --secret $(cat ~/rbd_secret_uuid) --base64 $(cat client.cinder.key)
systemctl stop libvirtd
rm -f client.cinder.key secret.xml rbd_secret_uuid


openstack-config --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller-vip:5000/
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken username neutron
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken password neutron1234
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken project_name services
openstack-config --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_ha_queues true
openstack-config --set /etc/neutron/neutron.conf DEFAULT notification_driver neutron.openstack.common.notifier.rpc_notifier
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent tunnel_types vxlan
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent vxlan_udp_port 4789
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs enable_tunneling True
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs tunnel_id_ranges 100:1000
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs integration_bridge br-int
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs tunnel_bridge br-tun
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs local_ip \
$(ip addr | grep 'eth3:' -A2 | tail -n1 | awk '{print $2}' | awk -F '/' '{print $1}')
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings physnet1:br-eth2
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver 
openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent l2_population False


openstack-config --set /etc/nova/nova.conf DEFAULT instance_usage_audit True
openstack-config --set /etc/nova/nova.conf DEFAULT instance_usage_audit_period hour
openstack-config --set /etc/nova/nova.conf DEFAULT notify_on_state_change vm_and_task_state
openstack-config --set /etc/nova/nova.conf DEFAULT notification_driver nova.openstack.common.notifier.rpc_notifier
sed  -i -e  's/nova.openstack.common.notifier.rpc_notifier/nova.openstack.common.notifier.rpc_notifier\nnotification_driver = ceilometer.compute.nova_notifier/g' /etc/nova/nova.conf
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken auth_uri http://controller-vip:5000/
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken auth_url http://controller-vip:35357/
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken username ceilometer
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken password ceilometer1234
openstack-config --set /etc/ceilometer/ceilometer.conf keystone_authtoken project_name services
openstack-config --set /etc/ceilometer/ceilometer.conf DEFAULT memcache_servers controller-1:11211,controller-2:11211,controller-3:11211
openstack-config --set /etc/ceilometer/ceilometer.conf oslo_messaging_rabbit rabbit_hosts controller-1,controller-2,controller-3
openstack-config --set /etc/ceilometer/ceilometer.conf oslo_messaging_rabbit rabbit_ha_queues true
openstack-config --set /etc/ceilometer/ceilometer.conf publisher telemetry_secret ceilometersecret
openstack-config --set /etc/ceilometer/ceilometer.conf service_credentials os_auth_url http://controller-vip:5000/v2.0
openstack-config --set /etc/ceilometer/ceilometer.conf service_credentials os_tenant_name services
openstack-config --set /etc/ceilometer/ceilometer.conf service_credentials os_username ceilometer
openstack-config --set /etc/ceilometer/ceilometer.conf service_credentials os_password ceilometer1234
openstack-config --set /etc/ceilometer/ceilometer.conf database connection mongodb://controller-1,controller-2,controller-3:27017/ceilometer?replicaSet=ceilometer
openstack-config --set /etc/ceilometer/ceilometer.conf database max_retries -1

openstack-config --set /etc/ceilometer/ceilometer.conf database metering_time_to_live 432000


sed -i "s/^#listen_tls = 0/listen_tls = 0/" /etc/libvirt/libvirtd.conf
sed -i "s/^#listen_tcp = 1/listen_tcp = 1/" /etc/libvirt/libvirtd.conf
sed -i "s/^#auth_tcp = \"sasl\"/auth_tcp = \"none\"/" /etc/libvirt/libvirtd.conf
sed -i "s/^#LIBVIRTD_ARGS=\"--listen\"/LIBVIRTD_ARGS=\"--listen\"/" /etc/sysconfig/libvirtd

usermod -s /bin/bash nova

echo nova1234 \
| passwd --stdin nova

su - nova
pwd
ssh-keygen


cat << EOF > ~/.ssh/config
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
EOF

ssh-copy-id compute-1


ssh-copy-id compute-2

ssh-copy-id compute-3


exit

cat > /etc/sysctl.d/tcp_keepalive.conf << EOF
net.ipv4.tcp_keepalive_intvl = 1
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_time = 5
EOF
sysctl -p /etc/sysctl.d/tcp_keepalive.conf

#controller-1
#---------------------------------------------------------------------

for i in 1 2 3; do ssh compute-$i "mkdir -p /etc/pacemaker/"; scp /etc/pacemaker/authkey compute-$i:/etc/pacemaker/; done

#---------------------------------------------------------------------

yum install -y pacemaker-remote
systemctl start pacemaker_remote
systemctl enable pacemaker_remote

#controller-1
#---------------------------------------------------------------------

pcs resource create nova-evacuate ocf:openstack:NovaEvacuate auth_url=http://controller-vip:35357/v2.0/ username=admin password=admin1234 tenant_name=admin

pcs constraint order start controller-vip then nova-evacuate; done

for i in openstack-glance-api-clone neutron-metadata-agent-clone openstack-nova-conductor-clone; \
do pcs constraint order start $i then nova-evacuate require-all=false; done

pcs resource disable openstack-keystone --wait=600s


for controller in $(cibadmin -Q -o nodes | grep uname | sed s/.*uname..// | awk -F\" '{print $1}'); \
do pcs property set --node ${controller} osprole-controller=true; done

for i in $(cibadmin -Q --xpath //primitive --node-path | tr ' ' '\n' | awk -F "id='" '{print $2}' | awk -F "'" '{print $1}' | uniq); do
    found=0
    if [ -n "$(pcs stonith | awk '{print $1}')" ]; then
        for x in $(pcs stonith | awk '{print $1}'); do
            if [ $x = $i ]; then
                found=1
            fi
	    done
    fi
    if [ $found = 0 ]; then
        pcs constraint location $i rule resource-discovery=exclusive score=0 osprole-controller eq true
    fi
done

pcs resource create neutron-openvswitch-agent-compute systemd:neutron-openvswitch-agent --clone interleave=true --disabled --force
pcs constraint location neutron-openvswitch-agent-compute-clone rule resource-discovery=exclusive score=0 osprole-compute eq true
pcs constraint order start neutron-server-clone then neutron-openvswitch-agent-compute-clone require-all=false

pcs resource create libvirtd-compute systemd:libvirtd --clone interleave=true --disabled --force
pcs constraint location libvirtd-compute-clone rule resource-discovery=exclusive score=0 osprole-compute eq true
pcs constraint order start neutron-openvswitch-agent-compute-clone then libvirtd-compute-clone
pcs constraint colocation add libvirtd-compute-clone with neutron-openvswitch-agent-compute-clone

pcs resource create ceilometer-compute systemd:openstack-ceilometer-compute --clone interleave=true --disabled --force
pcs constraint location ceilometer-compute-clone rule resource-discovery=exclusive score=0 osprole-compute eq true
pcs constraint order start openstack-ceilometer-notification-clone then ceilometer-compute-clone require-all=false
pcs constraint order start libvirtd-compute-clone then ceilometer-compute-clone
pcs constraint colocation add ceilometer-compute-clone with libvirtd-compute-clone

pcs resource create nova-compute-checkevacuate ocf:openstack:nova-compute-wait auth_url=http://controller-vip:35357/v2.0/ username=admin password=admin1234 tenant_name=admin op start timeout=300 --clone interleave=true --disabled --force
pcs constraint location nova-compute-checkevacuate-clone rule resource-discovery=exclusive score=0 osprole-compute eq true
pcs constraint order start openstack-nova-conductor-clone then nova-compute-checkevacuate-clone require-all=false

pcs resource create nova-compute systemd:openstack-nova-compute --clone interleave=true --disabled --force
pcs constraint location nova-compute-clone rule resource-discovery=exclusive score=0 osprole-compute eq true
pcs constraint order start nova-compute-checkevacuate-clone then nova-compute-clone require-all=true
pcs constraint order start nova-compute-clone then nova-evacuate require-all=false
pcs constraint order start libvirtd-compute-clone then nova-compute-clone
pcs constraint colocation add nova-compute-clone with libvirtd-compute-clone


for i in 1 2 3; do \
pcs resource create compute-$i ocf:pacemaker:remote reconnect_interval=60 op monitor interval=20; \
pcs property set --node compute-$i osprole-compute=true; done

pcs stonith update fence-server pcmk_host_map="controller-1:test1;controller-2:test2;controller-3:test3;compute-1:test4;compute-2:test5;compute-3:test6"

pcs stonith create fence-compute fence_compute auth-url=http://controller-vip:35357/v2.0/ login=admin passwd=admin1234 tenant-name=admin record-only=1 action=off --force

for i in 1 2 3; do \
pcs stonith level add 1 compute-$i fence-server,fence-compute; done

for i in openstack-keystone neutron-openvswitch-agent-compute libvirtd-compute ceilometer-compute nova-compute-checkevacuate nova-compute; do \
pcs resource enable $i; done

sleep 120
pcs resource cleanup
sleep 60
pcs status

#---------------------------------------------------------------------
```

----------

  [1]: http://192.168.0.100/architecture-1.jpg
  [2]: http://192.168.0.100/architecture-2.jpg
