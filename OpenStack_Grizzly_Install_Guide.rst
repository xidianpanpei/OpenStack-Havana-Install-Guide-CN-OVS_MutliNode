==========================================================
  OpenStack Havana 安装指南
==========================================================

:Version: 1.0
:Source: https://github.com/xidianpanpei/OpenStack-Havana-Install-Guide-CN-OVS_MutliNode
:Keywords: 多点OpenStack安装, Havana, Neutron, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM, Ubuntu Server 12.04 (64 bits).

作者
==========

`crAzyli0n <http://xidianpanpei.github.como>`_ <pannpei@gmail.com>

本指南fork自
`Shi Dongliang <https://github.com/ist0ne/OpenStack-Havana-Install-Guide-CN>`_ 
的git仓库。  
同时，本指南同时参考
`Bilel Msekni <https://github.com/mseknibilel/OpenStack-Havana-Install-Guide>`_ 
的git仓库。


内容列表
=================

::

  0. 简介
  1. 环境搭建
  2. 控制节点
  3. 网络节点
  4. 计算节点
  5. OpenStack使用
  6. 参考文档


0. 简介
==============

OpenStack Havana安装指南旨在让你轻松创建自己的OpenStack云平台。

状态: Stable


1. 环境搭建
====================

:节点角色: NICs
:控制节点: eth0 (10.10.10.51), eth1 (192.168.100.51)
:网络节点: eth0 (10.10.10.52), eth2 (192.168.100.52)
:计算节点: eth0 (10.10.10.53), eth1 (192.168.100.53)

**注意1:** 你总是可以使用dpkg -s <packagename>确认你使用的是havana软件包(版本: 2013.1)

**注意2:** 这个是当前网络架构

.. image:: http://vdisk.weibo.com/s/uzRmR_K4OJXdG

2. 控制节点
===============

2.1. 准备Ubuntu
-----------------

* 安装好Ubuntu 12.04 Server 64bits后, 进入sudo模式直到完成本指南::

   sudo su -

* 添加Havana仓库::

   apt-get install ubuntu-cloud-keyring python-software-properties software-properties-common python-keyring
   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/havana main >> /etc/apt/sources.list.d/havana.list

* 升级系统::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

2.2.设置网络
------------

* 如下编辑网卡配置文件/etc/network/interfaces:: 

   #Not internet connected(used for OpenStack management)
   auto eth0
   iface eth0 inet static
   address 10.10.10.51
   netmask 255.255.255.0
   gateway 10.10.10.1

   #For Exposing OpenStack API over the internet
   auto eth1
   iface eth1 inet static
   address 192.168.100.51
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

* 重启网络服务::

   service networking restart

* 开启路由转发::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   sysctl -p

2.3. 安装MySQL
------------

* 安装MySQL并为root用户设置密码::

   apt-get install -y mysql-server python-mysqldb

* 配置mysql监听所有网络接口请求::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

2.4. 安装RabbitMQ和NTP
------------

* 安装RabbitMQ::

   apt-get install -y rabbitmq-server 

* 安装NTP服务::

   apt-get install -y ntp

2.5. 创建数据库
------------

* 创建数据库::

   mysql -u root -p
   
   #Keystone
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   
   #Glance
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';

   #Neutron
   CREATE DATABASE neutron;
   GRANT ALL ON neutron.* TO 'neutronUser'@'%' IDENTIFIED BY 'neutronPass';

   #Nova
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';      

   #Cinder
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';

   quit;

2.6. 配置Keystone
------------

* 安装keystone软件包::

   apt-get install -y keystone

* 在/etc/keystone/keystone.conf中设置连接到新创建的数据库::

   connection = mysql://keystoneUser:keystonePass@10.10.10.51/keystone

* 重启身份认证服务并同步数据库::

   service keystone restart
   keystone-manage db_sync

