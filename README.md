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
TEAM_MASTER="team0"
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
TEAM_MASTER="team0"
TEAM_PORT_CONFIG='{ "prio" : -100 }'
#VAGRANT-END</pre>

<p>Создадим <b>ifcfg-team0</b>:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-team0
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.255.1
NETMASK=255.255.255.252
DEVICE=team0
USERCTL=no
DEVICETYPE="Team"
TEAM_CONFIG='{ "runner" : { "name" : "activebackup", "hwaddr_policy" : "by_active" }, "link_watch" : { "name" : "ethtool" } }'</pre>

<p>Добавим route config файл route-team0:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/route-team0
192.168.0.0/16 via 192.168.255.2</pre>

<p>Перезапустим network сервис:</p>

<pre>[root@inetRouter ~]# systemctl restart network
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
TEAM_MASTER="team0"
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
TEAM_MASTER="team0"
TEAM_PORT_CONFIG='{ "prio" : -100 }'
#VAGRANT-END</pre>

<!-- <p>- <b>ifcfg-eth3</b>:</p>

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
#VAGRANT-END</pre> -->

<p>- добавляем сетевой интерфейс<b>ifcfg-team0</b>:</p>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-team0
NM_CONTROLLED=no
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.255.2
NETMASK=255.255.255.252
DEVICE=team0
USERCTL=no
DEVICETYPE="Team"
TEAM_CONFIG='{ "runner" : { "name" : "activebackup", "hwaddr_policy" : "by_active" }, "link_watch" : { "name" : "ethtool" } }'</pre>

<p>Добавим маршрут по умолчанию:</p>

<pre>[root@centralRouter ~]# echo GATEWAY=192.168.255.1 >> /etc/sysconfig/network-scripts/ifcfg-team0
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

<p>Установим на сервера tracerote и tcpdump.</p>
