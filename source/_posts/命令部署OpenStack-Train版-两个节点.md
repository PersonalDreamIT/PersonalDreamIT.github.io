---
title: 命令部署OpenStack-Train版-两个节点
date: 2023-02-01 23:20:59
updated: 2023-02-02 00:26:30
categories:
  - 云计算
tags:
  - OpenStack
  - 云计算大赛
---

## OpenStack Train版在线命令部署【简化版】

> MariaDB
> RabbitMQ
> Memcached
> Etcd
> keystone
> glance
> nova
> placement
> neutron
> 以上组件部署完成，即完成了最小化部署。
> horizon
> cinder

<table>
<thead>
<tr>
<th>hostname</th>
<th>systemctl</th>
<th>host resource</th>
<th>ip</th>
</tr>
</thead>
<tbody><tr>
<td>controller</td>
<td>centos7.6</td>
<td>4C8G</td>
<td>ens32 172.16.80.11     ens33 NONE</td>
</tr>
<tr>
<td>compute</td>
<td>centos7.6</td>
<td>4C8G</td>
<td>ens32 172.16.80.12     ens33 NONE</td>
</tr>
</tbody></table>

<!-- more -->

### 0 init

> controller compute
> 两个节点分别初始化

```bash
#配置网卡
cat /etc/sysconfig/network-scripts/ifcfg-ens32
cat > /etc/sysconfig/network-scripts/ifcfg-ens33 <<EOF
DEVICE=ens33
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
EOF
systemctl restart network

#配置hosts解析
cat >> /etc/hosts <<EOF
172.16.80.11 controller
172.16.80.12 compute
EOF
cat /etc/hosts

#防火墙以及Seliunx模块
systemctl stop firewalld
systemctl disable firewalld
setenforce 0 
sed -i 's/enforcing/disabled/g' /etc/selinux/config 

#配置yum源
yum install vim net-tools wget -y
mkdir /etc/yum.repos.d/bak
mv /etc/yum.repos.d/*repo /etc/yum.repos.d/bak
yum clean all && yum makecache
yum repolist
cd /etc/yum.repos.d/
#添加yum.repo
yum repolist
```

### 1 time

> controller[server] compute[client]
> 两个节点部署chrony服务
> 使2个节点的时钟服务一致(这是集群服务的必要配置)

```bash
#[controller]
yum install chrony -y
#修改配置文件
vim /etc/chrony.conf
#注释四行；添加一行；开启两行【删除四行，添加3行】
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server time5.aliyun.com iburst
allow 172.16.80.0/24
local stratum 10

[root@controller ~]# egrep -v "^#|^$" /etc/chrony.conf 
server time5.aliyun.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
allow 172.16.80.0/24
local stratum 10
logdir /var/log/chrony

#开启服务并检验
systemctl enable chronyd && systemctl start chronyd 
chronyc sources           
timedatectl 

#[compute]
yum install chrony -y
#修改配置文件
vim /etc/chrony.conf
#注释四行；添加一行
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server controller iburst

[root@compute ~]# egrep -v "^#|^$" /etc/chrony.conf 
server controller iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony

#开启服务并检验
systemctl enable chronyd && systemctl start chronyd
timedatectl
chronyc sources
```

### 2 安装基础软件包

#### 2.1 安装OpenStack软件包

> controller compute

```bash
#安装openstack库
yum install centos-release-openstack-train -y
#更新
yum upgrade -y
#重启
reboot
#安装openstack客户端
yum install python-openstackclient openstack-selinux openstack-utils -y
```

#### 2.2 安装mariadb数据库

> controller

