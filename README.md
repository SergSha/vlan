<h3>### VLAN ###</h3>

<p>Строим бонды и вланы</p>

<h4>Описание домашнего задания</h4>

<p>в Office1 в тестовой подсети появляется сервера с доп интерфесами и адресами в internal сети testLAN</p>
<ul type="disc">
<li>testClient1 - 10.10.10.254</li>
<li>testClient2 - 10.10.10.254</li>
<li>testServer1- 10.10.10.1</li>
<li>testServer2- 10.10.10.1<br />
развести вланами<br />
testClient1 <-> testServer1<br />
testClient2 <-> testServer2<br />
между centralRouter и inetRouter<br />
"пробросить" 2 линка (общая inernal сеть) и объединить их в бонд<br />
проверить работу c отключением интерфейсов</p>

<p>Формат сдачи ДЗ: vagrant + ansible</p>

<h4>1. Работа со стендом и настройка VLAN</h4>

<p>В домашней директории создадим директорию vlan, в котором будут храниться настройки виртуальных машин:</p>

<pre>[user@localhost otus]$ mkdir ./vlan
[user@localhost otus]$</pre>

<p>Перейдём в директорию vlan:</p>

<pre>[user@localhost otus]$ cd ./vlan/
[user@localhost vlan]$</pre>

<p>Создадим Vagrantfile:</p>

