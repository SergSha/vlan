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
DEVICE=eth1
ONBOOT=yes
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=no
USERCTL=no</pre>

<p>- <b>ifcfg-eth2</b>:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth2
DEVICE=eth2
ONBOOT=yes
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=no
USERCTL=no</pre>

<p>Создадим <b>ifcfg-bond0</b>:</p>

<pre>[root@inetRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
NAME=bond0
TYPE=Bond
BONDING_MASTER=yes
IPADDR=192.168.255.1
NETMASK=255.255.255.252
ONBOOT=yes
BOOTPROTO=static
BONDING_OPTS="mode=1 miimon=100 fail_over_mac=1"
NM_CONTROLLED=no
USERCTL=no</pre>

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
       valid_lft 84845sec preferred_lft 84845sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:95:5c:6b brd ff:ff:ff:ff:ff:ff
4: <b>eth2</b>: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:8d:ab:9a brd ff:ff:ff:ff:ff:ff
5: eth3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:f4:ba:52 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.11/24 brd 192.168.50.255 scope global noprefixroute eth3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fef4:ba52/64 scope link
       valid_lft forever preferred_lft forever
8: <b>bond0</b>: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
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
DEVICE=eth1
ONBOOT=yes
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=no
USERCTL=no</pre>

<p>- <b>ifcfg-eth2</b>:</p>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth2
DEVICE=eth2
ONBOOT=yes
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=no
USERCTL=no</pre>

<p>- добавляем сетевой интерфейс<b>ifcfg-bond0</b>:</p>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
NAME=bond0
TYPE=Bond
BONDING_MASTER=yes
IPADDR=192.168.255.2
NETMASK=255.255.255.252
GATEWAY=192.168.255.1
ONBOOT=yes
BOOTPROTO=static
BONDING_OPTS="mode=1 miimon=100 fail_over_mac=1"
NM_CONTROLLED=no
USERCTL=no
</pre>

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
       valid_lft 85519sec preferred_lft 85519sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:2f:8a:52 brd ff:ff:ff:ff:ff:ff
4: <b>eth2</b>: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:fc:bb:53 brd ff:ff:ff:ff:ff:ff
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
40: <b>bond0</b>: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:fc:bb:53 brd ff:ff:ff:ff:ff:ff
    inet 192.168.255.2/30 brd 192.168.255.3 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe2f:8a52/64 scope link
       valid_lft forever preferred_lft forever
43: <b>vlan100@eth3</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:fa:38:00 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fefa:3800/64 scope link
       valid_lft forever preferred_lft forever
44: <b>vlan101@eth3</b>: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
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
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=9.84 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=8.73 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=61 time=9.07 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=61 time=8.84 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 8.732/9.124/9.847/0.440 ms
[root@centralRouter ~]#</pre>

<pre>[root@centralRouter ~]# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
06:36:41.459293 IP centralRouter > dns.google: ICMP echo request, id 9571, seq 1, length 64
06:36:41.469109 IP dns.google > centralRouter: ICMP echo reply, id 9571, seq 1, length 64
06:36:42.461708 IP centralRouter > dns.google: ICMP echo request, id 9571, seq 2, length 64
06:36:42.470391 IP dns.google > centralRouter: ICMP echo reply, id 9571, seq 2, length 64
06:36:43.463505 IP centralRouter > dns.google: ICMP echo request, id 9571, seq 3, length 64
06:36:43.472546 IP dns.google > centralRouter: ICMP echo reply, id 9571, seq 3, length 64
06:36:44.465040 IP centralRouter > dns.google: ICMP echo request, id 9571, seq 4, length 64
06:36:44.473839 IP dns.google > centralRouter: ICMP echo reply, id 9571, seq 4, length 64
06:36:46.463878 ARP, Request who-has gateway tell centralRouter, length 28
06:36:46.466651 ARP, Reply gateway is-at 08:00:27:95:5c:6b (oui Unknown), length 46</pre>

<pre>[root@centralRouter ~]# tcpdump -i eth2
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes</pre>

<p>Отключим сетевой интерфейс et2:</p>

<pre>[root@centralRouter ~]# ip l set dev eth1 down
[root@centralRouter ~]#</pre>

<p>Снова запустим ping:</p>

<pre>[root@centralRouter ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=9.23 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=8.83 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=61 time=9.20 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=61 time=9.14 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 8.839/9.105/9.239/0.184 ms
[root@centralRouter ~]#</pre>

<pre>[root@centralRouter ~]# tcpdump -i eth1
tcpdump: eth1: That device is not up
[root@centralRouter ~]#</pre>

<pre>[root@centralRouter ~]# tcpdump -i eth2
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
06:40:44.510464 IP centralRouter > dns.google: ICMP echo request, id 9575, seq 1, length 64
06:40:44.519667 IP dns.google > centralRouter: ICMP echo reply, id 9575, seq 1, length 64
06:40:45.512336 IP centralRouter > dns.google: ICMP echo request, id 9575, seq 2, length 64
06:40:45.521146 IP dns.google > centralRouter: ICMP echo reply, id 9575, seq 2, length 64
06:40:46.514041 IP centralRouter > dns.google: ICMP echo request, id 9575, seq 3, length 64
06:40:46.523167 IP dns.google > centralRouter: ICMP echo reply, id 9575, seq 3, length 64
06:40:47.516081 IP centralRouter > dns.google: ICMP echo request, id 9575, seq 4, length 64
06:40:47.525191 IP dns.google > centralRouter: ICMP echo reply, id 9575, seq 4, length 64</pre>

<p>Как мы видим, после отключения интерфейса eth1 пакеты пошли через интерфейс eth2, то есть соединение bond между серверами inetRouter и centralRouter работает.</p>

<p>Запустим на сервере testClient1 ping до 10.10.10.1, чтобы понять, на какой сервер (testServer1 или testServer2) пойдут пакеты:</p>

<pre>[root@<b>testClient1</b> ~]# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=4.54 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.43 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=1.31 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 1.314/2.432/4.543/1.493 ms
[root@testClient1 ~]# ^C
[root@testClient1 ~]#</pre>

<p>На серверах testServer1 и testServer2 запустим команды tcpdump:</p>

<pre>[root@<b>testServer1</b> ~]# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
07:43:40.730399 IP 10.10.10.254 > testServer1: ICMP echo request, id 24594, seq 1, length 64
07:43:40.730564 ARP, Request who-has 10.10.10.254 tell testServer1, length 28
07:43:40.732737 ARP, Reply 10.10.10.254 is-at 08:00:27:b4:62:d8 (oui Unknown), length 46
07:43:40.732749 IP testServer1 > 10.10.10.254: ICMP echo reply, id 24594, seq 1, length 64
07:43:41.731764 IP 10.10.10.254 > testServer1: ICMP echo request, id 24594, seq 2, length 64
07:43:41.731805 IP testServer1 > 10.10.10.254: ICMP echo reply, id 24594, seq 2, length 64
07:43:42.734222 IP 10.10.10.254 > testServer1: ICMP echo request, id 24594, seq 3, length 64
07:43:42.734270 IP testServer1 > 10.10.10.254: ICMP echo reply, id 24594, seq 3, length 64
07:43:46.739843 ARP, Request who-has testServer1 tell 10.10.10.254, length 46
07:43:46.739881 ARP, Reply testServer1 is-at 08:00:27:cb:ea:2d (oui Unknown), length 28</pre>

<pre>[root@<b>testServer2</b> ~]# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
07:43:40.721701 ARP, Request who-has 10.10.10.254 tell testServer2, length 46</pre>

<p>Как мы наблюдаем, icmp пакеты с сервера testClient1 шли именно на сервер testServer1, то есть по vlan100.</p>

<p>Теперь запустим ping на сервере testClient2 до 10.10.10.1, чтобы понять, на какой сервер (testServer1 или testServer2) пойдут пакеты:</p>

<pre>[root@<b>testClient2</b> ~]# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=4.48 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.49 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=1.43 ms
64 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=1.40 ms
^C
--- 10.10.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 1.404/2.204/4.484/1.317 ms
[root@testClient2 ~]#</pre>

<p>На серверах testServer1 и testServer2 запустим команды tcpdump:</p>

<pre>[root@<b>testServer1</b> ~]# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
07:56:47.264136 ARP, Request who-has 10.10.10.254 tell testServer1, length 46</pre>

<pre>[root@<b>testServer2</b> ~]# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
07:56:47.255303 IP 10.10.10.254 > testServer2: ICMP echo request, id 23124, seq 1, length 64
07:56:47.255481 ARP, Request who-has 10.10.10.254 tell testServer2, length 28
07:56:47.258479 ARP, Reply 10.10.10.254 is-at 08:00:27:bd:d2:56 (oui Unknown), length 46
07:56:47.258487 IP testServer2 > 10.10.10.254: ICMP echo reply, id 23124, seq 1, length 64
07:56:48.257238 IP 10.10.10.254 > testServer2: ICMP echo request, id 23124, seq 2, length 64
07:56:48.257288 IP testServer2 > 10.10.10.254: ICMP echo reply, id 23124, seq 2, length 64
07:56:49.258912 IP 10.10.10.254 > testServer2: ICMP echo request, id 23124, seq 3, length 64
07:56:49.258966 IP testServer2 > 10.10.10.254: ICMP echo reply, id 23124, seq 3, length 64
07:56:50.261067 IP 10.10.10.254 > testServer2: ICMP echo request, id 23124, seq 4, length 64
07:56:50.261131 IP testServer2 > 10.10.10.254: ICMP echo reply, id 23124, seq 4, length 64
07:56:53.271864 ARP, Request who-has testServer2 tell 10.10.10.254, length 46
07:56:53.271885 ARP, Reply testServer2 is-at 08:00:27:83:66:ee (oui Unknown), length 28</pre>

<p>Здесь как мы видим, icmp пакеты с сервера testClient2 уже идут на сервер testServer2, то есть по vlan101.</p>