* 使用git仓库中脚本填充keystone数据库： `脚本文件夹 <https://github.com/xidianpanpei/OpenStack-Havana-Install-Guide-CN-OVS_MutliNode/tree/master/KeystoneScripts>`_ ::

   #注意在执行脚本前请按你的网卡配置修改HOST_IP和HOST_IP_EXT

   wget https://github.com/xidianpanpei/OpenStack-Havana-Install-Guide-CN-OVS_MutliNode/blob/master/KeystoneScripts/keystone_basic.sh
   wget https://github.com/xidianpanpei/OpenStack-Havana-Install-Guide-CN-OVS_MutliNode/blob/master/KeystoneScripts/keystone_endpoints_basic.sh

   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* 创建一个简单的凭据文件，这样稍后就不会因为输入过多的环境变量而感到厌烦::

   vi creds-admin

   #Paste the following:
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://192.168.100.51:5000/v2.0/"

   # Load it:
   source creds-admin

* 通过命令行列出Keystone中添加的用户::

   keystone user-list

2.7. 设置Glance
------------

* 安装Glance::

   apt-get install -y glance

* 按下面更新/etc/glance/glance-api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   delay_auth_decision = true
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* 按下面更新/etc/glance/glance-registry-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* 按下面更新/etc/glance/glance-api.conf::

   sql_connection = mysql://glanceUser:glancePass@10.10.10.51/glance

* 和::

   [paste_deploy]
   flavor = keystone
   
* 按下面更新/etc/glance/glance-registry.conf::

   sql_connection = mysql://glanceUser:glancePass@10.10.10.51/glance

* 和::

   [paste_deploy]
   flavor = keystone

* 重启glance-api和glance-registry服务::

   service glance-api restart; service glance-registry restart

* 同步glance数据库::

   glance-manage db_sync

* 重启服务使配置生效::

   service glance-registry restart; service glance-api restart

* 测试Glance, 从网络上传cirros云镜像::

   glance image-create --name cirros --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img

   注意：通过此镜像创建的虚拟机可通过用户名/密码登陆， 用户名：cirros 密码：cubswin:)

* 本地创建Ubuntu云镜像::

   wget http://cloud-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img
   glance add name="Ubuntu 12.04 cloudimg amd64" is_public=true container_format=ovf disk_format=qcow2 < ./precise-server-cloudimg-amd64-disk1.img

* 列出镜像检查是否上传成功::

   glance image-list

2.8. 设置Neutron
------------

* 安装Neutron组件::

   apt-get install -y neutron-server

* 编辑/etc/neutron/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = service_pass

* 编辑OVS配置文件/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini:: 

   #Under the database section
   [DATABASE]
   connection = mysql://neutronUser:neutronPass@10.10.10.51/neutron

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True

   #Firewall driver for realizing neutron security group function
   [SECURITYGROUP]
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* 编辑/etc/neutron/neutron.conf::

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = service_pass
   signing_dir = /var/lib/neutron/keystone-signing

* 重启neutron所有服务::

   cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i restart; done

2.9. 设置Nova
------------------

* 安装nova组件::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor

* 在/etc/nova/api-paste.ini配置文件中修改认证信息::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

* 如下修改/etc/nova/nova.conf::

   [DEFAULT] 
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=10.10.10.51
   nova_url=http://10.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.10.10.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=10.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.10.51
   vncserver_listen=0.0.0.0

   # Network settings
   network_api_class=nova.network.neutronv2.api.API
   neutron_url=http://10.10.10.51:9696
   neutron_auth_strategy=keystone
   neutron_admin_tenant_name=service
   neutron_admin_username=neutron
   neutron_admin_password=service_pass
   neutron_admin_auth_url=http://10.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   #If you want Neutron + Nova Security groups
   #firewall_driver=nova.virt.firewall.NoopFirewallDriver
   #security_group_api=neutron
   #If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   #Metadata
   service_neutron_metadata_proxy = True
   neutron_metadata_proxy_shared_secret = helloOpenStack

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900
 
* 同步数据库::

   nova-manage db sync

* 重启所有nova服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* 检查所有nova服务是否启动正常::

   nova-manage service list

2.10. 设置Cinder
------------------

* 安装软件包::

   apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* 配置iscsi服务::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* 重启服务::
   
   service iscsitarget start
   service open-iscsi start

* 如下配置/etc/cinder/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 192.168.100.51
   service_port = 5000
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* 编辑/etc/cinder/cinder.conf::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@10.10.10.51/cinder
   api_paste_config = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

* 接下来同步数据库::

   cinder-manage db sync

* 最后别忘了创建一个卷组命名为cinder-volumes::

   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2
   #Type in the followings:
   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