```bash
#安装软件包
yum install mariadb mariadb-server python2-PyMySQL -y
#配置以下内容， bind-address设置为控制节点的管理IP地址，以使其他节点能够通过管理网络进行访问：
#vi /etc/my.cnf.d/openstack.cnf
#[mysqld]
#bind-address = 172.16.80.11     #绑定ip地址，跟随自己实际情况
#default-storage-engine = innodb   #将默认引擎改为innodb，来支持高级数据库功能
#innodb_file_per_table = on   #修改InnoDB为独立表空间模式
#max_connections = 4096    #最大连接数
#collation-server = utf8_general_ci
#character-set-server = utf8

vim /etc/my.cnf.d/openstack.cnf    
[mysqld]
bind-address = 172.16.80.11
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8

#启动数据库服务并设置服务开机启动
systemctl start mariadb.service && systemctl enable mariadb.service 

#运行mysql_secure_installation 脚本初始化数据库服务，并为数据库root帐户设置密码(这里设为123456)
mysql_secure_installation

#mysql_secure_installation  #运行命令
#Enter current password for root (enter for none):       #直接回车
#Set root password? [Y/n] y   #设置密码
#New password:               #设置密码123456
#Re-enter new password:   #再次设置密码
#Remove anonymous users? [Y/n] y   #是否删除匿名用户
#Disallow root login remotely? [Y/n] n  #不允许远程登录
#Remove test database and access to it? [Y/n] y  #删除测试数据库并访问
#Reload privilege tables now? [Y/n] y   #是否重新加载特权表
```

#### 2.3 安装RabbitMQ消息队列

> controller

```bash
#安装软件包
yum install rabbitmq-server -y
#启动消息队列服务并设置服务开机启动
systemctl enable rabbitmq-server && systemctl start rabbitmq-server
#添加openstack 用户，并设置密码，这里设置为123456
rabbitmqctl add_user openstack 123456
#为openstack用户增加配置、读取及写入相关权限
rabbitmqctl set_permissions -p / openstack '.*' '.*' '.*'
#设置角色 赋予管理员角色
rabbitmqctl set_user_tags openstack administrator
#安装web插件（可执行也可不执行）  启动web管理界面
rabbitmq-plugins enable rabbitmq_management
#验证，可访问网址
netstat -nltp | grep 5672 #5672是默认端口，25672是测试工具cli端口
http://172.16.80.11:15672 #用户与密码为guest
```

#### 2.4 安装Memcached缓存数据库

> controller

```bash
#安装软件包
yum install memcached python-memcached -y
#使用控制节点的管理IP地址配置服务。这使其他节点能够通过管理网络进行访问
vim  /etc/sysconfig/memcached
OPTIONS="-l 127.0.0.1,::1,controller" #修改一行
#启动Memcached服务并将其配置为在系统引导时启动
systemctl enable memcached.service && systemctl start memcached.service
```

#### 2.5 安装Etcd服务

> controller

```bash
#安装软件包
yum install etcd -y

#编辑/etc/etcd/etcd.conf文件，以控制节点管理IP地址设置相关选项，以使其他节点通过管理网络进行访问
vim /etc/etcd/etcd.conf            
#:%s/localhost/172.16.80.11/g  
#可以使用:%s/localhost/172.16.80.11/g 实现替换
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://172.16.80.11:2380"    #将localhost改成自己的ip
ETCD_LISTEN_CLIENT_URLS="http://172.16.80.11:2379"   
ETCD_NAME="controller"  
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.80.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.16.80.11:2379"
ETCD_INITIAL_CLUSTER="controller=http://172.16.80.11:2380"   #将default改为上面的ETCD_NAME
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"

#启动etcd服务并设为开机启动：
systemctl enable etcd && systemctl start etcd
```

### 3 安装OpenStack服务

#### 3.1 安装keystone服务

> controller

