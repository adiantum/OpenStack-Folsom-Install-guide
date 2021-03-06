==========================================================
  Руководство по установке OpenStack Folsom
==========================================================

:Версия: 1.1
:Источник: https://github.com/mseknibilel/OpenStack-Folsom-Install-guide
:Перевод: https://github.com/adiantum/OpenStack-Folsom-Install-guide
:Ключевые слова: OpenStack, Folsom, Quantum, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM, Ubuntu Server 12.04 LTE (64 bits).

Авторы
==========

Copyright (C) Bilel Msekni <bilel.msekni@telecom-sudparis.eu>

Эта работа лицензирована в соответствии с Creative Commons Attribution-ShareAlike 3.0 Unported License.
 
Чтобы посмотреть копию лицензии перейдите по ссылке: [ http://creativecommons.org/licenses/by-sa/3.0/ ].

Перевод: Ilya Alekseyev <ilyaalekseyev@acm.org>

Содержание
=================

::

  0. Что это?
  1. Требования
  2. Подготовка
  3. Keystone 
  4. Glance
  5. KVM
  6. OpenVSwitch
  7. Quantum
  8. Nova
  9. Cinder
  10. Horizon
  11. Запуск первой виртуальной машины
  12. Добавление ноды с вычислительными ресурсами
  13. Лицензирование
  14. Контакты
  15. Благодарности
  16. To do

0. Что это?
==============

Руководство по установке OpenStack Folsom это простой и проверенный путь для создания вашей собственной инсталляции платформы OpenStack. 

Version 1.1, 10 Oct 2012

Status: Stable


1. Требования
====================

:Роль ноды: Сетевые интерфейсы
:Контроллер: eth0 (192.168.100.232), eth1 (192.168.100.234), eth2 (192.168.100.236)
:Нода вычислительных ресурсов: eth0 (192.168.100.250), eth1 (192.168.100.252)

**Замечание 1:** Используя это руководство вы можете создать инсталляцию на одной ноде.

**Замечание 2:** Если у вас имеется только два сетевых интерфейса на контроллере, вы также можете использовать это руководство, но вам необходимо проигнорировать все части относящиеся к eth2 и ваши виртуальные машины не будут иметь доступа к Интернет.


2. Подготовка
===============

2.1. Добавление официальных репозиториев OpenStack Folsom
-----------------

* После установки Ubuntu 12.04 Server 64bits, перейдите в режим с правами root и не выходите из этого режима пока не закончите установку::

   sudo su

* Добавьте репозитории OpenStack Folsom к списку репозиториев Ubuntu::

   echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main >> /etc/apt/sources.list.d/folsom.list
   apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 5EDB1B62EC4926EA

* Обновите список пакетов::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

2.2. MySQL & RabbitMQ
------------

* Установите MySQL::

   apt-get install mysql-server python-mysqldb

* Сконфигурируйте MySQL, разрешив все входящие соединения::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* Установите RabbitMQ::

   apt-get install rabbitmq-server 

2.3. Синхронизация нод
------------------

* Установите NTP сервис::

   apt-get install ntp

* Сконфигурируйте NTP сервер для синхронизации между нодой контроллера и нодой вычислительных ресурсов::
   
   sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
   service ntp restart  

2.4. Другие сервисы
-------------------
* Установите другие необходимые сервисы::

   apt-get install vlan bridge-utils

* Разрешите IP_Forwarding::

   nano /etc/sysctl.conf
   #Uncomment net.ipv4.ip\_forward=1

* Вы можете проверить что IP_Forwarding включен выполнив следующую команду::
   
   sysctl -p
   # The valid response should be this: net.ipv4.ip_forward = 1

3. Keystone
=====================================================================

Раздел описывает установку сервиса идентификации OpenStack:

* Установите пакеты keystone::

   apt-get install keystone python-keystone python-keystoneclient

* Создайте новую базу данных в MySQL для keystone::

   mysql -u root -p
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   quit;

* Скорректируйте настройки подключения к новой БД в файле /etc/keystone/keystone.conf::

   connection = mysql://keystoneUser:keystonePass@192.168.100.232/keystone

* Перезапустите сервис идентификации, а затем синхронизируйте БД::

   service keystone restart
   keystone-manage db_sync

* Добавьте в БД keystone записи, используя два скрипта находящиеся в `Scripts folder <https://github.com/adiantum/OpenStack-Folsom-Install-guide/tree/master/Scripts>`_ этого git репозитория. Обратите внимание, что вы ДОЛЖНЫ изменить параметр HOST_IP перед выполнением скриптов::

   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh
   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* Создайте файл с учетными записями и загрузите его в переменные окружения, чтобы не использовать дополнительных параметров при вызове команд в дальнейшем::

   nano creds
   #Вставьте следующее:
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://192.168.100.232:5000/v2.0/"
   # Загрузите в окружение:
   source creds

* Для проверки Keystone можно использовать простой curl запрос::

   apt-get install curl openssl
   curl http://192.168.100.232:35357/v2.0/endpoints -H 'x-auth-token: ADMIN'

4. Glance
=====================================================================

* После установки Keystone, мы проинсталлируем сервис образов виртуальных машин a.k.a Glance::

   apt-get install glance python-glance python-glanceclient

* Создайте новую базу данных MySQL для Glance::

   mysql -u root -p
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
   quit;

* Внесите следующие изменения в /etc/glance/glance-api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Внесите следующие изменения в /etc/glance/glance-registry-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Внесите следующие изменения в /etc/glance/glance-api.conf::

   sql_connection = mysql://glanceUser:glancePass@192.168.100.232/glance

* А также::

   [paste_deploy]
   flavor = keystone

* Поправьте настройки доступа к БД в /etc/glance/glance-registry.conf::

   sql_connection = mysql://glanceUser:glancePass@192.168.100.232/glance

* А также::

   [paste_deploy]
   flavor = keystone

* Перезапустите сервисы glance-api и glance-registry::

   service glance-api restart; service glance-registry restart

* Синхронизируйте БД glance::

   glance-manage db_sync

* Перезапустите сервисы еще раз::

   service glance-registry restart; service glance-api restart

* Для проверки корректности установки Glance's загрузите новый образ в хранилище. Начните с загрузки ubuntu cloud image на вашу ноду и затем сохраните его в Glance::

   mkdir images
   cd images
   wget http://uec-images.ubuntu.com/releases/precise/release/ubuntu-12.04-server-cloudimg-amd64.tar.gz
   tar xzvf ubuntu-12.04-server-cloudimg-amd64.tar.gz
   glance add name="Ubuntu" is_public=true container_format=ovf disk_format=qcow2 < precise-server-cloudimg-amd64.img

* Теперь посмотрите список загруженных образов чтобы убедиться что образ был корректно сохранен::

   glance image-list

5. KVM
=====================================================================

* KVM is needed as the hypervisor that will be used to create virtual machines. Before you install KVM, make sure that your hardware enables virtualization::

   apt-get install cpu-checker
   kvm-ok

* Normally you would get a good response. Now, move to install kvm and configure it::

   apt-get install -y kvm libvirt-bin pm-utils

* Edit the /etc/libvirt/qemu.conf file and uncomment::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* Delete default virtual bridge ::

   virsh net-destroy default
   virsh net-undefine default

* Enable live migration by updating /etc/libvirt/libvirtd.conf file::

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"

* Edit libvirtd_opts variable in /etc/init/libvirt-bin.conf file::

   env libvirtd_opts="-d -l"

* Edit /etc/default/libvirt-bin file ::

   libvirtd_opts="-d -l"

* Restart the libvirt service to load the new values::

   service libvirt-bin restart

6. OpenVSwitch
=====================================================================

* Install the openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* Create the bridges::

   #br-int will be used for integration	
   ovs-vsctl add-br br-int
   #br-eth1 will be used for VM communication 
   ovs-vsctl add-br br-eth1 
   ovs-vsctl add-port br-eth1 eth1
   #br-ex will be used to ensure access to VM from the outside world (a.k.a internet)
   ovs-vsctl add-br br-ex
   ovs-vsctl add-port br-ex eth2

7. Quantum
=====================================================================

First, I am really impressed with this new project, it literaly eliminated the network overhead i used to deal with during the nova-network era.

* Install the Quantum server and the Quantum OVS plugin::

   apt-get install quantum-server python-cliff python-pyparsing quantum-plugin-openvswitch

* Create a database::

   mysql -u root -p
   CREATE DATABASE quantum;
   GRANT ALL ON quantum.* TO 'quantumUser'@'%' IDENTIFIED BY 'quantumPass';
   quit; 

* Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@192.168.100.232/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type=vlan
   network_vlan_ranges = physnet1:1:4094
   bridge_mappings = physnet1:br-eth1

* Restart the quantum server::

   service quantum-server restart

* Install the OVS plugin agent::

   apt-get install quantum-plugin-openvswitch-agent

* Install quantum DHCP and l3 agents::

   apt-get -y install quantum-dhcp-agent
   apt-get -y install quantum-l3-agent

* Edit /etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* In addition, update the /etc/quantum/l3\_agent.ini::

   auth_url = http://192.168.100.232:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* Restart all the services::

   service quantum-server restart
   service quantum-plugin-openvswitch-agent restart
   service quantum-dhcp-agent restart
   service quantum-l3-agent restart

8. Nova
=================

* Start by installing nova components::

   apt-get install -y nova-api nova-cert nova-common novnc nova-compute-kvm nova-consoleauth nova-scheduler nova-novncproxy

* Prepare a Mysql database for Nova::

   mysql -u root -p
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
   quit;

* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova

* Modify the nova.conf like this::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   scheduler_driver=nova.scheduler.simple.SimpleScheduler
   s3_host=192.168.100.232
   ec2_host=192.168.100.232
   ec2_dmz_host=192.168.100.232
   rabbit_host=192.168.100.232
   cc_host=192.168.100.232
   nova_url=http://192.168.100.232:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@192.168.100.232/nova
   ec2_url=http://192.168.100.232:8773/services/Cloud 
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://192.168.100.232:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=192.168.100.232:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.232:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=127.0.0.1
   vncserver_listen=0.0.0.0 

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://192.168.100.232:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://192.168.100.232:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* Don't forget to update the ownership rights of the nova directory::

   chown -R nova. /etc/nova
   chmod 644 /etc/nova/nova.conf

* Add this line to the sudoers file::

   sudo visudo
   #Paste this line anywhere you like:
   nova ALL=(ALL) NOPASSWD:ALL

* Synchronize your database::

   nova-manage db sync

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

9. Cinder
=================

Cinder is the newest OpenStack project and it aims at managing the volumes for VMs. Although Cinder is a replacement of the old nova-volume service, its installation is now a seperated from the nova install process.

* Install the required packages::

   apt-get install cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* Prepare a Mysql database for Cinder::

   mysql -u root -p
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
   quit;

* Configure /etc/cinder/api-paste.ini like the following::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 192.168.100.232
   service_port = 5000
   auth_host = 192.168.100.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* Edit the /etc/cinder/cinder.conf to::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@192.168.100.232/cinder
   api_paste_confg = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

* Then, synchronize your database::

   cinder-manage db sync

* Restart the cinder services::

   service cinder-volume restart
   service cinder-api restart
 

* Finally, don't forget to create a volumegroup and name it cinder-volumes::

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
   write

* Proceed to create the physical volume then the volume group::

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

10. Horizon
============

* To install horizon, proceed like this :::

   apt-get install openstack-dashboard memcached

* Edit /etc/apache2/apache2.conf to add this line::

   ServerName localhost

* I had some issues with the OpenStack ubuntu theme so i disabled it to go back to the default look::

   nano /etc/openstack-dashboard/local_settings.py
   #Comment these lines
   #Enable the Ubuntu theme if it is present.
   #try:
   #    from ubuntu_theme import *
   #except ImportError:
   #    pass

* Reload Apache and memcached::

   service apache2 restart; service memcached restart

You can now access your OpenStack @192.168.100.232/horizon with credentials admin:admin_pass.

11. Your First VM
============

To start your first VM, you will need to create networks for it. This is easy using the new Quantum project but we first need to create a new tenant as it is not recommended to play with the admin tenant. 

* Create a new tenant ::

   keystone tenant-create --name project_one

* Create a new user and assign the admin role to it in the new tenant::

   keystone user-create --name=user_one --pass=user_one --tenant-id $put_id_of_project_one --email=user_one@domain.com
   keystone user-role-add --tenant-id $put_id_of_project_one  --user-id $put_id_of_user_one --role-id $put_id_of_admin_role

* Create a new network for the tenant::

   quantum net-create --tenant-id $put_id_of_project_one net_proj_one --provider:network_type vlan --provider:physical_network physnet1 --provider:segmentation_id 1024

* Create a new subnet inside the new tenant network::

   quantum subnet-create --tenant-id $put_id_of_project_one net_proj_one 10.10.10.0/24

* Create a router for the new tenant::

   quantum router-create --tenant_id $put_id_of_project_one router_proj_one

* Add the router to the subnet::

   quantum router-interface-add $put_router_id_here $put_subnet_id_here

You can now start creating VMs but they will not be accessible from the internet. If you like them to be so, perform the following:

* Create your external network with the tenant id belonging to the service tenant::

   quantum net-create ext_net --tenant-id $SERVICE_TENANT_ID --router:external=True

* Create a subnet containing your floating IPs::

   quantum subnet-create ext_net 192.168.100.10/28 -- --enable_dhcp=False

* Set the router for the external network::

   quantum router-gateway-set $ROUTER_ID $EXT_NET_ID

**This is it !**, You can now login to your OpenStack dashboard and start creating internet accessible VMs.

I Hope you enjoyed this guide, please if you have any feedbacks, don't hesitate.

12. Adding a compute node
=========================

This part is comming soon (Testing Stage)

13. Licensing
============

This work is licensed under the Creative Commons Attribution-ShareAlike 3.0 Unported License.

To view a copy of this license, visit [ http://creativecommons.org/licenses/by-sa/3.0/ and `Licence <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/licence.png>`_ ].

14. Contacts
===========

Bilel Msekni: bilel.msekni@telecom-sudparis.eu

15. Acknowledgment
=================

This work has been based on:

* Emilien Macchi's Folsom guide [https://github.com/EmilienM/openstack-folsom-guide]
* OpenStack Documentation [http://docs.openstack.org/trunk/openstack-compute/install/apt/content/]
* OpenStack Quantum Install [http://docs.openstack.org/trunk/openstack-network/admin/content/ch_install.html]

16. Todo
=======
This guide is just a startup. Your suggestion are all welcomed.

Some of this guide's needs might be:

*