<pre>[user@localhost vlan]$ vi ./Vagrantfile</pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :inetRouter => {
    :box_name => "centos/7",
    :vm_name => "inetRouter",
    #:public => {:ip => '10.10.10.1', :adapter => 1},
    :net => [
      {ip: '192.168.255.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "router-net"},
      {ip: '192.168.255.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "router-net"},
      {ip: '192.168.50.11', adapter: 8},
    ]
  },
  :centralRouter => {
    :box_name => "centos/7",
    :vm_name => "centralRouter",
    :net => [
      {ip: '192.168.255.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "router-net"},
      {ip: '192.168.255.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "router-net"},
      {ip: '10.10.10.10', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "dir-net"},
      {ip: '192.168.50.12', adapter: 8},
    ]
  },
  :testServer1 => {
    :box_name => "centos/7",
    :vm_name => "testServer1",
    :net => [
      {ip: '10.10.10.1', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "net-vlan"},
      {ip: '192.168.50.21', adapter: 8},
    ]
  },
  :testServer2 => {
    :box_name => "centos/7",
    :vm_name => "testServer2",
    :net => [
      {ip: '10.10.10.1', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "net-vlan"},
      {ip: '192.168.50.22', adapter: 8},
    ]
  },
  :testClient1 => {
    :box_name => "centos/7",
    :vm_name => "testClient1",
    :net => [
      {ip: '10.10.10.254', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "net-vlan"},
      {ip: '192.168.50.31', adapter: 8},
    ]
  },
  :testClient2 => {
    :box_name => "centos/7",
    :vm_name => "testClient2",
    :net => [
      {ip: '10.10.10.254', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "net-vlan"},
      {ip: '192.168.50.32', adapter: 8},
    ]
  },
}
Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      boxconfig[:net].each do |ipconf|
        box.vm.network "private_network", ipconf
      end
      if boxconfig.key?(:public)
        box.vm.network "public_network", boxconfig[:public]
      end
      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
        cp ~vagrant/.ssh/auth* ~root/.ssh
      SHELL
#      if boxconfig[:vm_name] == "testClient2"
#        box.vm.provision "ansible" do |ansible|
#          ansible.playbook = "ansible/playbook.yml"
#          ansible.inventory_path = "ansible/hosts"
#          ansible.host_key_checking = "false"
#          ansible.limit = "all"
#        end
#      end
    end
  end
end</pre>

<p>Запустим эти виртуальные машины:</p>

<pre>[student@pv-homeworks1-10 vlan]$ vagrant up</pre>

<p>Состояние запущенных виртуальных машин:</p>

<pre>[student@pv-homeworks1-10 vlan]$ vagrant status
Current machine states:

inetRouter                running (virtualbox)
centralRouter             running (virtualbox)
testServer1               running (virtualbox)
testServer2               running (virtualbox)
testClient1               running (virtualbox)
testClient2               running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[student@pv-homeworks1-10 vlan]$</pre>

<p></p>

<pre>[student@pv-homeworks1-10 vlan]$ vagrant ssh inetRouter
[vagrant@inetRouter ~]$ sudo -i
[root@inetRouter ~]#</pre>

<p>Установим iptables, iptables-services:</p>

<pre>[root@inetRouter ~]# yum install -y iptables iptables-services</pre>

<p>Запустим сервис iptables и включим в автозагрузку:</p>

<pre>[root@inetRouter ~]# systemctl enable iptables --now
Created symlink from /etc/systemd/system/basic.target.wants/iptables.service to /usr/lib/systemd/system/iptables.service.
[root@inetRouter ~]#</pre>

<p>Обратим внимание на файл /etc/sysconfig/iptables. Данный файл содержит в себе базовые правила, которые появляются с установкой iptables:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/iptables
# sample configuration for iptables service
# you can edit this manually or use system-config-firewall
# please do not ask us to add additional ports/services to this default configuration
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
#-A INPUT -j REJECT --reject-with icmp-host-prohibited
#-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT</pre>

<p>Настроим NAT для выхода в интернет из нашей сети:</p>

<pre>[root@inetRouter ~]# iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
[root@inetRouter ~]#</pre>

<p>Чтобы после перезагрузки правила iptables не удалились, сохраним эти правила:</p>

<pre>[root@inetRouter ~]# iptables-save > /etc/sysconfig/iptables
[root@inetRouter ~]#</pre>

<p>В файле /etc/sysconfig/iptables закомментируем строки:</p> 

<pre>-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited</pre>

<p>чтобы не запрещали ping между хостами через данный сервер.</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/iptables
# Generated by iptables-save v1.4.21 on Thu Oct  6 09:27:00 2022
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [28:2128]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
COMMIT
# Completed on Thu Oct  6 09:27:00 2022
# Generated by iptables-save v1.4.21 on Thu Oct  6 09:27:00 2022
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [84:6480]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
<b>#-A INPUT -j REJECT --reject-with icmp-host-prohibited</b>
<b>#-A FORWARD -j REJECT --reject-with icmp-host-prohibited</b>
COMMIT
# Completed on Thu Oct  6 09:27:00 2022</pre>

<p>Чтобы изменения правил iptables применились, перезапустим сервис iptables:</p>

<pre>[root@inetRouter ~]# systemctl restart iptables
[root@inetRouter ~]#</pre>

<p>Разрешим forwarding трафика через себя:</p>

<pre>[root@inetRouter ~]# echo net.ipv4.conf.all.forwarding=1 >> /etc/sysctl.d/01-forwarding.conf
[root@inetRouter ~]#</pre>

<p>Применим измененные настройки sysctl:</p>

<pre>[root@inetRouter ~]# sysctl -p /etc/sysctl.d/01-forwarding.conf
net.ipv4.conf.all.forwarding = 1
[root@inetRouter ~]#</pre>

<p>Изменим настройки сетевых интерфейсов:<br />
- <b>ifcfg-eth1</b>:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth1
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=yes
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.255.1
NETMASK=255.255.255.252
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>на:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth1
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
#IPADDR=192.168.255.1
#NETMASK=255.255.255.252
DEVICE=eth1
#PEERDNS=no
USERCTL=no
DEVICETYPE="TeamPort"
TEAM_MASTER="bond0"
TEAM_PORT_CONFIG='{ "prio" : -100 }'
#VAGRANT-END</pre>

<p>- <b>ifcfg-eth2</b>:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth2
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=yes
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.255.1
NETMASK=255.255.255.252
DEVICE=eth2
PEERDNS=no
#VAGRANT-END</pre>

<p>на:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth2
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
#IPADDR=192.168.255.1
#NETMASK=255.255.255.252
DEVICE=eth2
#PEERDNS=no
USERCTL=no
DEVICETYPE="TeamPort"
TEAM_MASTER="bond0"
TEAM_PORT_CONFIG='{ "prio" : -100 }'
#VAGRANT-END</pre>

<p>Создадим <b>ifcfg-bond0</b>:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-bond0
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.255.1
NETMASK=255.255.255.252
DEVICE=bond0
USERCTL=no
DEVICETYPE="Team"
TEAM_CONFIG='{ "runner" : { "name" : "activebackup", "hwaddr_policy" : "by_active" }, "link_watch" : { "name" : "ethtool" } }'</pre>

<p>Добавим route config файл route-bond0:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/route-bond0
192.168.0.0/16 via 192.168.255.2</pre>

<p>Перезапустим network сервис:</p>

<pre>[root@inetRouter ~]# systemctl restart network
[root@inetRouter ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@inetRouter ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 63610sec preferred_lft 63610sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:95:5c:6b brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fe95:5c6b/64 scope link 
       valid_lft forever preferred_lft forever
4: <b>eth2</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:8d:ab:9a brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fe8d:ab9a/64 scope link 
       valid_lft forever preferred_lft forever
5: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:f4:ba:52 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.11/24 brd 192.168.50.255 scope global noprefixroute eth3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fef4:ba52/64 scope link 
       valid_lft forever preferred_lft forever
7: <b>bond0</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:95:5c:6b brd ff:ff:ff:ff:ff:ff
    inet 192.168.255.1/30 brd 192.168.255.3 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe95:5c6b/64 scope link 
       valid_lft forever preferred_lft forever
[root@inetRouter ~]#</pre>

<p>В отдельном терминале подключимся по ssh к centralRouter:</p>

<pre>[student@pv-homeworks1-10 vlan]$ vagrant ssh centralRouter
[vagrant@centralRouter ~]$ sudo -i
[root@centralRouter ~]#</pre>

<p>Также включаем forwarding через сервер:</p>

<pre>[root@centralRouter ~]# echo net.ipv4.conf.all.forwarding=1 >> /etc/sysctl.d/01-forwarding.conf
[root@centralRouter ~]#</pre>

<pre>[root@centralRouter ~]# sysctl -p /etc/sysctl.d/01-forwarding.conf
net.ipv4.conf.all.forwarding = 1
[root@centralRouter ~]#</pre>

<p>Отключаем default route на сетевом интерфейсе eth0:</p>

<pre>[root@centralRouter ~]# echo DEFROUTE=no >> /etc/sysconfig/network-scripts/ifcfg-eth0
[root@centralRouter ~]#</pre>

<p>Также как и на сервере inetRouter внесём изменения в сетевых интерфейсах:</p>

<p>- <b>ifcfg-eth1</b>:</p>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth1
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
#IPADDR=192.168.255.2
#NETMASK=255.255.255.252
DEVICE=eth1
#PEERDNS=no
USERCTL=no
DEVICETYPE="TeamPort"
TEAM_MASTER="bond0"
TEAM_PORT_CONFIG='{ "prio" : -100 }'
#VAGRANT-END</pre>

<p>- <b>ifcfg-eth2</b>:</p>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth2
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
#IPADDR=192.168.255.2
#NETMASK=255.255.255.252
DEVICE=eth2
#PEERDNS=no
USERCTL=no
DEVICETYPE="TeamPort"
TEAM_MASTER="bond0"
TEAM_PORT_CONFIG='{ "prio" : -100 }'
#VAGRANT-END</pre>

<p>- <b>ifcfg-eth3</b>:</p>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth3
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=yes
BOOTPROTO=none
ONBOOT=yes
#IPADDR=10.10.10.10
#NETMASK=255.255.255.0
DEVICE=eth3
PEERDNS=no
#VAGRANT-END</pre>

<p>- добавляем сетевой интерфейс<b>ifcfg-bond0</b>:</p>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-bond0
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.255.2
NETMASK=255.255.255.252
DEVICE=bond0
USERCTL=no
DEVICETYPE="Team"
TEAM_CONFIG='{ "runner" : { "name" : "activebackup", "hwaddr_policy" : "by_active" }, "link_watch" : { "name" : "ethtool" } }'</pre>

<p>Добавим маршрут по умолчанию:</p>

<pre>[root@centralRouter ~]# echo GATEWAY=192.168.255.1 >> /etc/sysconfig/network-scripts/ifcfg-bond0
[root@centralRouter ~]#</pre>

<p>Добавим сетевые интерфейсы для vlan100 и vlan101:</p>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-<b>vlan100</b>
NM_CONTROLLED=no
BOOTPROTO=static
ONBOOT=yes
DEVICE=vlan100
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PHYSDEV=eth3
VLAN_ID=100</pre>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-<b>vlan101</b>
NM_CONTROLLED=no
BOOTPROTO=static
ONBOOT=yes
DEVICE=vlan101
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PHYSDEV=eth3
VLAN_ID=101</pre>

<p>Перезапустим network сервис:</p>

<pre>[root@centralRouter ~]# systemctl restart network
[root@centralRouter ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@centralRouter ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 85441sec preferred_lft 85441sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:2f:8a:52 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fe2f:8a52/64 scope link 
       valid_lft forever preferred_lft forever
4: <b>eth2</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:fc:bb:53 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fefc:bb53/64 scope link 
       valid_lft forever preferred_lft forever
5: <b>eth3</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:fa:38:00 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fefa:3800/64 scope link 
       valid_lft forever preferred_lft forever
6: eth4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:9b:69:a7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.12/24 brd 192.168.50.255 scope global noprefixroute eth4
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe9b:69a7/64 scope link 
       valid_lft forever preferred_lft forever
13: <b>bond0</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:2f:8a:52 brd ff:ff:ff:ff:ff:ff
    inet 192.168.255.2/30 brd 192.168.255.3 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe2f:8a52/64 scope link 
       valid_lft forever preferred_lft forever
14: <b>vlan100@eth3</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:fa:38:00 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fefa:3800/64 scope link 
       valid_lft forever preferred_lft forever
15: <b>vlan101@eth3</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:fa:38:00 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fefa:3800/64 scope link 
       valid_lft forever preferred_lft forever
[root@centralRouter ~]#</pre>

<p>В отдельном терминале подключимся по ssh к testServer1:</p>

<pre>[student@pv-homeworks1-10 vlan]$ vagrant ssh testServer1
[vagrant@testServer1 ~]$ sudo -i
[root@testServer1 ~]#</pre>

<p>Отключаем default route на интерфейсе etho:</p>

<pre>[root@testServer1 ~]# echo DEFROUTE=no >> /etc/sysconfig/network-scripts/ifcfg-eth0
[root@testServer1 ~]#</pre>

<p>Внесём изменение на сетевом интерфейсе eth1:</p>

<pre>[root@testServer1 ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth1
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=yes
BOOTPROTO=none
ONBOOT=yes
#IPADDR=10.10.10.1
#NETMASK=255.255.255.0
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>Добавляем сетевой интерфейс для vlan100:</p>

<pre>[root@testServer1 ~]# vi /etc/sysconfig/network-scripts/ifcfg-vlan100
NM_CONTROLLED=no
BOOTPROTO=static
ONBOOT=yes
IPADDR=10.10.10.1
NETMASK=255.255.255.0
DEVICE=vlan100
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PHYSDEV=eth1
VLAN_ID=100</pre>

<p>Перезапускаем network сервис:</p>

<pre>[root@testServer1 ~]# systemctl restart network
[root@testServer1 ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@testServer1 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 86276sec preferred_lft 86276sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:cb:ea:2d brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fecb:ea2d/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:26:4d:d8 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.21/24 brd 192.168.50.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe26:4dd8/64 scope link 
       valid_lft forever preferred_lft forever
7: <b>vlan100@eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:cb:ea:2d brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global vlan100
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fecb:ea2d/64 scope link 
       valid_lft forever preferred_lft forever
[root@testServer1 ~]#</pre>

<p>Аналогично testServer1 настраиваем настройки сетевых интерфейсов   на серверах testServer2, testClient1, testClient2.</p>

<p>В отдельном терминале подключимся по ssh к testServer2:</p>

<pre>[student@pv-homeworks1-10 vlan]$ vagrant ssh testServer2
[vagrant@testServer2 ~]$ sudo -i
[root@testServer2 ~]#</pre>

<p>Отключаем default route на интерфейсе etho:</p>

<pre>[root@testServer2 ~]# echo DEFROUTE=no >> /etc/sysconfig/network-scripts/ifcfg-eth0
[root@testServer2 ~]#</pre>

<p>Внесём изменение на сетевом интерфейсе eth1:</p>

<pre>[root@testServer2 ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth1
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=yes
BOOTPROTO=none
ONBOOT=yes
#IPADDR=10.10.10.1
#NETMASK=255.255.255.0
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>Добавляем сетевой интерфейс для vlan101:</p>

<pre>[root@testServer2 ~]# vi /etc/sysconfig/network-scripts/ifcfg-vlan101
NM_CONTROLLED=no
BOOTPROTO=static
ONBOOT=yes
IPADDR=10.10.10.1
NETMASK=255.255.255.0
DEVICE=vlan101
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PHYSDEV=eth1
VLAN_ID=101</pre>

<p>Перезапускаем network сервис:</p>

<pre>[root@testServer2 ~]# systemctl restart network
[root@testServer2 ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@testServer2 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 70576sec preferred_lft 70576sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:83:66:ee brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe83:66ee/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:39:c1:06 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.22/24 brd 192.168.50.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe39:c106/64 scope link 
       valid_lft forever preferred_lft forever
5: <b>vlan101@eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:83:66:ee brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global vlan101
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe83:66ee/64 scope link 
       valid_lft forever preferred_lft forever
[root@testServer2 ~]#</pre>

<p>В отдельном терминале подключимся по ssh к testClient1:</p>

<pre>[student@pv-homeworks1-10 vlan]$ vagrant ssh testClient1
[vagrant@testClient1 ~]$ sudo -i
[root@testClient1 ~]#</pre>

<p>Отключаем default route на интерфейсе etho:</p>

<pre>[root@testClient1 ~]# echo DEFROUTE=no >> /etc/sysconfig/network-scripts/ifcfg-eth0
[root@testClient1 ~]#</pre>

<p>Внесём изменение на сетевом интерфейсе eth1:</p>

<pre>[root@testClient1 ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth1
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=yes
BOOTPROTO=none
ONBOOT=yes
#IPADDR=10.10.10.254
#NETMASK=255.255.255.0
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>Добавляем сетевой интерфейс для vlan100:</p>

<pre>[root@testClient1 ~]# vi /etc/sysconfig/network-scripts/ifcfg-vlan100
NM_CONTROLLED=no
BOOTPROTO=static
ONBOOT=yes
IPADDR=10.10.10.254
NETMASK=255.255.255.0
DEVICE=vlan100
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PHYSDEV=eth1
VLAN_ID=100</pre>

<p>Перезапускаем network сервис:</p>

<pre>[root@testClient1 ~]# systemctl restart network
[root@testClient1 ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@testClient1 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 66993sec preferred_lft 66993sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b4:62:d8 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:feb4:62d8/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:eb:2f:95 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.31/24 brd 192.168.50.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feeb:2f95/64 scope link 
       valid_lft forever preferred_lft forever
6: <b>vlan100@eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:b4:62:d8 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.254/24 brd 10.10.10.255 scope global vlan100
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feb4:62d8/64 scope link 
       valid_lft forever preferred_lft forever
[root@testClient1 ~]#</pre>

<p>В отдельном терминале подключимся по ssh к testClient2:</p>

<pre>[student@pv-homeworks1-10 vlan]$ vagrant ssh testClient2
[vagrant@testClient2 ~]$ sudo -i
[root@testClient2 ~]#</pre>

<p>Отключаем default route на интерфейсе etho:</p>

<pre>[root@testClient2 ~]# echo DEFROUTE=no >> /etc/sysconfig/network-scripts/ifcfg-eth0
[root@testClient2 ~]#</pre>

<p>Внесём изменение на сетевом интерфейсе eth1:</p>

<pre>[root@testClient2 ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth1
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
NM_CONTROLLED=yes
BOOTPROTO=none
ONBOOT=yes
#IPADDR=10.10.10.254
#NETMASK=255.255.255.0
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>Добавляем сетевой интерфейс для vlan101:</p>

<pre>[root@testClient2 ~]# vi /etc/sysconfig/network-scripts/ifcfg-vlan101
NM_CONTROLLED=no
BOOTPROTO=static
ONBOOT=yes
IPADDR=10.10.10.254
NETMASK=255.255.255.0
DEVICE=vlan101
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PHYSDEV=eth1
VLAN_ID=101</pre>

<p>Перезапускаем network сервис:</p>

<pre>[root@testClient2 ~]# systemctl restart network
[root@testClient2 ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@testClient2 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 76662sec preferred_lft 76662sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:bd:d2:56 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.254/24 brd 10.10.10.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:febd:d256/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:fa:ea:78 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.32/24 brd 192.168.50.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fefa:ea78/64 scope link 
       valid_lft forever preferred_lft forever
5: <b>vlan101@eth1</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:bd:d2:56 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.254/24 brd 10.10.10.255 scope global vlan101
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:febd:d256/64 scope link 
       valid_lft forever preferred_lft forever
[root@testClient2 ~]#</pre>

<p>Установим на сервера traceroute и tcpdump.</p>











<p>На сервере centralRouter запустим ping до 8.8.8.8, который будет проходить через сервер inetRouter:</p>

<pre>[root@centralRouter ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=20.3 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=19.7 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=61 time=19.9 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=61 time=19.9 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=61 time=19.6 ms
64 bytes from 8.8.8.8: icmp_seq=6 ttl=61 time=19.3 ms
64 bytes from 8.8.8.8: icmp_seq=7 ttl=61 time=19.3 ms
64 bytes from 8.8.8.8: icmp_seq=8 ttl=61 time=19.4 ms
64 bytes from 8.8.8.8: icmp_seq=9 ttl=61 time=19.8 ms
64 bytes from 8.8.8.8: icmp_seq=10 ttl=61 time=19.5 ms
64 bytes from 8.8.8.8: icmp_seq=11 ttl=61 time=19.7 ms
^C
--- 8.8.8.8 ping statistics ---
11 packets transmitted, 11 received, 0% packet loss, time 10019ms
rtt min/avg/max/mdev = 19.365/19.720/20.319/0.335 ms
[root@centralRouter ~]#</pre>

<pre>[root@centralRouter ~]# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
18:38:10.841877 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 1, length 64
18:38:10.862118 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 1, length 64
18:38:11.843549 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 2, length 64
18:38:11.863303 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 2, length 64
18:38:12.845622 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 3, length 64
18:38:12.865541 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 3, length 64
18:38:13.847617 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 4, length 64
18:38:13.867544 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 4, length 64
18:38:14.849648 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 5, length 64
18:38:14.869270 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 5, length 64
18:38:15.839885 ARP, Request who-has gateway tell centralRouter, length 28
18:38:15.841275 ARP, Reply gateway is-at 08:00:27:95:5c:6b (oui Unknown), length 46
18:38:15.851179 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 6, length 64
18:38:15.870549 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 6, length 64
18:38:16.852965 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 7, length 64
18:38:16.872300 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 7, length 64
18:38:17.855557 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 8, length 64
18:38:17.874940 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 8, length 64
18:38:18.857410 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 9, length 64
18:38:18.877129 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 9, length 64
18:38:19.859197 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 10, length 64
18:38:19.878727 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 10, length 64
18:38:20.860959 IP centralRouter > dns.google: ICMP echo request, id 25218, seq 11, length 64
18:38:20.880655 IP dns.google > centralRouter: ICMP echo reply, id 25218, seq 11, length 64
18:38:20.881588 ARP, Request who-has centralRouter tell gateway, length 46
18:38:20.881599 ARP, Reply centralRouter is-at 08:00:27:2f:8a:52 (oui Unknown), length 28</pre>

<pre>[root@centralRouter ~]# tcpdump -i eth2
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes</pre>

<p>Отключим сетевой интерфейс et2:</p>

<pre>[root@centralRouter ~]# ip l set dev eth1 down
[root@centralRouter ~]#</pre>

<p>Снова запустим ping:</p>

<pre>[root@centralRouter ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=22.5 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=19.5 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=61 time=19.7 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=61 time=19.7 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=61 time=19.5 ms
^C
--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4007ms
rtt min/avg/max/mdev = 19.570/20.214/22.506/1.154 ms
[root@centralRouter ~]#</pre>

<pre>[root@centralRouter ~]# tcpdump -i eth2
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
18:45:35.197332 ARP, Request who-has gateway tell centralRouter, length 28
18:45:35.200400 ARP, Reply gateway is-at 08:00:27:95:5c:6b (oui Unknown), length 46
18:45:35.200411 IP centralRouter > dns.google: ICMP echo request, id 25228, seq 1, length 64
18:45:35.219808 IP dns.google > centralRouter: ICMP echo reply, id 25228, seq 1, length 64
18:45:36.199105 IP centralRouter > dns.google: ICMP echo request, id 25228, seq 2, length 64
18:45:36.218652 IP dns.google > centralRouter: ICMP echo reply, id 25228, seq 2, length 64
18:45:37.200775 IP centralRouter > dns.google: ICMP echo request, id 25228, seq 3, length 64
18:45:37.220454 IP dns.google > centralRouter: ICMP echo reply, id 25228, seq 3, length 64
18:45:38.203051 IP centralRouter > dns.google: ICMP echo request, id 25228, seq 4, length 64
18:45:38.222733 IP dns.google > centralRouter: ICMP echo reply, id 25228, seq 4, length 64
18:45:39.204801 IP centralRouter > dns.google: ICMP echo request, id 25228, seq 5, length 64
18:45:39.224342 IP dns.google > centralRouter: ICMP echo reply, id 25228, seq 5, length 64
18:45:40.224700 ARP, Request who-has centralRouter tell gateway, length 46
18:45:40.224714 ARP, Reply centralRouter is-at 08:00:27:fc:bb:53 (oui Unknown), length 28</pre>

<p>Как мы видим, teaming соединение между inetRouter и centralRouter работает.</p>