* 创建物理卷和卷组::

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

**注意:** 重启后卷组不会自动挂载 (点击`这个 <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/Tricks%26Ideas/load_volume_group_after_system_reboot.rst>`_ 设置在重启后自动挂载) 

* 重启cinder服务::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done

* 确认cinder服务在运行::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; done

2.11. 设置Horizon
------------------

* 如下安装horizon ::

   apt-get install -y openstack-dashboard memcached

* 如果你不喜欢OpenStack ubuntu主题, 你可以停用它::

   dpkg --purge openstack-dashboard-ubuntu-theme

* 重启Apache和memcached服务::

   service apache2 restart; service memcached restart

3. 网络节点
================

3.1. 准备节点
-----------------

* 安装好Ubuntu 12.04 Server 64bits后, 进入sudo模式直到完成本指南::

   sudo su -

* 添加Havana仓库::

   apt-get install ubuntu-cloud-keyring python-software-properties software-properties-common python-keyring
   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/havana main >> /etc/apt/sources.list.d/havana.list

* 升级系统::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

* 安装ntp服务::

   apt-get install -y ntp

* 配置ntp服务从控制节点同步时间::

   #Comment the ubuntu NTP servers
   sed -i 's/server 0.ubuntu.pool.ntp.org/#server 0.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 1.ubuntu.pool.ntp.org/#server 1.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 2.ubuntu.pool.ntp.org/#server 2.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 3.ubuntu.pool.ntp.org/#server 3.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   
   #Set the network node to follow up your conroller node
   sed -i 's/server ntp.ubuntu.com/server 10.10.10.51/g' /etc/ntp.conf

   service ntp restart

3.2. 配置网络
-----------------

* 3块网卡如下设置::

   # OpenStack management
   auto eth0
   iface eth0 inet static
   address 10.10.10.52
   netmask 255.255.255.0

   # VM internet Access
   auto eth2
   iface eth2 inet static
   address 192.168.100.52
   netmask 255.255.255.0

* 开启路由转发::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   sysctl -p


3.3. OpenVSwitch
------------

* 安装OpenVSwitch软件包::

   apt-get install -y openvswitch-controller openvswitch-switch openvswitch-datapath-dkms

* 重新启动openvswitch-switch::

   /etc/init.d/openvswitch-switch restart

* 添加网桥 br-ex 并把网卡 eth1 加入 br-ex::

   ovs-vsctl  add-br br-ex
   ovs-vsctl add-port br-ex eth1

* 如下编辑/etc/network/interfaces::

   # This file describes the network interfaces available on your system
   # and how to activate them. For more information, see interfaces(5).

   # The loopback network interface
   auto lo
   iface lo inet loopback

   # Not internet connected(used for OpenStack management)
   # The primary network interface
   auto eth0
   iface eth0 inet static
   # This is an autoconfigured IPv6 interface
   # iface eth0 inet6 auto
   address 10.10.10.52
   netmask 255.255.255.0

   #For Exposing OpenStack API over the internet
   auto eth1
   iface eth1 inet manual
   up ifconfig $IFACE 0.0.0.0 up
   up ip link set $IFACE promisc on
   down ip link set $IFACE promisc off
   down ifconfig $IFACE down

   auto br-ex
   iface br-ex inet static
   address 192.168.100.52
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

* 重启网络服务::

   /etc/init.d/networking restart

* 创建内网网桥br-int::

   ovs-vsctl add-br br-int

* 查看网桥配置::

   root@openstack-network:~# ovs-vsctl list-br
   br-ex
   br-int

   root@openstack-network:~# ovs-vsctl show
   ebea0b50-e450-41ea-babb-a094ca8d69fa
       Bridge br-int
           Port br-int
               Interface br-int
                   type: internal
       Bridge br-ex
           Port "eth1"
               Interface "eth1"
           Port br-ex
               Interface br-ex
                   type: internal
       ovs_version: "1.4.0+build0"

3.4. Neutron-*
------------

* 安装Neutron组件::

   apt-get -y install neutron-plugin-openvswitch-agent neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent

* 编辑/etc/neutron/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = service_pass

