---
layout: post
title: Install - OpenStack流程
category: 技术
---


RDO  install OpenStack Rocky

关闭防火墙
systemctl restart network

systemctl stop firewalld

systemctl disable firewalld

setenforce 0

sed -i 's/=enforcing/=disabled/' /etc/selinux/config

yum install yum-fastestmirror 


systemctl disable firewalld
systemctl stop firewalld
systemctl disable NetworkManager
systemctl stop NetworkManager
systemctl enable network
systemctl start network



设置主机名
hostnamectl set-hostname controller

hostnamectl set-hostname compute

 
添加主机映射
cat << EOF >> /etc/hosts
192.168.56.125 controller
EOF



官网是yum安装centos-release-openstack-rocky，配置阿里的源
cat << EOF >> /etc/yum.repos.d/openstack.repo
[openstack-rocky]
name=openstack-rocky
baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-rocky/
enabled=1
gpgcheck=0
[qume-kvm]
name=qemu-kvm
baseurl= https://mirrors.aliyun.com/centos/7/virt/x86_64/kvm-common/
enabled=1
gpgcheck=0
EOF



On RHEL:
$ sudo yum install -y https://www.rdoproject.org/repos/rdo-release.rpm
$ sudo yum update -y
$ sudo yum install -y openstack-packstack
$ sudo packstack --allinone



yum update -y
yum install -y centos-release-openstack-rocky
yum update -y
yum install -y openstack-packstack
packstack --allinone




CentOS7.5安装OpenStack Rocky版本
https://docs.openstack.org/install-guide/
密码的设置，都设置成000000
 
预备
关闭防火墙
systemctl restart network

systemctl stop firewalld

systemctl disable firewalld

setenforce 0

sed -i 's/=enforcing/=disabled/' /etc/selinux/config


CentOS7 Failed to start LSB: Bring up/down networking.解决方法
修改网卡mac地址 08:00:27:ef:be:a2
 

更新软件包
yum upgrade -y

更新完成后重启系统
reboot
 

设置主机名
hostnamectl set-hostname controller

hostnamectl set-hostname compute

 

添加主机映射

cat << EOF >> /etc/hosts
192.168.56.122 controller
192.168.56.123 compute
192.168.56.111 storage
EOF

 

配置时间同步
controller节点
安装软件包
yum install -y chrony

 

编辑/etc/chrony.conf文件
server controller iburst
allow 192.168.0.0/16

 

启动服务
systemctl start chronyd
systemctl enable chronyd

 

compute节点
安装软件包
yum install -y chrony
 

编辑/etc/chrony.conf文件
server controller iburst
 

启动服务
systemctl start chronyd
systemctl enable chronyd
 

###取消掉###
配置OpenStack-rocky的yum源文件
官网是yum安装centos-release-openstack-rocky，手动配置了阿里的源


cat << EOF >> /etc/yum.repos.d/openstack.repo
[openstack-rocky]
name=openstack-rocky
baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-rocky/
enabled=1
gpgcheck=0
[qume-kvm]
name=qemu-kvm
baseurl= https://mirrors.aliyun.com/centos/7/virt/x86_64/kvm-common/
enabled=1
gpgcheck=0
EOF


安装OpenStack客户端和selinux服务
yum install -y python-openstackclient openstack-selinux

安装数据库服务
在controller节点安装数据库
yum install -y mariadb mariadb-server python2-PyMySQL
 

修改数据库配置文件
新建数据库配置文件
/etc/my.cnf.d/openstack.cnf
添加以下内容

[mysqld]
bind-address = 192.168.56.122
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8

 

启动数据库服务
systemctl enable mariadb.service
systemctl start mariadb.service

 
设置数据库密码
运行mysql_secure_installation命令，创建数据库root密码

mysql_secure_installation

安装消息队列服务
在controller节点安装rabbitmq-server
yum install -y rabbitmq-server -y
 

启动消息队列服务
systemctl start rabbitmq-server.service
systemctl enable rabbitmq-server.service

route add default gw 192.168.56.1

添加openstack用户
rabbitmqctl add_user openstack 000000


设置openstack用户最高权限
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

 

安装memcached 服务
在controller节点上安装memcached
yum install -y memcached

 

修改memcached配置文件
编辑/etc/sysconfig/memcached，修改以下内容

修改OPTIONS="-l 127.0.0.1,::1"为

OPTIONS="-l 127.0.0.1,::1,controller"

 

启动memcached服务
systemctl start memcached.service
systemctl enable memcached.service

 

Etcd是集群中的一个十分重要的组件，用于保存集群所有的网络配置和对象的状态信息
安装etcd服务
在controller节点上安装etcd服务
yum install etcd -y

修改etcd配置文件，使其他节点能够访问
编辑/etc/etcd/etcd.conf，在各自的位置修改以下内容

#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.56.120:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.56.120:2379"
ETCD_NAME="controller"


#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.56.120:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.56.120:2379"
ETCD_INITIAL_CLUSTER="controller=http://192.168.56.120:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
 


启动etcd服务
systemctl start etcd
systemctl enable etcd
 

集群健康检查
etcdctl cluster-health
 

安装keystone服务
创建数据库
mysql -uroot -p000000
 

CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '000000';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '000000';
FLUSH PRIVILEGES;
 
安装软件包
yum install openstack-keystone httpd mod_wsgi -y
 

编辑配置文件
/etc/keystone/keystone.conf

[database]
connection = mysql+pymysql://keystone:000000@controller/keystone

[token]
provider = fernet
 

同步数据库
su -s /bin/sh -c "keystone-manage db_sync" keystone
 

初始化fernet key库
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
 