```bash
#安装和配置keystone
mysql -u root -p123456
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '123456';
flush privileges;
exit

#安装和配置keystone组件
yum install openstack-keystone httpd mod_wsgi -y

#利用命令添加配置
crudini --set /etc/keystone/keystone.conf database connection mysql+pymysql://keystone:123456@controller/keystone
crudini --set /etc/keystone/keystone.conf token provider fernet

#同步认证服务数据库：
su -s /bin/sh -c "keystone-manage db_sync" keystone

#验证是否同步：
mysql -uroot -p123456 -e "use keystone;show tables;"

#初始化密钥存储库 Fernet key库：
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

#引导身份认证服务： 替换ADMIN_PASS为管理用户的合适密码，这里为123456
keystone-manage bootstrap --bootstrap-password 123456 \
--bootstrap-admin-url http://controller:5000/v3/ \
--bootstrap-internal-url http://controller:5000/v3/ \
--bootstrap-public-url http://controller:5000/v3/ \
--bootstrap-region-id RegionOne

#配置apache http服务
#编辑/etc/httpd/conf/httpd.conf文件并配置ServerName选项以引用控制节点：
vim /etc/httpd/conf/httpd.conf
ServerName controller
#创建到/usr/share/keystone/wsgi-keystone.conf文件的链接：
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
#启动Apache HTTP服务并配置开机启动
systemctl enable httpd && systemctl start httpd
#配置administrative 账户
#创建并编辑admin-openrc文件并添加以下内容：
cat > admin-openrc <<EOF
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=123456
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
#创建并编辑demo-openrc文件并添加以下内容：
cat > demo-openrc << EOF
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=123456
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

source admin-openrc

#创建项目和用户
#创建demo域，不创建也行，默认存在default域，admin项目和admin用户和角色 ...看清楚，不需要创建
openstack domain create --description "An Example Domain" demo
#创建service项目，供nova，glance等组件使用
openstack project create --domain default --description "Service Project" service
#创建demo项目
openstack project create --domain default --description "Demo Project" demo
#创建demo用户
openstack user create --domain default --password-prompt demo #密码为123456
#创建user角色
openstack role create user
#将demo用户和角色添加到demo项目
openstack role add --project demo --user demo user

#验证认证服务操作
unset OS_AUTH_URL OS_PASSWORD
#作为管理员用户，请求身份验证令牌
openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue
#作为myuser用户，请求身份验证令牌：
openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name demo --os-username demo token issue

#使用环境脚本
#1.加载admin-openrc文件以使用Identity服务的位置以及管理项目和用户凭据执行环境变量：
#2.请求身份验证令牌：
#验证admin
source admin-openrc
openstack token issue  #获取token
#验证demo
source demo-openrc
openstack token issue
```

#### 3.2 安装Glance服务

> controller

```bash
#创建glance数据库
mysql -u root -p123456
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '123456';
flush privileges;
exit

source admin-openrc
#要创建服务凭据
#创建glance用户
openstack user create --domain default --password 123456 glance
#把admin角色添加到glance用户和项目中
openstack role add --project service --user glance admin
#创建glance服务实体
openstack service create --name glance --description "OpenStack Image" image
#创建镜像服务API端点
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292

#安装和配置组件
yum install openstack-glance -y

#/etc/glance/glance-api.conf
crudini --set /etc/glance/glance-api.conf database connection mysql+pymysql://glance:123456@controller/glance
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_type password
crudini --set /etc/glance/glance-api.conf keystone_authtoken project_domain_name default
crudini --set /etc/glance/glance-api.conf keystone_authtoken user_domain_name default
crudini --set /etc/glance/glance-api.conf keystone_authtoken project_name service
crudini --set /etc/glance/glance-api.conf keystone_authtoken username glance
crudini --set /etc/glance/glance-api.conf keystone_authtoken password 123456
crudini --set /etc/glance/glance-api.conf paste_deploy flavor keystone
crudini --set /etc/glance/glance-api.conf glance_store stores file,http
crudini --set /etc/glance/glance-api.conf glance_store default_store file
crudini --set /etc/glance/glance-api.conf glance_store filesystem_store_datadir /var/lib/glance/images/

#/etc/glance/glance-registry.conf
crudini --set /etc/glance/glance-registry.conf database connection mysql+pymysql://glance:123456@controller/glance
crudini --set /etc/glance/glance-registry.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/glance/glance-registry.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/glance/glance-registry.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/glance/glance-registry.conf keystone_authtoken auth_type password
crudini --set /etc/glance/glance-registry.conf keystone_authtoken project_domain_name default
crudini --set /etc/glance/glance-registry.conf keystone_authtoken user_domain_name default
crudini --set /etc/glance/glance-registry.conf keystone_authtoken project_name service
crudini --set /etc/glance/glance-registry.conf keystone_authtoken username glance
crudini --set /etc/glance/glance-registry.conf keystone_authtoken password 123456
crudini --set /etc/glance/glance-registry.conf paste_deploy flavor keystone

#同步镜像服务数据库
 su -s /bin/sh -c "glance-manage db_sync" glance

#完成安装,启动镜像服务并设为开机启动：
systemctl enable openstack-glance-api.service openstack-glance-registry.service && systemctl restart openstack-glance-api.service openstack-glance-registry.service

#验证操作
egrep -v "^#|^$" /etc/glance/glance-api.conf
egrep -v "^#|^$" /etc/glance/glance-registry.conf

#下载镜像
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
#将镜像上传到image服务，指定磁盘格式为QCOW2，指定裸容器格式和公开可见性，以便所有项目都可以访问它：
source admin-openrc
openstack image create "cirros" \
--file cirros-0.4.0-x86_64-disk.img \
--disk-format qcow2 \
--container-format bare \
--public
#或
source admin-openrc
glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility public
#查看上传的镜像，镜像状态应为active状态
openstack image list 
```

