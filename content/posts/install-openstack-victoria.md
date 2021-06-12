---
title: "openstack victoria版安装"
categories: [ "openstack" ]
tags: [ "openstack" ]
draft: false
slug: "install-openstack-victoria"
date: "2021-06-12T00:00:00+08:00"
---

近期公司业务需求，需要安装一套Openstack环境学习，看了一下现在已经出了`wallaby`版了，我果断选择了上一个版本`victoria`。因为没有足够多的物理服务器了，只好找了一台64核256G内存6T硬盘的机器来创建几台虚拟机来搭环境了。

# 实验环境

此次实验使用到了三台虚拟机，都是使用centos8系统，一台机器当作控制和网络节点，另外两台当作计算节点，使用OVS+VLAN的网络模式，eth0作为管理网络，eth1互相连接到OVS网桥上模拟trunk网卡，controller多增加一个eth2用于访问外部网络。

| 节点        | 作用               | eth0          | eth1 | eth2       |
| ----------- | ------------------ | ------------- | ---- | ---------- |
| controller  | 控制节点、网络节点 | 172.16.10.100 | 无IP | 桥接，无IP |
| compute-101 | 计算节点           | 172.16.10.101 | 无IP | ❌          |
| compute-102 | 计算节点           | 172.16.10.102 | 无IP | ❌          |



# 安装虚拟机

## 安装依赖

安装KVM和Linux网桥

```bash
yum install -y qemu-kvm libvirt virt-install bridge-utils virt-manager dejavu-lgc-sans-fonts
```

> `dejavu-lgc-sans-fonts`用于解决 `virt-manaer` 乱码


启动

```bash
systemctl enable libvirtd && systemctl start libvirtd
```

安装OVS

```bash
yum install openvswitch
```

启动OVS

```bash
systemctl enable openvswitch && systemctl start openvswitch
```

### 创建虚拟机

使用 virt-manager 创建三台虚拟机