引导身份认证
keystone-manage bootstrap --bootstrap-password 000000 \
--bootstrap-admin-url http://controller:5000/v3/ \
--bootstrap-internal-url http://controller:5000/v3/ \
--bootstrap-public-url http://controller:5000/v3/ \
--bootstrap-region-id RegionOne

 
编辑httpd配置文件
/etc/httpd/conf/httpd.conf
ServerName controller

 
创建文件链接
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
 

启动httpd服务
systemctl restart httpd

systemctl enable httpd


 

编写环境变量脚本admin-openrc
export OS_USERNAME=admin
export OS_PASSWORD=000000
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
 

创建service项目
openstack project create --domain default \
--description "Service Project" service


验证
openstack user list


openstack token issue



Failed to discover available identity versions when contacting http://controller:5000/v3. Attempting to parse version from URL.
是网络没有通。




//------------------------------------------------
Install glance
  
mysql -u root -p000000

CREATE DATABASE glance;

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';

FLUSH PRIVILEGES;

. admin-openrc


update user set password=password("000000") where user="glance";


openstack user create --domain default --password-prompt glance


openstack role add --project service --user glance admin


openstack service create --name glance \
  --description "OpenStack Image" image


openstack endpoint create --region RegionOne \
  image public http://controller:9292


openstack endpoint create --region RegionOne \
  image internal http://controller:9292


openstack endpoint create --region RegionOne \
  image admin http://controller:9292


yum install openstack-glance
Edit the /etc/glance/glance-api.conf file and complete the following actions:


[database]
# ...
connection = mysql+pymysql://glance:000000@controller/glance

In the [keystone_authtoken] and [paste_deploy] sections, configure Identity service access:

[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/



Edit the /etc/glance/glance-registry.conf file and complete the following actions:



[database]
# ...
connection = mysql+pymysql://glance:000000@controller/glance


[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = 000000

[paste_deploy]
# ...
flavor = keystone


# 
su -s /bin/sh -c "glance-manage db_sync" glance



systemctl enable openstack-glance-api.service \
  openstack-glance-registry.service
systemctl start openstack-glance-api.service \
  openstack-glance-registry.service


 

 

 


验证
. admin-openrc

yum -y install wget
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img


I had done the openstack service create --name glance.... twice So, I deleted it with "openstack service delete {id} where {id} was found via the openstack service list



openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public



openstack image list

//---------------------------------------------------------

安装nova服务
controller节点
创建数据库
mysql -u root -p000000

drop DATABASE nova_api;
drop DATABASE nova;
drop DATABASE nova_cell0;
drop DATABASE placement;

 

CREATE DATABASE nova_api;

CREATE DATABASE nova;

CREATE DATABASE nova_cell0;

CREATE DATABASE placement;
 
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
IDENTIFIED BY '000000';

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
IDENTIFIED BY '000000';
 

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY '000000';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
IDENTIFIED BY '000000';

 
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
IDENTIFIED BY '000000';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
IDENTIFIED BY '000000';

GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
IDENTIFIED BY '000000';

GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
IDENTIFIED BY '000000';

flush privileges;
 

创建相关用户、服务
[root@controller ~]# 
openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:

[root@controller ~]# 
openstack role add --project service --user nova admin

openstack service create --name nova \
--description "OpenStack Compute" compute


openstack endpoint create --region RegionOne \
compute public http://controller:8774/v3


openstack endpoint create --region RegionOne \
compute internal http://controller:8774/v3

openstack endpoint create --region RegionOne \
compute admin http://controller:8774/v3


openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:


[root@controller ~]# 
openstack role add --project service --user placement admin

[root@controller ~]#  
openstack service create --name placement \
--description "Placement API" placement


[root@controller ~]# 
openstack endpoint create --region RegionOne \
placement public http://controller:8778


openstack endpoint create --region RegionOne \
placement internal http://controller:8778


openstack endpoint create --region RegionOne \
placement admin http://controller:8778

 

安装软件包
[root@controller ~]# 
yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api -y

 

 

编辑配置文件/etc/nova/nova.conf
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:000000@controller

[api_database]
connection = mysql+pymysql://nova:000000@controller/nova_api

[database]
connection = mysql+pymysql://nova:000000@controller/nova
 

[placement_database]
connection = mysql+pymysql://placement:000000@controller/placement


[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = 000000
 

[DEFAULT]
my_ip = 192.168.56.113

[DEFAULT]
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = 000000

 

编辑/etc/httpd/conf.d/00-nova-placement-api.conf，添加以下内容

<Directory /usr/bin>
<IfVersion >= 2.4>
Require all granted
</IfVersion>
<IfVersion < 2.4>
Order allow,deny
Allow from all
</IfVersion>
</Directory>

 

重启httpd服务
[root@controller ~]# 
systemctl restart httpd
 

同步nova_api数据库
[root@controller ~]# 
su -s /bin/sh -c "nova-manage api_db sync" nova


su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
 

su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

 
su -s /bin/sh -c "nova-manage db sync" nova



验证cell0和cell1注册成功
[root@controller ~]# 
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova


启动服务
[root@controller ~]# 
systemctl restart openstack-nova-api.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service openstack-nova-conductor

[root@controller ~]# 
systemctl enable openstack-nova-api.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service openstack-nova-conductor


systemctl stop openstack-nova-api.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service openstack-nova-conductor


官网没有启动nova-conductor服务，这个服务是交互数据库的，如果不启动这个服务，虚拟机创建不成功

 

compute节点
安装软件包
[root@compute ~]# 
yum install openstack-nova-compute -y
 

编辑配置文件/etc/nova/nova.conf
[DEFAULT]
enabled_apis = osapi_compute,metadata

 

[DEFAULT]
transport_url = rabbit://openstack:000000@controller

 

[api]
auth_strategy = keystone
 

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = 000000

 

[DEFAULT]
my_ip = 192.168.100.20

 

[DEFAULT]
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

 

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http:// 192.168.100.10:6080/vnc_auto.html

 

[glance]
api_servers = http://controller:9292

 

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

 

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = 000000

 

检查是否支持虚拟化
egrep -c '(vmx|svm)' /proc/cpuinfo

如果等于0，则要在/etc/nova/nova.conf的[libvirt]下添加以下参数

[libvirt]
virt_type = qemu

 

启动服务
systemctl start libvirtd.service openstack-nova-compute.service

systemctl enable libvirtd.service openstack-nova-compute.service


controller节点
确认数据库中有计算节点
# 
. admin-openrc

[root@controller ~]# 
openstack compute service list --service nova-compute

 

发现计算节点
[root@controller ~]# 
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

启动服务并设置开机启动

systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
验证

openstack compute service list --service nova-compute

 

如果想要自动发现新compute节点，可以在/etc/nova/nova.conf的[scheduler]下添加以下参数

[scheduler]
discover_hosts_in_cells_interval = 300


 
安装neutron服务
controller节点
创建数据库
[root@controller ~]# 
mysql -uroot -p000000

drop database neutron;
 
create database neutron;

MariaDB [(none)]> 
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
IDENTIFIED BY '000000';

GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
IDENTIFIED BY '000000';

FLUSH PRIVILEGES;

创建用户、服务
[root@controller ~]# 
openstack user create --domain default --password-prompt neutron

User Password:
Repeat User Password:


[root@controller ~]# 
openstack role add --project service --user neutron admin

[root@controller ~]# 
openstack service create --name neutron \
--description "OpenStack Networking" network

openstack endpoint create --region RegionOne \
network public http://controller:9696


openstack endpoint create --region RegionOne \
network internal http://controller:9696


openstack endpoint create --region RegionOne \
network admin http://controller:9696


配置provider network网络
安装软件包
[root@controller ~]# 
yum install openstack-neutron openstack-neutron-ml2  openstack-neutron-linuxbridge ebtables -y

 

编辑/etc/neutron/neutron.conf配置文件
[database]
connection = mysql+pymysql://neutron:000000@controller/neutron


[DEFAULT]
core_plugin = ml2
service_plugins =

 

[DEFAULT]
transport_url = rabbit://openstack:000000@controller

 
[DEFAULT]
auth_strategy = keystone
 

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 000000
 

[DEFAULT]
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

 

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = 000000
 

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

 

编辑配置文件/etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = flat,vlan

[ml2]
tenant_network_types =

 

[ml2]
mechanism_drivers = linuxbridge
 

[ml2]
extension_drivers = port_security
 

[ml2_type_flat]
flat_networks = provider
 

[securitygroup]
enable_ipset = true

 

编辑/etc/neutron/plugins/ml2/linuxbridge_agent.ini配置文件
[linux_bridge]
physical_interface_mappings = provider:eth1
 

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

 

编辑配置文件/etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
 

配置Self-service网络
安装软件包
# 
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y

配置/etc/neutron/neutron.conf文件
[database]
connection = mysql+pymysql://neutron:000000@controller/neutron
 

[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
 

[DEFAULT]
transport_url = rabbit://openstack:000000@controller
 

[DEFAULT]
auth_strategy = keystone
 

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 000000

 

[DEFAULT]
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
 

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = 000000
 

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp


编辑/etc/neutron/plugins/ml2/ml2_conf.ini文件
[ml2]
type_drivers = flat,vlan,vxlan
 

[ml2]
tenant_network_types = vxlan
 

[ml2]
mechanism_drivers = linuxbridge,l2population
 

[ml2]
extension_drivers = port_security
 

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000

[securitygroup]
enable_ipset = true
 

编辑/etc/neutron/plugins/ml2/linuxbridge_agent.ini文件
[linux_bridge]
physical_interface_mappings = provider:eth1

[vxlan]
enable_vxlan = true
local_ip = 192.168.200.10
l2_population = true
 

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
 

编辑/etc/neutron/l3_agent.ini文件
[DEFAULT]
interface_driver = linuxbridge
 

编辑/etc/neutron/dhcp_agent.ini文件
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
 

编辑/etc/neutron/metadata_agent.ini文件
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
 

编辑/etc/nova/nova.conf文件
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = 000000
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET

 

创建链接
[root@controller ~]# 
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
 

同步数据库
[root@controller ~]# 
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron 

 

启动服务
[root@controller ~]# 
systemctl restart openstack-nova-api

[root@controller ~]# 
systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service

[root@controller ~]# 
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service

Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-server.service to /usr/lib/systemd/system/neutron-server.service.

Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service to /usr/lib/systemd/system/neutron-linuxbridge-agent.service.

Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-dhcp-agent.service to /usr/lib/systemd/system/neutron-dhcp-agent.service.

Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-metadata-agent.service to /usr/lib/systemd/system/neutron-metadata-agent.service.

如果选择了Self-service网络，还需要启动这个服务

[root@controller ~]# 
systemctl start neutron-l3-agent.service

[root@controller ~]# 
systemctl enable neutron-l3-agent.service

Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-l3-agent.service to /usr/lib/systemd/system/neutron-l3-agent.service.

 

compute节点
安装软件包
[root@compute ~]# 
yum install openstack-neutron-linuxbridge ebtables ipset -y
 

编辑配置/etc/neutron/neutron.conf文件
[DEFAULT]
transport_url = rabbit://openstack:000000@controller
 

[DEFAULT]
auth_strategy = keystone
 

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 000000

 
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp


配置provider网络
编辑配置/etc/neutron/plugins/ml2/linuxbridge_agent.ini文件
[linux_bridge]
physical_interface_mappings = provider:eth1
 

[vxlan]
enable_vxlan = false
 

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
 

配置Self-service网络
编辑配置/etc/neutron/plugins/ml2/linuxbridge_agent.ini文件
[linux_bridge]
physical_interface_mappings = provider:enp0s8
 

[vxlan]
enable_vxlan = true
local_ip = 192.168.56.112
l2_population = true
 

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
 

配置nova配置/etc/nova/nova.conf文件
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = 000000

 

启动服务
[root@compute ~]# 
systemctl restart openstack-nova-compute

[root@compute ~]# 
systemctl restart neutron-linuxbridge-agent.service

[root@compute ~]# 
systemctl enable neutron-linuxbridge-agent.service

Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service to /usr/lib/systemd/system/neutron-linuxbridge-agent.service.

 

验证
[root@controller ~]# 
openstack network agent list

 

安装dashboard
controller节点
安装软件包
[root@controller ~]
yum install -y openstack-dashboard
 

编辑配置文件/etc/openstack-dashboard/local_settings
OPENSTACK_HOST = "controller"

ALLOWED_HOSTS = ['*', 'localhost']

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
 

CACHES = {
'default': {
'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
'LOCATION': 'controller:11211',

}
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
"identity": 3,
"image": 2,
"volume": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {

...

'enable_router': False,

'enable_quotas': False,

'enable_distributed_router': False,

'enable_ha_router': False,

'enable_lb': False,

'enable_firewall': False,

'enable_vpn': False,

'enable_fip_topology_check': False,

}
 

编辑/etc/httpd/conf.d/openstack-dashboard.conf
WSGIApplicationGroup %{GLOBAL}

 
启动服务
[root@controller ~]# 
systemctl restart httpd.service memcached.service
 

验证
浏览器打开
192.168.56.113/dashboard


创建虚拟机
创建provider网络
[root@controller ~]# 
. admin-openrc

[root@controller ~]# 
openstack network create --share --external --provider-physical-network provider --provider-network-type flat provider


创建子网
[root@controller ~]# 
openstack subnet create --network provider --allocation-pool start=192.168.56.150,end=192.168.56.200 --dns-nameserver 8.8.8.8 --gateway 192.168.56.1 --subnet-range 192.168.56.0/24 provider

 

创建Self-service网络
[root@controller ~]# 
openstack network create selfservice

[root@controller ~]# 
openstack subnet create --network selfservice --dns-nameserver 8.8.4.4 --gateway 172.16.1.1 --subnet-range 172.16.1.0/24 selfservice


创建路由
openstack router create router
 

创建子网接口
openstack router add subnet router selfservice
 

创建网关    
openstack router set router --external-gateway provider
 

创建类型
openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
 

创建一个Self-service网络的虚拟机
这里的net-id是
openstack network list
查看到的id
[root@controller ~]# 
openstack server create --flavor m1.nano --image cirros --nic net-id=ee2c3042-6c00-4cc5-9d48-4222a59a7c97 cirros


查看是否创建成功
[root@controller ~]# 
openstack server list




//-------------------------------------------------------------------
八. 安装和配置swift
《一》安装和配置控制器节点

<一>前提条件 
代理服务依赖于身份验证和授权机制，如身份服务。但是，与其他服务不同的是，它还提供了一种内部机制，允许其在没有任何其他OpenStack服务的情况下运行。在配置对象存储服务之前，您必须创建服务凭证和API端点。 
1.来源admin凭据来访问仅管理员CLI命令：

# . admin-openrc

2.要创建身份服务凭据，请完成以下步骤： 
A.创建swift用户：
openstack user create --domain default --password-prompt swift

B.将admin角色添加到swift用户：
openstack role add --project service --user swift admin

C.创建swift服务实体：
openstack service create --name swift \
  --description "OpenStack Object Storage" object-store

3.创建对象存储服务API端点：
openstack endpoint create --region RegionOne \
  object-store public http://controller:8080/v1/AUTH_%\(tenant_id\)s
openstack endpoint create --region RegionOne \
  object-store internal http://controller:8080/v1/AUTH_%\(tenant_id\)s
openstack endpoint create --region RegionOne \
  object-store admin http://controller:8080/v1


<二>安装和配置的部件

1、安装软件包：

yum install -y openstack-swift-proxy python-swiftclient \
  python-keystoneclient python-keystonemiddleware \
  memcached

2、从Object Storage源存储库获取代理服务配置文件：
curl -o /etc/swift/proxy-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/rocky

3、 编辑/etc/swift/proxy-server.conf文件并完成以下操作：

A.在该[DEFAULT]部分中，配置绑定端口，用户和配置目录：

[DEFAULT]
...
bind_port = 8080
user = swift
swift_dir = /etc/swift

B.在[pipeline:main]部分中，
****删除tempurl和 tempauth模块并添加authtoken和keystoneauth 模块：

[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server

pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server


C.在[app:proxy-server]部分中，启用自动帐户创建：

[app:proxy-server]
use = egg:swift#proxy
...
account_autocreate = True


[filter:keystoneauth]
use = egg:swift#keystoneauth
operator_roles = admin,user


D.在[filter:keystoneauth]部分中，配置操作员角色：

[filter:keystoneauth]
use = egg:swift#keystoneauth
...
operator_roles = admin,user




E.在[filter:authtoken]小节中，配置身份服务访问：

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = swift
password = 000000
delay_auth_decision = True


F.在[filter:cache]部分中，配置memcached位置：

[filter:cache]
use = egg:swift#memcache
...
memcache_servers = controller:11211


安装和配置存储节点
（此处的存储节点为计算节点和网络节点，分别在两个节点都运行以下步骤，以下是以其中一个节点为例）

<一>先决条件
在安装和配置存储节点上的对象存储服务之前，必须准备存储设备（添加两块硬盘，大小为5G）

1、 安装支持实用程序包： 
yum install -y xfsprogs rsync

2、 格式/dev/sdb和/dev/sdc设备XFS：

mkfs.xfs /dev/sdb
mkfs.xfs /dev/sdc


3、 创建挂载点目录结构：

mkdir -p /srv/node/sdb
mkdir -p /srv/node/sdc


4、 编辑/etc/fstab文件，添加以下:

/dev/sdb /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
/dev/sdc /srv/node/sdc xfs noatime,nodiratime,nobarrier,logbufs=8 0 2


5、 安装设备：

mount /srv/node/sdb
mount /srv/node/sdc


6、 创建或编辑/etc/rsyncd.conf文件包含以下：

uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address =192.168.56.122
[account]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/account.lock
[container]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/container.lock
[object]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/object.lock



7、开始rsyncd服务并将其配置为在系统启动时启动：

systemctl enable rsyncd.service
systemctl start rsyncd.service    


/usr/bin/rsync --daemon --config=/etc/rsyncd.conf

<二>安装和配置组件
1.安装软件包;

yum install -y openstack-swift-account openstack-swift-container \
  openstack-swift-object

2.从对象存储源存储库获取accounting, container, and object服务配置文件：

curl -o /etc/swift/account-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/rocky
curl -o /etc/swift/container-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/rocky
curl -o /etc/swift/object-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/rocky



3、 编辑/etc/swift/account-server.conf文件并完成以下操作：

A.在[DEFAULT]部分中，配置绑定IP地址、绑定端口、用户、配置目录和挂载点目录：

[DEFAULT]
...
bind_ip = 192.168.56.123
bind_port = 6202
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True


B. 在[pipeline:main]部分，启用适当的模块：

[pipeline:main]
pipeline = healthcheck recon account-server


C. 在[filter:recon]部分，配置 recon (meters)缓存目录：

[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift




4、 编辑/etc/swift/container-server.conf文件并完成以下操作：

A.在[DEFAULT]部分中，配置绑定IP地址、绑定端口、用户、配置目录和挂载点目录：

[DEFAULT]
...
bind_ip = 192.168.56.123
bind_port = 6201
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True



B. 在[pipeline:main]部分，启用适当的模块：

[pipeline:main]
pipeline = healthcheck recon container-server


C. 在[filter:recon]部分，配置 recon (meters)缓存目录：

[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift




5、编辑/etc/swift/object-server.conf文件并完成以下操作：

A.在[DEFAULT]部分中，配置绑定IP地址、绑定端口、用户、配置目录和挂载点目录：

[DEFAULT]
...
bind_ip = 192.168.56.123
bind_port = 6200
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True


B. 在[pipeline:main]部分，启用适当的模块：

[pipeline:main]
pipeline = healthcheck recon object-server


C. 在[filter:recon]部分，配置 recon (meters)缓存目录：

[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock 





6、确保挂载点目录结构的正确所有权：

chown -R swift:swift /srv/node

7、创建recon目录并确保它的正确所有权：

mkdir -p /var/cache/swift
chown -R swift:swift /var/cache/swift



《三》创建并分发初始铃声
在启动对象存储服务之前，您必须创建初始帐户，容器和对象环。环形构建器创建每个节点用来确定和部署存储体系结构的配置文件。

<一>创建账户ring
帐户服务器使用帐户环来维护容器列表。

1.转到/etc/swift目录。

2.创建基本account.builder文件：

swift-ring-builder account.builder create 10 2 1

3.将每个存储节点添加到环中：

swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.56.122 --port 6202 --device sdb --weight 100
swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.56.122 --port 6202 --device sdc --weight 100
swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.56.123 --port 6202 --device sdb --weight 100
swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.56.123 --port 6202 --device sdc --weight 100

4.验证 ring 的内容：
swift-ring-builder account.builder

5.平衡 ring：
swift-ring-builder account.builder rebalance

<二>创建容器ring
帐户服务器使用帐户 ring 来维护一个容器的列表。 
1、切换到 /etc/swift目录。

2、创建基本container.builder文件：

swift-ring-builder container.builder create 10 2 1

3、添加每个节点到 ring 中：

swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.56.122 --port 6201 --device sdb --weight 100
swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.56.122 --port 6201 --device sdc --weight 100


swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.56.123 --port 6201 --device sdb --weight 100
swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.56.123 --port 6201 --device sdc --weight 100



4、验证 ring 的内容：
swift-ring-builder container.builder

5、平衡 ring：
swift-ring-builder container.builder rebalance

<三>创建对象ring
对象服务器使用对象环来维护对象在本地设备上的位置列表。 
1、切换到 /etc/swift目录。

2、创建基本object.builder文件：
swift-ring-builder object.builder create 10 2 1

3、添加每个节点到 ring 中：
swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.56.122 --port 6200 --device sdb --weight 100 
swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.56.122 --port 6200 --device sdc --weight 100


swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.56.123 --port 6200 --device sdb --weight 100
swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.56.123 --port 6200 --device sdc --weight 100



4、验证 ring 的内容：
swift-ring-builder object.builder

5、平衡 ring：
swift-ring-builder object.builder rebalance

分发环配置文件

复制account.ring.gz，container.ring.gz和object.ring.gz 文件到每个存储节点和其他运行了代理服务的额外节点的 /etc/swift 目录。


《四》完成安装
1、/etc/swift/swift.conf从Object Storage源存储库中获取文件：

curl -o /etc/swift/swift.conf \
  https://git.openstack.org/cgit/openstack/swift/plain/etc/swift.conf-sample?h=stable/rocky


2、编辑/etc/swift/swift.conf文件并完成以下操作：

A.在该[swift-hash]部分中，为您的环境配置散列路径前缀和后缀。

[swift-hash]
...
swift_hash_path_suffix = hsystsy.cy
swift_hash_path_prefix = hsystsy.cy




B.在该[storage-policy:0]部分中，配置默认存储策略：

[storage-policy:0]
...
name = Policy-0
default = yes


3、将swift.conf文件复制到/etc/swift每个存储节点上的目录以及任何运行代理服务的其他节点。 
4、在所有节点上，确保配置目录的正确所有权：
chown -R root:swift /etc/swift

5.在控制器节点和运行代理服务的任何其他节点上，启动Object Storage代理服务（包括其依赖关系），并将其配置为在系统引导时启动：

systemctl enable openstack-swift-proxy.service memcached.service
systemctl restart openstack-swift-proxy.service memcached.service


在存储节点上，启动对象存储服务，并配置它们在系统启动时启动：
systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
systemctl start openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
systemctl enable openstack-swift-container.service \
  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
  openstack-swift-container-updater.service
systemctl start openstack-swift-container.service \
  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
  openstack-swift-container-updater.service
systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service
systemctl start openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service 


《五》验证操作
验证对象存储服务的操作。 
如果您使用的是红帽企业版Linux 7或CentOS 7，并且其中一个或多个步骤不起作用，请检查/var/log/audit/audit.log SELinux消息文件，指出拒绝swift进程的操作。如果存在，则将/srv/node目录的安全上下文更改为swift_data_t类型，object_r 角色和system_u用户的最低安全级别（s0）：

chcon -R system_u:object_r:swift_data_t:s0 /srv/node

1、来源demo证书：
. demo-openrc

2、显示服务状态：
swift stat

3、创建container1容器：
openstack container create cont1

4、将测试文件上传到container1容器：
openstack object create cont2 testfile 

5、在container1容器中列出文件：
openstack object list container2

6、从container1容器中下载一个测试文件：
openstack object save container2 testfile




//-------------------------------------------------------------
CentOS7安装OpenStack(Rocky版)-09.安装Cinder存储服务组件（控制节点）
---Cinder install

9.0.Cinder概述
9.1.在控制节点安装cinder存储服务
1）创建cinder数据库
2）在keystone上面注册cinder服务（创建服务证书）
3）安装cinder相关软件包
4）快速修改cinder配置
5）同步cinder数据库
6）修改nova配置文件
7）重启nova-api服务
8）启动cinder存储服务
9.2.在存储节点服务器安装cinder存储服务
1）安装LVM相关软件包
2）启动LVM的metadata服务并配置开机自启动
3）创建LVM逻辑卷
4）配置过滤器，防止系统出错
5）在存储节点安装配置cinder组件
6）在存储节点快速修改cinder配置
7）在存储节点启动cinder服务并配置开机自启动
9.3.在控制节点进行验证
1）获取管理员变量
2）查看存储卷列表
 

Openstack的Cinder存储服务组件，cinder服务可以提供云磁盘（卷），类似阿里云云盘


# openstack-Mitaka-cinder块存储服务中文文档
# https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/cinder.html

# openstack-rocky版本Cinder官方安装文档
# https://docs.openstack.org/cinder/rocky/install/

9.0.Cinder概述
OpenStack块存储服务(cinder)为虚拟机添加持久的存储，块存储提供一个基础设施为了管理卷，以及和OpenStack计算服务交互，为实例提供卷。此服务也会激活管理卷的快照和卷类型的功能。

块存储服务通常包含下列组件：

1）cinder-api
接受API请求，并将其路由到``cinder-volume``执行。
2）cinder-volume
与块存储服务和例如``cinder-scheduler``的进程进行直接交互。它也可以与这些进程通过一个消息队列进行交互。``cinder-volume``服务响应送到块存储服务的读写请求来维持状态。它也可以和多种存储提供者在驱动架构下进行交互。
3）cinder-scheduler守护进程
选择最优存储提供节点来创建卷。其与``nova-scheduler``组件类似。
4）cinder-backup守护进程
``cinder-backup``服务提供任何种类备份卷到一个备份存储提供者。就像``cinder-volume``服务，它与多种存储提供者在驱动架构下进行交互。
5）消息队列
在块存储的进程之间路由信息。

9.1.在控制节点安装cinder存储服务
# Install and configure controller node
https://docs.openstack.org/cinder/rocky/install/cinder-controller-install-rdo.html

1）创建cinder数据库
# 创建相关数据库，授权访问用户

mysql -u root -p000000
----------------------------------------
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'cinder';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'cinder';
flush privileges;
show databases;
select user,host from mysql.user;
exit
----------------------------------------
2）在keystone上面注册cinder服务（创建服务证书）
# 在keystone上创建cinder用户

openstack user create --domain default --password=cinder cinder
openstack user list

# 在keystone上将cinder用户配置为admin角色并添加进service项目，以下命令无输出
openstack role add --project service --user cinder admin


# 创建cinder服务的实体
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
openstack service list

# 创建cinder服务的API端点（endpoint）
openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
openstack endpoint list



3）安装cinder相关软件包
yum install openstack-cinder -y
yum install openstack-utils
4）快速修改cinder配置
openstack-config --set  /etc/cinder/cinder.conf database connection  mysql+pymysql://cinder:cinder@controller/cinder
openstack-config --set  /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:000000@controller
openstack-config --set  /etc/cinder/cinder.conf DEFAULT auth_strategy  keystone 
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken auth_uri  http://controller:5000
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken auth_url  http://controller:5000
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken memcached_servers  controller:11211
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken auth_type  password
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken project_domain_name  default 
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken user_domain_name  default
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken project_name  service 
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken username  cinder
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken password  cinder
openstack-config --set  /etc/cinder/cinder.conf DEFAULT my_ip 192.168.56.122
openstack-config --set  /etc/cinder/cinder.conf oslo_concurrency lock_path  /var/lib/nova/tmp 


5）同步cinder数据库
# 有35张表
su -s /bin/sh -c "cinder-manage db sync" cinder

# 验证数据库
mysql -h192.168.56.122 -ucinder -pcinder -e "use cinder;show tables;"

6）修改nova配置文件
# 配置nova调用cinder服务
openstack-config --set  /etc/nova/nova.conf cinder os_region_name  RegionOne

# 检查生效的nova配置
grep '^[a-z]' /etc/nova/nova.conf |grep os_region_name

7）重启nova-api服务
systemctl restart openstack-nova-api.service
systemctl status openstack-nova-api.service

8）启动cinder存储服务
# 需要启动2个服务
systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl status openstack-cinder-api.service openstack-cinder-scheduler.service

systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl list-unit-files |grep openstack-cinder |grep enabled
# 至此，控制端的cinder服务安装完毕，在dashboard上面可以看到项目目录中多了一个卷服务


# 接下来安装块存储节点服务器storage node
9.2.在存储节点服务器安装cinder存储服务
# 存储节点建议单独部署服务器（最好是物理机），测试时也可以部署在控制节点或者计算节点
# 在本文，存储节点使用LVM逻辑卷提供服务，需要提供一块空的磁盘用以创建LVM逻辑卷
# 我这里在VMware虚拟机增加一块100GB的磁盘

1）安装LVM相关软件包
yum install lvm2 device-mapper-persistent-data -y
2）启动LVM的metadata服务并配置开机自启动
systemctl start lvm2-lvmetad.service
systemctl status lvm2-lvmetad.service

systemctl enable lvm2-lvmetad.service
systemctl list-unit-files |grep lvm2-lvmetad |grep enabled


3）创建LVM逻辑卷
# 检查磁盘状态
fdisk -l
# 创建LVM 物理卷 /dev/sdd
pvcreate /dev/sdd

# 创建 LVM 卷组 cinder-volumes，块存储服务会在这个卷组中创建逻辑卷
vgcreate cinder-volumes /dev/sdd


4）配置过滤器，防止系统出错
# 默认只会有openstack实例访问块存储卷组，不过，底层的操作系统也会管理这些设备并尝试将逻辑卷与系统关联。
# 默认情况下LVM卷扫描工具会扫描整个/dev目录，查找所有包含lvm卷的块存储设备。如果其他项目在某个磁盘设备sda，sdc等上使用了lvm卷，那么扫描工具检测到这些卷时会尝试缓存这些lvm卷，可能导致底层操作系统或者其他服务无法正常调用他们的lvm卷组，从而产生各种问题，需要手动配置LVM，让LVM卷扫描工具只扫描包含"cinder-volume"卷组的设备/dev/sdb，我这边磁盘分区都是格式化的手工分区，目前不存在这个问题，以下是配置演示

vim /etc/lvm/lvm.conf
-----------------------------
devices {
filter = [ "a/sdb/", "r/.*/"]
}
-----------------------------

# 配置规则：
# 每个过滤器组中的元素都以a开头accept接受，或以 r 开头reject拒绝，后面连接设备名称的正则表达式规则。
# 过滤器组必须以"r/.*/"结束，过滤所有保留设备。
# 可以使用命令:vgs -vvvv来测试过滤器。
# 注意：

# 如果存储节点的操作系统磁盘/dev/sda使用的是LVM卷组，也需要将该设备添加到过滤器中，例如：
filter = [ "a/sda/", "a/sdb/", "r/.*/"]
# 如果计算节点的操作系统磁盘/dev/sda使用的是LVM卷组，也需要修改这些节点的/etc/lvm/lvm.conf，在过滤器中增加该类型的磁盘设备，例如：
filter = [ "a/sda/", "r/.*/"]

5）在存储节点安装配置cinder组件
yum install openstack-cinder targetcli python-keystone -y
yum install -y openstack-utils

6）在存储节点快速修改cinder配置
openstack-config --set  /etc/cinder/cinder.conf database connection  mysql+pymysql://cinder:cinder@controller/cinder
openstack-config --set  /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:000000@controller
openstack-config --set  /etc/cinder/cinder.conf DEFAULT auth_strategy  keystone 
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken www_authenticate_uri  http://controller:5000
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken auth_url  http://controller:5000
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken memcached_servers  controller:11211
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken auth_type  password
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken project_domain_name  default 
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken user_domain_name  default
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken project_name  service 
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken username  cinder
openstack-config --set  /etc/cinder/cinder.conf keystone_authtoken password  cinder
openstack-config --set  /etc/cinder/cinder.conf DEFAULT my_ip 192.168.56.123
openstack-config --set  /etc/cinder/cinder.conf lvm volume_driver cinder.volume.drivers.lvm.LVMVolumeDriver
openstack-config --set  /etc/cinder/cinder.conf lvm volume_group cinder-volumes
openstack-config --set  /etc/cinder/cinder.conf lvm iscsi_protocol  iscsi
openstack-config --set  /etc/cinder/cinder.conf lvm iscsi_helper  lioadm
openstack-config --set  /etc/cinder/cinder.conf DEFAULT enabled_backends  lvm
openstack-config --set  /etc/cinder/cinder.conf DEFAULT glance_api_servers  http://controller:9292
openstack-config --set  /etc/cinder/cinder.conf oslo_concurrency lock_path  /var/lib/cinder/tmp
# 如果存储节点是双网卡，选项my_ip需要配置存储节点的管理IP，否则配置本机IP

7）在存储节点启动cinder服务并配置开机自启动
# 需要启动2个服务
systemctl start openstack-cinder-volume.service target.service
systemctl status openstack-cinder-volume.service target.service
systemctl restart openstack-cinder-volume.service target.service

systemctl enable openstack-cinder-volume.service target.service
systemctl list-unit-files |grep openstack-cinder |grep enabled
systemctl list-unit-files |grep target.service |grep enabled
# 至此，在存储节点安装cinder服务就完成了

systemctl restart openstack-cinder-api  openstack-cinder-scheduler  openstack-cinder-volume


9.3.在控制节点进行验证
1）获取管理员变量
2）查看存储卷列表
openstack volume service list

例：
[root@openstack01 tools]# openstack volume service list
+------------------+-------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                          | Zone | Status  | State | Updated At                 |
+------------------+-------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | openstack01.zuiyoujie.com     | nova | enabled | up    | 2018-10-31T10:55:19.000000 |
| cinder-volume    | openstack02.zuiyoujie.com@lvm | nova | enabled | up    | 2018-10-31T10:55:21.000000 |
+------------------+-------------------------------+------+---------+-------+----------------------------+
# 返回以上信息，表示cinder相关节点安装完成



swift --os-auth-url http://192.168.56.122:5000/v1 --auth-version 1 \
      --os-project-name service --os-project-domain-name default\
      --os-username admin --os-user-domain-name default\
      --os-password 000000 list


swift --os-auth-url http://192.168.56.122:5000/v3 --auth-version 3       --os-project-name admin --os-project-domain-name Default      --os-username admin --os-user-domain-name default      --os-password 000000 list


//--------------------------------------------------------------------
Install Heat

创建Heat用户
mysql -u root -p000000

CREATE DATABASE heat;

GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' \
  IDENTIFIED BY '000000';
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' \
  IDENTIFIED BY '000000';


. admin-openrc


openstack user create --domain default --password-prompt heat

openstack role add --project service --user heat admin


创建业务流程服务API终结点：

openstack service create --name heat \
  --description "Orchestration" orchestration

openstack service create --name heat-cfn \
  --description "Orchestration"  cloudformation

openstack endpoint create --region RegionOne \
  orchestration public http://controller:8004/v1/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  orchestration internal http://controller:8004/v1/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  orchestration admin http://controller:8004/v1/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  cloudformation public http://controller:8000/v1

openstack endpoint create --region RegionOne \
  cloudformation internal http://controller:8000/v1

openstack endpoint create --region RegionOne \
  cloudformation admin http://controller:8000/v1


创建包含项目和堆栈用户的热域：
openstack domain create --description "Stack projects and users" heat

创建heat域管理员用户以管理heat域中的项目和用户：
openstack user create --domain heat --password-prompt heat_domain_admin


将admin角色添加到heat域中的heat\u域管理员用户，以启用heat\u域管理员用户的管理堆栈管理权限：
openstack role add --domain heat --user-domain heat --user heat_domain_admin admin


openstack role create heat_stack_owner


将heat_stack_owner角色添加到演示项目和用户，以启用演示用户的堆栈管理：
openstack role add --project service --user admin heat_stack_owner


您必须将heat堆栈所有者角色添加到管理堆栈的每个用户。
openstack role create heat_stack_user


安装和配置组件
yum install -y openstack-heat-api openstack-heat-api-cfn \
  openstack-heat-engine


编辑/etc/heat/heat.conf文件并完成以下操作：
[database]
...
connection = mysql+pymysql://heat:000000@controller/heat

[DEFAULT]
...
transport_url = rabbit://openstack:000000@controller


[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = heat
password = 000000

[trustee]
...
auth_type = password
auth_url = http://controller:35357
username = heat
password = 000000
user_domain_name = default

[clients_keystone]
...
auth_uri = http://controller:5000


[DEFAULT]
...
heat_metadata_server_url = http://controller:8000
heat_waitcondition_server_url = http://controller:8000/v1/waitcondition

[DEFAULT]
...
stack_domain_admin = heat_domain_admin
stack_domain_admin_password = 000000
stack_user_domain_name = heat


填充业务流程数据库：
su -s /bin/sh -c "heat-manage db_sync" heat


启动服务
systemctl enable openstack-heat-api.service \
  openstack-heat-api-cfn.service openstack-heat-engine.service
systemctl start openstack-heat-api.service \
  openstack-heat-api-cfn.service openstack-heat-engine.service


Heat Dashboard installation guide

pip install heat-dashboard

cp <heat-dashboard-dir>/enabled/_[1-9]*.py \
      /usr/share/openstack-dashboard/openstack_dashboard/local/enabled
/usr/lib/python2.7/site-packages/heat_dashboard/

POLICY_FILES['orchestration'] = '<heat-dashboard-dir>/conf/heat_policy.json'

$ cd <heat-dashboard-dir>
$ python ./manage.py compilemessages

$ cd <horizon-dir>
$ DJANGO_SETTINGS_MODULE=openstack_dashboard.settings python manage.py collectstatic --noinput
$ DJANGO_SETTINGS_MODULE=openstack_dashboard.settings python manage.py compress --force


$ sudo service apache2 restart
systemctl restart httpd


//---------------------------------------------------------------
OpenStack系统管理

使用仪表盘进行openstack管理
https://docs.openstack.org/zh_CN/user-guide/dashboard.html

使用命令行对openstack进行管理
https://docs.openstack.org/zh_CN/user-guide/cli.html

使用PySDK进行openstack管理
https://docs.openstack.org/zh_CN/user-guide/sdk.html

命令行速查
https://docs.openstack.org/zh_CN/user-guide/cli-cheat-sheet.html