* 编辑OVS配置文件/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini:: 

   #Under the database section
   [DATABASE]
   connection = mysql://neutronUser:neutronPass@10.10.10.51/neutron

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   enable_tunneling = True
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 10.10.10.52

   #Firewall driver for realizing neutron security group function
   [SECURITYGROUP]
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* 更新/etc/neutron/metadata_agent.ini::

   # The Neutron user information for accessing the Neutron API.
   auth_url = http://10.10.10.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = neutron
   admin_password = service_pass

   # IP address used by Nova metadata server
   nova_metadata_ip = 10.10.10.51

   # TCP Port used by Nova metadata server
   nova_metadata_port = 8775

   metadata_proxy_shared_secret = helloOpenStack

* 编辑/etc/neutron/neutron.conf::

   # 确保RabbitMQ IP指向了控制节点
   rabbit_host = 10.10.10.51

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = service_pass
   signing_dir = /var/lib/neutron/keystone-signing

   [DATABASE]
   connection = mysql://neutronUser:neutronPass@10.10.10.51/neutron

* 编辑/etc/neutron/l3_agent.ini::

   [DEFAULT]
   interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
   use_namespaces = True
   external_network_bridge = br-ex
   signing_dir = /var/cache/neutron
   admin_tenant_name = service
   admin_user = neutron
   admin_password = service_pass
   auth_url = http://10.10.10.51:35357/v2.0
   l3_agent_manager = neutron.agent.l3_agent.L3NATAgentWithStateReport
   root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
   interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver

* 编辑/etc/neutron/dhcp_agent.ini::

   [DEFAULT]
   interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
   dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
   use_namespaces = True
   signing_dir = /var/cache/neutron
   admin_tenant_name = service
   admin_user = neutron
   admin_password = service_pass
   auth_url = http://10.10.10.51:35357/v2.0
   dhcp_agent_manager = neutron.agent.dhcp_agent.DhcpAgentWithStateReport
   root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
   state_path = /var/lib/neutron

* 重启neutron所有服务::

   cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i restart; done

4. 计算节点
================

4.1. 准备节点
-----------------

* 安装好Ubuntu 12.04 Server 64bits后,升级系统并重启之后,进入sudo模式直到完成本指南::

   sudo su -

* 添加Havana仓库::

   apt-get install ubuntu-cloud-keyring python-software-properties software-properties-common python-keyring
   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/havana main >> /etc/apt/sources.list.d/havana.list

* 升级系统::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

* 安装ntp服务::

   apt-get install -y ntp

* 配置ntp服务从控制节点同步时间::

   #Comment the ubuntu NTP servers
   sed -i 's/server 0.ubuntu.pool.ntp.org/#server 0.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 1.ubuntu.pool.ntp.org/#server 1.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 2.ubuntu.pool.ntp.org/#server 2.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 3.ubuntu.pool.ntp.org/#server 3.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   
   #Set the network node to follow up your conroller node
   sed -i 's/server ntp.ubuntu.com/server 10.10.10.51/g' /etc/ntp.conf

   service ntp restart

4.2. 配置网络
-----------------

* 如下配置网络::

   # This file describes the network interfaces available on your system
   # and how to activate them. For more information, see interfaces(5).

   # The loopback network interface
   auto lo
   iface lo inet loopback

   # Not internet connected(used for OpenStack management)
   # The primary network interface
   auto eth0
   iface eth0 inet static
   address 10.10.10.53
   netmask 255.255.255.0
   gateway 10.10.10.1

   # VM Configuration
   auto eth1
   iface eth1 inet static
   address 192.168.100.53
   netmask 255.255.255.0
   gateway 192.168.100.1

* 开启路由转发::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   sysctl -p

4.3. KVM
------------------

* 确保你的硬件启用virtualization::

   apt-get install cpu-checker
   kvm-ok

* 现在安装kvm并配置它::

   apt-get install -y kvm libvirt-bin pm-utils

* 在/etc/libvirt/qemu.conf配置文件中启用cgroup_device_acl数组::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* 删除默认的虚拟网桥::

   virsh net-destroy default
   virsh net-undefine default

* 更新/etc/libvirt/libvirtd.conf配置文件::

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"

* E编辑libvirtd_opts变量在/etc/init/libvirt-bin.conf配置文件中::

   env libvirtd_opts="-d -l"