![image-20210604140054913](https://oss.typesafe.cn/vm-nodes.png)

### 配置网络

#### 配置管理网卡

给虚拟机配置桥接网络，参考[Linux虚拟化技术KVM](https://typesafe.cn/posts/linux-kvm/)，效果如图

![](https://oss.typesafe.cn/vm-manage-port-config.png)

#### 配置trunk网卡

使用ovs创建一个虚拟网桥。

```bash
ovs-vsctl add-br br-vlan
```

此时网桥`br-vlan`上是没有任何虚拟网卡的，然后关闭虚拟机，在`virt-manager`上添加一个网络设备

![](https://oss.typesafe.cn/kvm-ovs-trunk-port-config.png)

使用命令找到虚拟机并编辑虚拟机XML文件。

```bash
virsh list --all
virsh edit controller
```

找到对应的 `interface` 元素，在 `source` 和 `model`中间添加一行。

```bash
<virtualport type='openvswitch' />
```

然后保存退出，启动虚拟机，启动成功之后会 `virtualport` 中间生成一个新的元素。

![](https://oss.typesafe.cn/kvm-ovs-trunk-port-xml.png)

确认是否添加成功。

```bash
ovs-vsctl show
```

成功的话可以在网桥上看到自动生成的几个虚拟网卡。
```bash
Bridge br-vlan
    Port br-vlan
        Interface br-vlan
            type: internal
    Port "vnet1"
        Interface "vnet1"
    Port "vnet2"
        Interface "vnet2"
    Port "vnet3"
        Interface "vnet3"
ovs_version: "2.9.0"
```

以上步骤重复几次以完成其他虚拟机的配置。

# 安装Openstack

## 配置环境

全部节点都需要操作。

#### 切换网络服务为 `network-scripts`

```bash
# 安装Network服务
yum install network-scripts -y
# 停用NetworkManager并禁止开机启动
systemctl stop NetworkManager && systemctl disable NetworkManager
# 启用Network并设置开机启动
systemctl enable network && systemctl start network
```

#### 设置静态IP

编辑网卡配置文件

```bash
vim /etc/sysconfig/network-scripts/ifcfg-eth0
```

修改并添加以下内容

```bash
BOOTPROTO=static
ONBOOT=yes
IPADDR=172.16.10.100
NETMASK=255.255.255.0
GATEWAY=172.16.0.1
```

重启网络

```bash
systemctl restart network
```

#### 修改主机名称

```bash
# 修改控制节点
hostnamectl set-hostname controller
# 修改计算节点compute-101
hostnamectl set-hostname compute-101
# 修改计算节点compute-102
hostnamectl set-hostname compute-102
```

### 修改hosts文件

添加以下内容

```bash
172.16.10.100 controller
172.16.10.101 compute-101
172.16.10.102 compute-102
```

### 关闭防火墙

```bash
systemctl stop firewalld && systemctl disable firewalld
```

## 安装基础服务

全部节点都需要操作。

### 升级软件包

```bash
yum upgrade -y
```

### 安装时间同步服务`chronyd `

```bash
yum install chrony -y
```

计算节点修改配置文件 `/etc/chrony.conf` 中的 `pool 2.centos.pool.ntp.org iburst` 为 `server controller iburst` 直接与控制节点同步时间。

重启chrony服务并开机自启

```bash
systemctl restart chronyd && systemctl enable chronyd
```

### 安装openstack存储库

```bash
yum config-manager --enable powertools
yum install centos-release-openstack-victoria -y
```

### 安装openstack客户端和openstack-selinux

```bash
yum install python3-openstackclient openstack-selinux -y
```

### 禁用selinux

```bash
cat>/etc/selinux/config<<EOF
SELINUX=permissive
SELINUXTYPE=targeted
setenforce 0
EOF
```

### 修改用户权限

```bash
echo "neutron ALL = (root) NOPASSWD: ALL" > /etc/sudoers.d/neutron
echo "nova ALL = (root) NOPASSWD: ALL" > /etc/sudoers.d/nova
```

# 控制节点

## 安装数据库

1. 安装Mariadb数据库，也可安装MySQL数据库。
    ```bash
    yum install mariadb mariadb-server python3-PyMySQL -y
    ```

1. 创建和编辑`vim /etc/my.cnf.d/openstack.cnf`文件，添加如下信息

   ```shell
   [mysqld]
   bind-address = 0.0.0.0
   default-storage-engine = innodb
   innodb_file_per_table = on
   max_connections = 4096
   collation-server = utf8_general_ci
   character-set-server = utf8
   ```

2. 启动数据库并设置为开机自启

   ```shell
   systemctl start mariadb && systemctl enable mariadb
   ```

3. 配置数据库

   ```shell
   mysql_secure_installation
   
   # 输入当前用户root密码，若为空直接回车
   Enter current password for root (enter for none):
   OK, successfully used password, moving on...
   # 是否设置root密码
   Set root password? [Y/n] y
   # 输入新密码
   New password:
   # 再次输入新密码
   Re-enter new password:
   # 是否删除匿名用户
   Remove anonymous users? [Y/n] y
   # 是否禁用远程登录
   Disallow root login remotely? [Y/n] n
   # 是否删除测试数据库
   Remove test database and access to it? [Y/n] y
   # 是否重新加载权限表
   Reload privilege tables now? [Y/n] y
   
   # 以上步骤根据实际情况做配置即可，不一定要与此处保持一致
   ```

## 安装消息队列

1. 安装软件包

   ```shell
   yum install rabbitmq-server -y
   ```

2. 启动消息队列服务并设置为开机自启

   ```shell
   systemctl start rabbitmq-server && systemctl enable rabbitmq-server
   ```

3. 添加openstack用户并设置密码

   ```shell
   rabbitmqctl add_user openstack RABBIT_PASS
   ```

4. 给openstack用户可读可写可配置权限

   ```shell
   rabbitmqctl set_permissions openstack ".*" ".*" ".*"
   ```

5. 为了方便监控，启用Web界面管理插件

   ```shell
   rabbitmq-plugins enable rabbitmq_management
   ```

6. 浏览器访问 IP:15672，使用 guest/guest 登录rabbitmq

## 安装Memcached缓存

1. 安装软件包

   ```shell
   yum install memcached python3-memcached -y
   ```

2. 编辑`vim /etc/sysconfig/memcached`文件，将OPTTONS行修改成如下信息

   ```shell
   OPTIONS="-l 127.0.0.1,::1,controller"
   ```

3. 启动Memcached服务并设置开机自启

   ```shell
   systemctl start memcached && systemctl enable memcached
   ```

## 安装KeyStone服务

### 创建数据库

1. 连接数据库

   ```mysql
   mysql -u root -p
   ```

2. 创建keystone数据库

   ```mysql
   CREATE DATABASE keystone;
   ```

3. 授予keystone数据库权限，然后退出

   ```mysql
   GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
   GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
   exit;
   ```

### 安装软件包

1. 安装软件

   ```bash
   yum install openstack-keystone httpd python3-mod_wsgi -y
   ```

2. 修改配置文件

   ```bash
   cat > /etc/keystone/keystone.conf <<EOF
   [DEFAULT]
   [database]
   connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
   [token]
   provider = fernet
   EOF
   ```

3. 初始化数据库

   ```bash
   su -s /bin/sh -c "keystone-manage db_sync" keystone
   ```

4. 初始化Fernet

   ```bash
   keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
   keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
   ```

5. 引导身份认证服务

   ```bash
   keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
     --bootstrap-admin-url http://controller:5000/v3/ \
     --bootstrap-internal-url http://controller:5000/v3/ \
     --bootstrap-public-url http://controller:5000/v3/ \
     --bootstrap-region-id RegionOne
   ```

## 配置Apache HTTP服务

1. 修改`vim /etc/httpd/conf/httpd.conf`文件，添加如下信息

   ```bash
   ServerName controller
   ```

2. 创建软链接

   ```bash
   ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
   ```

3. 启动httpd服务 并设置开机自启

   ```bash
   systemctl start httpd && systemctl enable httpd
   ```

4. 创建环境变量脚本

   ```bash
   cat > admin-openrc <<EOF
   export OS_USERNAME=admin
   export OS_PASSWORD=ADMIN_PASS
   export OS_PROJECT_NAME=admin
   export OS_USER_DOMAIN_NAME=Default
   export OS_PROJECT_DOMAIN_NAME=Default
   export OS_AUTH_URL=http://controller:5000/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2
   EOF
   ```

   执行命令 `source admin-openrc` 或者 `. admin-openrc` 使环境变量生效。

## 创建域、项目、用户和角色

1. 创建域，程序中已存在默认域，此命令只是一个创建域的例子，可以不执行

   ```shell
   openstack domain create --description "An Example Domain" example
   ```

2. 创建service项目，也叫做租户

   ```shell
   openstack project create --domain default --description "Service Project" service
   ```

3. 验证token令牌

   ```bash
   openstack token issue
   ```

## 安装Glance服务

### 创建数据库

1. 连接数据库

   ```mysql
   mysql -u root -p
   ```

2. 创建glance数据库

   ```mysql
   CREATE DATABASE glance;
   ```

3. 授予glance数据库权限，然后退出

   ```mysql
   GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
   GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
   exit;
   ```

### 创建glance用户并关联角色

1. 创建glance用户并设置密码为GLANCE_PASS，此处与上面创建用户的不同之处是未使用交互式的方式，直接将密码放入了命令中

   ```shell
   openstack user create --domain default --password GLANCE_PASS glance
   ```

2. 使用admin角色将Glance用户添加到服务项目中

   ```shell
   # 在service的项目上给glance用户关联admin角色
   openstack role add --project service --user glance admin
   ```

### 创建glance服务并注册API

1. 创建glance服务

   ```shell
   openstack service create --name glance --description "OpenStack Image" image
   ```

2. 注册API，也就是创建镜像服务的API终端endpoints

   ```shell
   openstack endpoint create --region RegionOne image public http://controller:9292
   openstack endpoint create --region RegionOne image internal http://controller:9292
   openstack endpoint create --region RegionOne image admin http://controller:9292
   ```

### 安装并配置glance

1. 安装软件包

   ```shell
   dnf install openstack-glance -y
   ```

2. 修改配置文件

   ```shell
   cat > /etc/glance/glance-api.conf<<EOF
   [database]
   connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
   
   [keystone_authtoken]
   www_authenticate_uri  = http://controller:5000
   auth_url = http://controller:5000
   memcached_servers = controller:11211
   auth_type = password
   project_domain_name = Default
   user_domain_name = Default
   project_name = service
   username = glance
   password = GLANCE_PASS
   
   [paste_deploy]
   flavor = keystone
   
   [glance_store]
   stores = file,http
   default_store = file
   filesystem_store_datadir = /var/lib/glance/images/
   EOF
   ```

3. 同步数据库

   ```bash
   su -s /bin/sh -c "glance-manage db_sync" glance
   ```

4. 启动glance服务并设置开机自启

   ```shell
   systemctl start openstack-glance-api && systemctl enable openstack-glance-api
   ```

## 安装Placement服务

### 创建数据库

1. 连接数据库

   ```mysql
   mysql -u root -p
   ```

2. 创建Plancement数据库

   ```mysql
   CREATE DATABASE placement;
   ```

3. 授予Plancement数据库权限，然后退出

   ```mysql
   GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';
   GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';
   exit;
   ```

### 配置用户和Endpoint

1. 创建一个plancement用户并设置密码为PLACEMENT_PASS

   ```shell
   openstack user create --domain default --password PLACEMENT_PASS placement
   ```

2. 使用admin角色将Placement用户添加到服务项目中

   ```shell
   # 在service的项目上给placement用户关联admin角色
   openstack role add --project service --user placement admin
   ```

### 创建Placement服务并注册API

1. 创建Plancement服务

   ```shell
   openstack service create --name placement --description "Placement API" placement
   ```

2. 创建Plancement服务API端口

   ```shell
   openstack endpoint create --region RegionOne placement public http://controller:8778
   openstack endpoint create --region RegionOne placement internal http://controller:8778
   openstack endpoint create --region RegionOne placement admin http://controller:8778
   ```

### 安装Placement服务

1. 安装Plancement软件包

   ```shell
   yum install openstack-placement-api -y
   ```

2. 修改配置文件

   ```bash
   cat > /etc/placement/placement.conf <<EOF
   [placement_database]
   connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
   
   [api]
   auth_strategy = keystone
   
   [keystone_authtoken]
   auth_url = http://controller:5000/v3
   memcached_servers = controller:11211
   auth_type = password
   project_domain_name = Default
   user_domain_name = Default
   project_name = service
   username = placement
   password = PLACEMENT_PASS
   EOF
   ```

3. 同步数据库

   ```bash
   su -s /bin/sh -c "placement-manage db sync" placement
   ```

4. 重启httpd服务

   ```shell
   systemctl restart httpd
   ```

5. 检查Placement服务状态

   ```shell
   placement-status upgrade check
   ```

## 安装Nova服务

### 创建数据库

1. 连接数据库

   ```mysql
   mysql -u root -p
   ```

2. 创建nova_api，nova和nova_cell0数据库

   ```mysql
   CREATE DATABASE nova_api;
   CREATE DATABASE nova;
   CREATE DATABASE nova_cell0;
   ```

3. 分别授予三个数据库权限，然后退出

   ```mysql
   # 授权nova_api数据库
   GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
   GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
   # 授权nova数据库
   GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
   GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
   # 授权nova_cell0数据库
   GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
   GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
   exit;
   ```

### 配置用户和Endpoint

1. 创建nova用户并设置密码为NOVA_PASS

   ```shell
   openstack user create --domain default --password NOVA_PASS nova
   ```

2. 使用admin角色将nova用户添加到服务项目中

   ```shell
   # 在service的项目上给nova用户关联admin角色
   openstack role add --project service --user nova admin
   ```

### 创建Nova服务并注册API

1. 创建Nova服务

   ```shell
   openstack service create --name nova --description "OpenStack Compute" compute
   ```

2. 创建Nova服务API端口

   ```shell
   openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
   openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
   openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
   ```

### 安装并配置Nova

1. 安装nova相关软件包

   ```shell
   yum install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler -y
   ```

2. 修改配置文件

   ```bash
   cat > /etc/nova/nova.conf <<EOF
   [DEFAULT]
   enabled_apis = osapi_compute,metadata
   transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
   my_ip = 172.16.10.100
   
   [api_database]
   connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api
   
   [database]
   connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
   
   [api]
   auth_strategy = keystone
   
   [keystone_authtoken]
   www_authenticate_uri = http://controller:5000/
   auth_url = http://controller:5000/
   memcached_servers = controller:11211
   auth_type = password
   project_domain_name = Default
   user_domain_name = Default
   project_name = service
   username = nova
   password = NOVA_PASS
   
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
   password = PLACEMENT_PASS
   ```

3. 同步数据库

   ```shell
   # 同步nova_api数据库
   su -s /bin/sh -c "nova-manage api_db sync" nova
   # 同步nova_cell0数据库
   su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
   # 创建cell1
   su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
   # 同步nova数据库
   su -s /bin/sh -c "nova-manage db sync" nova
   ```

   验证nova_cell0和cell1是否添加成功

   ```shell
   su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
   ```

4. 启动服务并设为开机自启

   ```shell
   systemctl enable openstack-nova-api openstack-nova-scheduler openstack-nova-conductor openstack-nova-novncproxy
   systemctl start openstack-nova-api openstack-nova-scheduler openstack-nova-conductor openstack-nova-novncproxy
   ```

5. 使用命令`nova service-list`验证服务是否启动成功

## 安装neutron服务

### 创建数据库

1. 连接数据库

   ```mysql
   mysql -u root -p
   ```

2. 创建neutron数据库

   ```mysql
   CREATE DATABASE neutron;
   ```

3. 授予数据库权限，然后退出

   ```shell
   GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
   GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
   exit;
   ```

### 配置用户和Endpoint

1. 创建neutron用户并设置密码为NEUTRON_PASS

   ```shell
   openstack user create --domain default --password NEUTRON_PASS neutron
   ```

2. 使用admin角色将neutron用户添加到服务项目中

   ```shell
   # 在service的项目上给neutron用户关联admin角色
   openstack role add --project service --user neutron admin
   ```

### 创建Neutron服务并注册API

1. 创建Neutron服务

   ```shell
   openstack service create --name neutron --description "OpenStack Networking" network
   ```

2. 创建Neutron服务API端口

   ```shell
   openstack endpoint create --region RegionOne network public http://controller:9696
   openstack endpoint create --region RegionOne network internal http://controller:9696
   openstack endpoint create --region RegionOne network admin http://controller:9696
   ```

### 安装并配置neutron服务

1. 安装neutron相关软件包

   ```bash
   yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch python-neutronclient openstack-neutron-fwaas -y
   ```

2. 修改配置文件

   ```bash
   cat > /etc/neutron/neutron.conf <<EOF
   [database]
   connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
   
   [DEFAULT]
   core_plugin = ml2
   service_plugins =
   transport_url = rabbit://openstack:RABBIT_PASS@controller
   auth_strategy = keystone
   notify_nova_on_port_status_changes = true
   notify_nova_on_port_data_changes = true
   
   [keystone_authtoken]
   www_authenticate_uri = http://controller:5000
   auth_url = http://controller:5000
   memcached_servers = controller:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = neutron
   password = NEUTRON_PASS
   
   [nova]
   auth_url = http://controller:5000
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   region_name = RegionOne
   project_name = service
   username = nova
   password = NOVA_PASS
   
   [oslo_concurrency]
   lock_path = /var/lib/neutron/tmp
   EOF
   ```

3. 配置ML2组件

   ```bash
   cat > /etc/neutron/plugins/ml2/ml2_conf.ini <<EOF
   [ml2]
   type_drivers = flat,vlan
   tenant_network_types = vlan
   mechanism_drivers = openvswitch
   debug=True
   [ml2_type_flat]
   [ml2_type_vlan]
   network_vlan_ranges =physnet1:1000:1999,physnet2
   [ml2_type_vxlan]
   [securitygroup]
   enable_security_group = True
   enable_ipset = True
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
   [ovs]
   tenant_network_type = vlan
   bridge_mappings = physnet1:br-vlan,physnet2:br-ex
   EOF
   ```

4. 配置L3代理商

   ```bash
   cat> /etc/neutron/l3_agent.ini<<EOF
   [DEFAULT]
   interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
   router_delete_namespaces = True
   external_network_bridge =
   verbose = True
   [fwaas]
   driver=neutron_fwaas.services.firewall.drivers.linux.iptables_fwaas.IptablesFwaasDriver
   enabled = True
   [agent]
   extensions = fwaas
   [ovs]
   EOF
   ```

5. 配置DHCP代理商

   ```bash
   cat>/etc/neutron/dhcp_agent.ini<<EOF
   [DEFAULT]
   debug=True
   interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
   dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
   enable_isolated_metadata = True
   dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
   EOF
   
   echo "dhcp-option-force=26,1454" >/etc/neutron/dnsmasq-neutron.conf
   ```

6. 配置metadata代理器

   ```bash
   cat> /etc/neutron/metadata_agent.ini <<EOF
   [DEFAULT]
   nova_metadata_host = controller
   metadata_proxy_shared_secret = METADATA_SECRET
   memcache_servers = controller:11211
   EOF
   ```

7. 配置OVS组件

   ```bash
   cat > /etc/neutron/plugins/ml2/openvswitch_agent.ini <<EOF
   [DEFAULT]
   [agent]
   [ovs]
   bridge_mappings = physnet1:br-vlan,physnet2:br-ex
   [securitygroup]
   firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
   enable_security_group=true
   [xenapi]
   EOF
   ```

   配置OVS交换机

   ```bash
   systemctl enable openvswitch.service --now
   systemctl status openvswitch.service
   # 用于联通虚拟机的网桥
   ovs-vsctl add-br br-vlan
   ovs-vsctl add-port br-vlan eth1
   # 用于联通外部网络的网桥
   ovs-vsctl add-br br-ex
   ovs-vsctl add-port br-ex eth2
   ```

8. 配置软链接

   ```bash
   ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
   ```

9. 启动相关服务

   ```bash
   systemctl enable neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent neutron-ovs-cleanup openvswitch
   systemctl start neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent openvswitch
   systemctl status neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent openvswitch
   ```

10. 测试

    ```bash
    . admin-openrc
    openstack network agent list
    ```
    
11. 创建提供商网络（外部网络）

    `provider-physical-network`表示使用的物理网络，与网络节点`/etc/neutron/plugin.ini`中的配置一致。

    ```bash
    . admin-openrc
    # 添加默认的端口安全组规则
    openstack security group rule create --proto icmp default
    openstack security group rule create --proto tcp --dst-port 22 default
    openstack security group rule create --proto tcp --dst-port 3389 default
    openstack security group rule create --proto tcp --dst-port 80 default
    openstack security group rule create --proto tcp --dst-port 443 default
    openstack security group rule create --proto tcp --dst-port 123 default
    openstack security group rule create --proto tcp --dst-port 53 default
    openstack security group rule create --proto udp --dst-port 123 default
    openstack security group rule create --proto udp --dst-port 53 default
    
    openstack network create  --share --external \
      --provider-physical-network physnet2 \
      --provider-network-type flat provider
    
    # 子网需要与真实环境一致
    openstack subnet create --network provider \
      --allocation-pool start=192.168.68.10,end=192.168.68.250 \
      --dns-nameserver 1.2.4.8 --gateway 192.168.68.1 \
      --subnet-range 192.168.68.0/24 provider
    ```

    

# 计算节点

### 安装nova组件

1. 安装软件包

   ```bash
   yum install openstack-nova-compute -y
   ```

2. 修改配置文件

   > 需手动修改vnc下的novncproxy_base_url

   ```bash
   cat > /etc/nova/nova.conf <<EOF
   [DEFAULT]
   enabled_apis = osapi_compute,metadata
   transport_url = rabbit://openstack:RABBIT_PASS@controller
   my_ip = 172.16.10.101
   
   [api]
   auth_strategy = keystone
   
   [keystone_authtoken]
   www_authenticate_uri = http://controller:5000/
   auth_url = http://controller:5000/
   memcached_servers = controller:11211
   auth_type = password
   project_domain_name = Default
   user_domain_name = Default
   project_name = service
   username = nova
   password = NOVA_PASS
   
   [neutron]
   auth_url = http://controller:5000
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   region_name = RegionOne
   project_name = service
   username = neutron
   password = NEUTRON_PASS
   service_metadata_proxy = true
   metadata_proxy_shared_secret = METADATA_SECRET
   
   [vnc]
   enabled = true
   server_listen = 0.0.0.0
   server_proxyclient_address = $my_ip
   novncproxy_base_url = http://172.16.10.100:6080/vnc_auto.html
   
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
   password = PLACEMENT_PASS
   
   [libvirt]
   virt_type=qemu
   EOF
   ```

3. 启动服务

   ```bash
   systemctl enable libvirtd.service openstack-nova-compute.service --now
   systemctl status libvirtd.service openstack-nova-compute.service
   ```

4. 控制端验证nova节点信息

   ```bash
   . admin-openrc
   su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
   openstack hypervisor list
   openstack compute service list
   openstack catalog list
   nova-status upgrade check
   ```

### 安装neutron组件

1. 安装软件包

   ```bash
   yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch
   ```

2. 修改配置文件

   ```bash
   cat > /etc/neutron/neutron.conf <<EOF
   [DEFAULT]
   transport_url = rabbit://openstack:RABBIT_PASS@controller
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
   password = NEUTRON_PASS
   
   [oslo_concurrency]
   lock_path = /var/lib/neutron/tmp
   EOF
   ```

3. 配置OVS网桥

   ```bash
   systemctl enable openvswitch --now
   systemctl status openvswitch
   ovs-vsctl add-br br-vlan
   ovs-vsctl add-port br-vlan eth1
   ```

4. 配置neutron ovs组件

   ```bash
   cat>/etc/neutron/plugins/ml2/openvswitch_agent.ini<<EOF
   [DEFAULT]
   [agent]
   [ovs]
   bridge_mappings = physnet1:br-vlan
   [securitygroup]
   firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
   enable_security_group=True
   [xenapi]
   EOF
   ```

5. 启动服务

   ```bash
   # 重启nova-compute
   systemctl restart openstack-nova-compute
   systemctl enable neutron-openvswitch-agent --now
   systemctl status neutron-openvswitch-agent
   ```

## 访问dashboard

使用浏览器访问 http://172.16.10.100/dashboard，使用admin/ADMIN_PASS登录系统。