#### 3.3 安装nova placement服务

> controller

```bash
##安装和配置控制节点
mysql -u root -p123456
#创建nova_api, nova, nova_cell0数据库
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '123456';

CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY '123456';
flush privileges;
exit

source admin-openrc
#创建计算服务凭证
#创建nova用户
openstack user create --domain default --password 123456 nova
#为nova用户添加admin角色
openstack role add --project service --user nova admin
#创建nova服务端点
openstack service create --name nova --description "OpenStack Compute" compute
#创建compute API 服务端点
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1

#创建一个placement用户
openstack user create --domain default --password 123456 placement 
#将placement用户添加到admin角色和service项目中
openstack role add --project service --user placement admin 
openstack service create --name placement --description "Placement API" placement #创建API条目
openstack endpoint create --region RegionOne placement public http://controller:8778  #创建API端点
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778

##安装和配置组件
yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-placement-api -y

#/etc/nova/nova.conf
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:123456@controller
crudini --set /etc/nova/nova.conf DEFAULT my_ip 172.16.80.11
crudini --set /etc/nova/nova.conf DEFAULT use_neutron True
crudini --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
crudini --set /etc/nova/nova.conf api_database connection mysql+pymysql://nova:123456@controller/nova_api
crudini --set /etc/nova/nova.conf database connection mysql+pymysql://nova:123456@controller/nova
crudini --set /etc/nova/nova.conf api auth_strategy keystone
crudini --set /etc/nova/nova.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password 123456
crudini --set /etc/nova/nova.conf vnc enabled true
crudini --set /etc/nova/nova.conf vnc server_listen 172.16.80.11
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address 172.16.80.11
crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
crudini --set /etc/nova/nova.conf placement region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name default
crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password 123456

#vim /etc/placement/placement.conf
crudini --set /etc/placement/placement.conf placement_database connection mysql+pymysql://placement:123456@controller/placement
crudini --set /etc/placement/placement.conf api auth_strategy keystone
crudini --set /etc/placement/placement.conf keystone_authtoken auth_url http://controller:5000/v3
crudini --set /etc/placement/placement.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/placement/placement.conf keystone_authtoken auth_type password
crudini --set /etc/placement/placement.conf keystone_authtoken project_domain_name default
crudini --set /etc/placement/placement.conf keystone_authtoken user_domain_name default
crudini --set /etc/placement/placement.conf keystone_authtoken project_name service
crudini --set /etc/placement/placement.conf keystone_authtoken username placement
crudini --set /etc/placement/placement.conf keystone_authtoken password 123456

#因为软件包的bug，要修改配置文件添加以下内容
vim /etc/httpd/conf.d/00-placement-api.conf
#在ErrorLog /var/log/placement/placement-api.log后添加如下
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>

#重启httpd服务
systemctl restart httpd

su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova #注册cell0数据库
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova

su -s /bin/sh -c "placement-manage db sync" placement #写入数据库
#/usr/lib/python2.7/site-packages/pymysql/cursors.py:170: Warning: (1280, u"Name 'alembic_version_pkc' ignored for PRIMARY key.")
  result = self._query(query)
#以上为正常现象

#验证是否注册成功
nova-manage cell_v2 list_cells

#启动nova服务并设置开机自启
systemctl enable openstack-nova-api.service  openstack-nova-console.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service openstack-nova-console.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

> compute     易错警告⚠

```bash
#安装和配置计算节点
#安装和配置组件
yum install openstack-nova-compute crudini -y