* 编辑/etc/default/libvirt-bin文件 ::

   libvirtd_opts="-d -l"

* 重启libvirt服务使配置生效::

   service libvirt-bin restart

4.4. OpenVSwitch
------------------

* 安装OpenVSwitch软件包::

   apt-get install -y openvswitch-controller openvswitch-switch openvswitch-datapath-dkms

* 重启openvswitch-switch::

   /etc/init.d/openvswitch-switch restart

* 创建网桥::

   #br-int will be used for VM integration   
   ovs-vsctl add-br br-int

4.5. Neutron
------------------

* 安装Neutron openvswitch代理::

   apt-get -y install neutron-plugin-openvswitch-agent

* 编辑OVS配置文件/etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini:: 

   #Under the database section
   [DATABASE]
   connection = mysql://neutronUser:neutronPass@10.10.10.51/neutron

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 10.10.10.51
   enable_tunneling = True

   #Firewall driver for realizing neutron security group function
   [SECURITYGROUP]
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* 编辑/etc/neutron/neutron.conf::

   # 确保RabbitMQ IP指向了控制节点
   rabbit_host = 10.10.10.51

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = service_pass
   signing_dir = /var/lib/neutron/keystone-signing

   [DATABASE]
   connection = mysql://neutronUser:neutronPass@10.10.10.51/neutron

* 重启neutron openvswitch代理服务::

   service neutron-plugin-openvswitch-agent restart

4.6. Nova
------------------

* 安装nova组件::

   apt-get install -y nova-compute-kvm

   注意：如果你的宿主机不支持kvm虚拟化，可把nova-compute-kvm换成nova-compute-qemu
   同时/etc/nova/nova-compute.conf配置文件中的libvirt_type=qemu

* 在/etc/nova/api-paste.ini配置文件中修改认证信息::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

* 如下修改/etc/nova/nova.conf::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=10.10.10.51
   nova_url=http://10.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.10.10.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
   
   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   
   # Imaging service
   glance_api_servers=10.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService
   
   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.10.53
   vncserver_listen=0.0.0.0
   
   # Network settings
   network_api_class=nova.network.neutronv2.api.API
   neutron_url=http://10.10.10.51:9696
   neutron_auth_strategy=keystone
   neutron_admin_tenant_name=service
   neutron_admin_username=neutron
   neutron_admin_password=service_pass
   neutron_admin_auth_url=http://10.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   #If you want Neutron + Nova Security groups
   #firewall_driver=nova.virt.firewall.NoopFirewallDriver
   #security_group_api=neutron
   #If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   
   #Metadata
   service_neutron_metadata_proxy = True
   neutron_metadata_proxy_shared_secret = helloOpenStack
   
   # Compute #
   compute_driver=libvirt.LibvirtDriver
   
   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900
   cinder_catalog_info=volume:cinder:internalURL

* 修改/etc/nova/nova-compute.conf::

   [DEFAULT]
   libvirt_type=kvm
   compute_driver=libvirt.LibvirtDriver
   libvirt_ovs_bridge=br-int
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   libvirt_use_virtio_for_bridges=True

* 重启所有nova服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* 检查所有nova服务是否启动正常::

   nova-manage service list

5. 结束语 
================
由于项目组需求要求，该搭建教程中没有搭建Havana版本中新添加的Ceilmeter和Heat组件。

通过对Havana版本的搭建可以看到，Havana版本的整个搭建过程相比于Grizzly版本来说，并没有发生很明显的变化，主要是有些参数配置所在文件有所变更。同时，针对使用OpenvSwitch插件，其中原来中文版中的openvswitch-brcompat已经无法匹配Havana版本，因此此部分教程参照了英文版教程更改而来。

ps:在搭建过程中细心真的是非常重要的，否则，经常会因为某个配置错误，导致花费大量时间查错。

6. 参考文档
================

`Boostrapping Open vSwitch and Neutron <https://a248.e.akamai.net/cdn.hpcloudsvc.com/h9f25be84b35c201beea6b13c85876258/prodaw2/Bootstrapping_OVS_neutron--final_20130319.html>`_

`Cisco OpenStack Edition: Folsom Manual Install <http://docwiki.cisco.com/wiki/Cisco_OpenStack_Edition:_Folsom_Manual_Install>`_