#/etc/nova/nova.conf
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:123456@controller
crudini --set /etc/nova/nova.conf DEFAULT my_ip 172.16.80.12
crudini --set /etc/nova/nova.conf DEFAULT use_neutron True
crudini --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
crudini --set /etc/nova/nova.conf api auth_strategy keystone
crudini --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password 123456
crudini --set /etc/nova/nova.conf vnc enabled True
crudini --set /etc/nova/nova.conf vnc server_listen 0.0.0.0
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address 172.16.80.12
crudini --set /etc/nova/nova.conf vnc novncproxy_base_url http://controller:6080/vnc_auto.html
crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
crudini --set /etc/nova/nova.conf placement os_region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name Default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name default
crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password 123456

#[真实服务器]
#egrep -c '(vmx|svm)' /proc/cpuinfo //如果此命令返回值为one or greater，您的计算节点支持硬件加速，这通常不需要额外的配置。如果此命令返回值为zero，您的计算节点不支持硬件加速，您必须配置libvirt使用QEMU而不是KVM）
vim /etc/nova/nova.conf
[libvirt]
virt_type = qemu
#或
#判断计算节点是否支持虚拟机的硬件加速
virt_num=`egrep -c '(vmx|svm)' /proc/cpuinfo`
if [ $virt_num = '0' ];then
	crudini --set /etc/nova/nova.conf libvirt virt_type  qemu
fi

#[vm]
crudini --set /etc/nova/nova.conf libvirt virt_type  qemu

#启动计算服务（包括其相关性），并将其配置为在系统引导时自动启动：
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl restart libvirtd.service openstack-nova-compute.service
```

> controller

```bash
#添加compute节点到cell数据库
source admin-openrc
openstack compute service list --service nova-compute

#发现计算节点
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

#验证计算服务操作
#1.列出服务组件以验证每个进程成功启动和注册：
source admin-openrc
openstack compute service list #此输出应显示在控制节点上启用三个服务组件，并在计算节点上启用一个服务组件。
#2.列出身份服务中的API端点以验证与身份服务的连接：
openstack catalog list
#3.列出Image服务中的镜像以验证与Image服务的连通性：
openstack image list
#4.检查cells和placement API是否正常运行
nova-status upgrade check
```

#### 3.4 安装neutron服务

> controller

```bash
#创建数据库
mysql -u root -p123456
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '123456';
flush privileges;
exit

source admin-openrc
#创建服务凭证，完成以下操作
#创建neutron用户
openstack user create --domain default --password 123456 neutron
#添加admin角色到neutron用户
openstack role add --project service --user neutron admin
#创建neutron服务实体
openstack service create --name neutron --description "OpenStack Networking" network
#创建网络服务API端点
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696

#配置网络部分
#安装组件
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y

#配置服务组件
#/etc/neutron/neutron.conf
crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins router
crudini --set /etc/neutron/neutron.conf DEFAULT allow_overlapping_ips true
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:123456@controller
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes true
crudini --set /etc/neutron/neutron.conf database connection mysql+pymysql://neutron:123456@controller/neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password 123456
crudini --set /etc/neutron/neutron.conf nova auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf nova auth_type password
crudini --set /etc/neutron/neutron.conf nova project_domain_name default
crudini --set /etc/neutron/neutron.conf nova user_domain_name default
crudini --set /etc/neutron/neutron.conf nova region_name RegionOne
crudini --set /etc/neutron/neutron.conf nova project_name service
crudini --set /etc/neutron/neutron.conf nova username nova
crudini --set /etc/neutron/neutron.conf nova password 123456
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp

#配置网络二层插件
#/etc/neutron/plugins/ml2/ml2_conf.ini
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers flat,vlan,vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers linuxbridge,l2population
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks provider
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset true

#配置linux网桥代理 
#/etc/neutron/plugins/ml2/linuxbridge_agent.ini
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings provider:ens33
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 172.16.80.11
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

#通过验证下列所有SysTL值设置为1以确保Linux操作系统内核支持网桥过滤器：
vim /usr/lib/sysctl.d/00-system.conf 
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
sysctl -p

#配置三层代理
#Layer-3（L3）代理为自助虚拟网络提供路由和NAT服务。
#/etc/neutron/l3_agent.ini
crudini --set /etc/neutron/l3_agent.ini DEFAULT interface_driver linuxbridge

#配置DHCP代理
#/etc/neutron/dhcp_agent.ini
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver linuxbridge
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata true

#配置metadata
#/etc/neutron/metadata_agent.ini
crudini --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host controller
crudini --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret 123456

#配置计算服务使用网络服务
#/etc/nova/nova.conf 
crudini --set /etc/nova/nova.conf  neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf  neutron auth_type password
crudini --set /etc/nova/nova.conf  neutron project_domain_name default
crudini --set /etc/nova/nova.conf  neutron user_domain_name default
crudini --set /etc/nova/nova.conf  neutron region_name RegionOne
crudini --set /etc/nova/nova.conf  neutron project_name service
crudini --set /etc/nova/nova.conf  neutron username neutron
crudini --set /etc/nova/nova.conf  neutron password 123456
crudini --set /etc/nova/nova.conf  neutron service_metadata_proxy true
crudini --set /etc/nova/nova.conf  neutron metadata_proxy_shared_secret 123456

#完成安装启动服务
#网络服务初始化脚本需要一个指向ML2插件配置文件/etc/neutron/plugins/ml2/ml2_conf.ini的符号链接/etc/neutron/plugin.ini。 如果此符号链接不存在，请使用以下命令创建它：
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
#重启compute API服务
systemctl restart openstack-nova-api
#同步数据库
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf   --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

#启动网络服务并设为开机启动
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
systemctl status neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
#对于联网选项2，还启用并启动第三层服务：
systemctl enable neutron-l3-agent.service && systemctl start neutron-l3-agent.service
```

> 安装和配置compute节点  易错警告⚠

```bash
#计算节点处理实例的连接和安全组。
#安装和配置组件

#安装组件
yum install openstack-neutron-linuxbridge ebtables ipset -y

#配置公共组件
#/etc/neutron/neutron.conf 
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:123456@controller
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password 123456 
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp

#配置网络部分
#配置Linux网桥
#/etc/neutron/plugins/ml2/linuxbridge_agent.ini
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 172.16.80.12
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings provider:ens33

vim /etc/security/limits.conf
*               soft    nofile            1024000
*               hard    nofile            1024000

#配置计算服务使用网络服务
#/etc/nova/nova.conf
crudini --set /etc/nova/nova.conf neutron url http://controller:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name default
crudini --set /etc/nova/nova.conf neutron user_domain_name default
crudini --set /etc/nova/nova.conf neutron region_name RegionOne
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password 123456

#确保您的Linux操作系统内核支持网桥过滤器，方法是验证以下所有sysctl值均设置为1：
vim /usr/lib/sysctl.d/00-system.conf 
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
sysctl -p

#完成安装启动服务
#重启compute服务
systemctl restart openstack-nova-compute.service
#设置网桥服务开机启动
systemctl enable neutron-linuxbridge-agent.service && systemctl start neutron-linuxbridge-agent.service

#验证
source admin-openrc
openstack extension list --network
openstack network agent list
```

#### 3.5 安装Horizon服务

> controller

```bash
#安装dashboard服务
yum install openstack-dashboard -y 

#修改配置文件
vim /etc/openstack-dashboard/local_settings

OPENSTACK_HOST = "controller"        ##配置界面在控制节点使用

ALLOWED_HOSTS = ['*']               ##允许所有主机访问

##配置memcached存储服务
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': 'controller:11211',
    },
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST  ##启动v3的认证api

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True       ##启用domain支持

##配置api版本
OPENSTACK_API_VERSIONS = {              
    "data-processing": 1.1,
    "identity": 3,
    "image": 2,
    "volume": 2,
    "compute": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'       ##配置Default为默认域

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"                ##配置user角色为默认角色

TIME_ZONE = "Asia/Shanghai"                         ##配置时区

#修改配置文件
vim /etc/httpd/conf.d/openstack-dashboard.conf
WSGIApplicationGroup %{GLOBAL} #第四行添加

#重启httpd服务和memcached服务
systemctl restart httpd.service memcached.service

#浏览器输入ip
http://172.16.80.11/dashboard
=============================================================================
#搭建openstack The requested URL /auth/login/ was not found on this server报错
vim /usr/share/openstack-dashboard/openstack_dashboard/local/local_settings.py
WEBROOT = '/dashboard'
==============================================================================
```

> 扩展 自己制作离线yum源

```bash
###############控制节点################
yum -y install createrepo
mkdir -p /mnt/openstack/openstack_Train
cd /var/cache/yum/x86_64/7/
find ./* -name "*.rpm"  -exec cp {} /mnt/openstack/openstack_Train/ \; 
###############计算节点################
mkdir -p /mnt/openstack/openstack_Train_compute
cd /var/cache/yum/x86_64/7/
find ./* -name "*.rpm"  -exec cp {} /mnt/openstack/openstack_Train_compute/ \;
scp /mnt/openstack/openstack_Train_compute/* 192.168.200.10:/mnt/openstack/openstack_Train_compute

#到控制节点上把所有的rpm包都放到一起
cd /mnt/
mv -f openstack_Train_compute/*.rpm openstack_Train/

#创建yum
cd /mnt/openstack/openstack_Train
createrepo ./
ls repodata/

#打包
tar -zcvf openstack-train.tar.gz openstack_Train/

#这样就可以将tar包保存到本地，供以后使用
1. 将tar包上传，解压到/mnt/
2. 配置yum.repo文件，配置yum到本地目录
3. 就可以使用不需要联网下载
```

#### 3.6 安装Cinder服务

> controller

```bash
#安装和配置controller节点
#本节介绍如何在控制节点上安装和配置代码为cinder的块存储服务。 此服务至少需要一个为实例提供卷的额外存储节点。
#创建cinder数据库
#在安装和配置块存储服务之前，您必须创建数据库，服务凭据和API端点。
mysql -u root -p123456
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY '123456';
flush privileges;
exit

source admin-openrc
#要创建服务凭据
#创建一个cinder用户：
openstack user create --domain default --password 123456 cinder
#添加admin角色到cinder用户：
openstack role add --project service --user cinder admin
#创建cinderv2和cinderv3服务实体：
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
#创建块存储服务API端点：
openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s

#安装和配置组件
yum install openstack-cinder -y
#/etc/cinder/cinder.conf
crudini --set /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:123456@controller
crudini --set /etc/cinder/cinder.conf DEFAULT auth_strategy keystone
crudini --set /etc/cinder/cinder.conf DEFAULT my_ip 172.16.80.11
crudini --set /etc/cinder/cinder.conf database connection mysql+pymysql://cinder:123456@controller/cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_type password
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_domain_id default
crudini --set /etc/cinder/cinder.conf keystone_authtoken user_domain_id default
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_name service
crudini --set /etc/cinder/cinder.conf keystone_authtoken username cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken password 123456
crudini --set /etc/cinder/cinder.conf oslo_concurrency lock_path /var/lib/cinder/tmp

#同步块存储数据库：
su -s /bin/sh -c "cinder-manage db sync" cinder

#配置计算服务使用块存储
#编辑/etc/nova/nova.conf文件并添加以下内容：
#/etc/nova/nova.conf
crudini --set /etc/nova/nova.conf os_region_name RegionOne

#完成安装启动服务
#重新启动Compute API服务：
systemctl restart openstack-nova-api.service
#启动块存储服务并将其配置为在系统引导时启动：
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl restart openstack-cinder-api.service openstack-cinder-scheduler.service
```

> compute

```bash
#安装和配置cinder节点
#安装配置LVM
#安装支持的软件包 安装LVM软件包
yum install lvm2 device-mapper-persistent-data -y
#启动LVM元数据服务并将其配置为在系统引导时启动：
systemctl enable lvm2-lvmetad.service && systemctl start lvm2-lvmetad.service

#创建LVM物理逻辑卷/dev/sdb
pvcreate /dev/sdb

#创建cinder-volumes逻辑卷组
vgcreate cinder-volumes /dev/sdb

#只有实例才能访问块存储卷。 但是，底层操作系统管理与卷关联的设备。 默认情况下，LVM卷扫描工具会扫描包含卷的块存储设备的/ dev目录。 如果项目在其卷上使用LVM，则扫描工具将检测这些卷并尝试缓存它们，这可能会导致底层操作系统和项目卷出现各种问题。 您必须重新配置LVM以仅扫描包含cinder-volumes卷组的设备。 编辑/etc/lvm/lvm.conf文件并完成以下操作：
#在devices部分中，添加一个接受/ dev / sdb设备的过滤器并拒绝所有其他设备：
vim /etc/lvm/lvm.conf
devices {
filter = [ "a/sdb/", "r/.*/"]

#如果您的存储节点在操作系统磁盘上使用LVM，则还必须将关联的设备添加到过滤器。 例如，如果/ dev / sda设备包含操作系统：
filter = [ "a/sda/", "a/sdb/", "r/.*/"]

#如果您的计算节点在操作系统磁盘上使用LVM，则还必须修改这些节点上/etc/lvm/lvm.conf文件中的筛选器以仅包含操作系统磁盘。 例如，如果/ dev / sda设备包含操作系统：
filter = [ "a/sda/", "r/.*/"]

#安装和配置组件
#安装软件包
yum install openstack-cinder targetcli python-keystone -y
#/etc/cinder/cinder.conf
crudini --set /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:123456@controller
crudini --set /etc/cinder/cinder.conf DEFAULT auth_strategy keystone
crudini --set /etc/cinder/cinder.conf DEFAULT my_ip 172.16.80.12
crudini --set /etc/cinder/cinder.conf DEFAULT enabled_backends  lvm
crudini --set /etc/cinder/cinder.conf DEFAULT glance_api_servers http://controller:9292
crudini --set /etc/cinder/cinder.conf database connection mysql+pymysql://cinder:123456@controller/cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_type password
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_domain_id default
crudini --set /etc/cinder/cinder.conf keystone_authtoken user_domain_id default
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_name service
crudini --set /etc/cinder/cinder.conf keystone_authtoken username cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken password 123456
crudini --set /etc/cinder/cinder.conf oslo_concurrency lock_path /var/lib/cinder/tmp
#自动添加
crudini --set /etc/cinder/cinder.conf lvm volume_driver cinder.volume.drivers.lvm.LVMVolumeDriver 
crudini --set /etc/cinder/cinder.conf lvm volume_group cinder-volumes 
crudini --set /etc/cinder/cinder.conf lvm iscsi_protocol iscsi 
crudini --set /etc/cinder/cinder.conf lvm iscsi_helper lioadm

#完成安装启动服务
#设置存储服务开机启动
systemctl enable openstack-cinder-volume.service target.service 
systemctl restart openstack-cinder-volume.service target.service
```

> controller

```bash
#验证cinder配置
#验证Cinder操作,在控制器节点上执行这些命令。
. admin-openrc
#列出服务组件以验证每个进程的成功启动：
openstack volume service list
```
