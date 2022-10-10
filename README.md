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

<img src="network.png" alt="network.png" />

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
      {ip: '192.168.255.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "bond-net"},
      {ip: '192.168.255.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "bond-net"},
      {ip: '192.168.50.11', adapter: 8},
    ]
  },
  :centralRouter => {
    :box_name => "centos/7",
    :vm_name => "centralRouter",
    :net => [
      {ip: '192.168.255.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "bond-net"},
      {ip: '192.168.255.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "bond-net"},
      {ip: '10.10.10.10', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "vlan-net"},
      {ip: '192.168.50.12', adapter: 8},
    ]
  },
  :testServer1 => {
    :box_name => "centos/7",
    :vm_name => "testServer1",
    :net => [
      {ip: '10.10.10.1', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "vlan-net"},
      {ip: '192.168.50.21', adapter: 8},
    ]
  },
  :testServer2 => {
    :box_name => "centos/7",
    :vm_name => "testServer2",
    :net => [
      {ip: '10.10.10.1', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "vlan-net"},
      {ip: '192.168.50.22', adapter: 8},
    ]
  },
  :testClient1 => {
    :box_name => "centos/7",
    :vm_name => "testClient1",
    :net => [
      {ip: '10.10.10.254', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "vlan-net"},
      {ip: '192.168.50.31', adapter: 8},
    ]
  },
  :testClient2 => {
    :box_name => "centos/7",
    :vm_name => "testClient2",
    :net => [
      {ip: '10.10.10.254', adapter: 2, netmask: "255.255.255.0", virtualbox__intnet: "vlan-net"},
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

<h4>inetRouter</h4>

<p>Подключаемся по ssh к серверу inetRouter</p>

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
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
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
10.10.10.0/24 via 192.168.255.2</pre>

<p>Перезапустим network сервис:</p>

<pre>[root@inetRouter ~]# systemctl restart network
[root@inetRouter ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@inetRouter ~]# ip -d a
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff promiscuity 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 81495sec preferred_lft 81495sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: &lt;BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:f3:37:2d brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>bond_slave state ACTIVE mii_status UP link_failure_count 0 perm_hwaddr 08:00:27:f3:37:2d queue_id 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
4: <b>eth2</b>: &lt;BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:bc:fb:e7 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>bond_slave state BACKUP mii_status UP link_failure_count 0 perm_hwaddr 08:00:27:bc:fb:e7 queue_id 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
5: <b>eth3</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:7c:27:06 brd ff:ff:ff:ff:ff:ff promiscuity 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 192.168.50.11/24 brd 192.168.50.255 scope global noprefixroute eth3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe7c:2706/64 scope link 
       valid_lft forever preferred_lft forever
6: <b>bond0</b>: &lt;BROADCAST,MULTICAST,MASTER,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:f3:37:2d brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>bond mode active-backup active_slave eth1 miimon 100 updelay 0 downdelay 0 use_carrier 1 arp_interval 0 arp_validate none arp_all_targets any primary_reselect always fail_over_mac active xmit_hash_policy layer2 resend_igmp 1 num_grat_arp 1 all_slaves_active 0 min_links 0 lp_interval 1 packets_per_slave 1 lacp_rate slow ad_select stable tlb_dynamic_lb 1 numtxqueues 16 numrxqueues 16 gso_max_size 65536 gso_max_segs 65535</b> 
    inet 192.168.255.1/30 brd 192.168.255.3 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fef3:372d/64 scope link 
       valid_lft forever preferred_lft forever
[root@inetRouter ~]#</pre>

<h4>centralRouter</h4>

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
USERCTL=no</pre>

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
ONBOOT=yes
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
DEVICE=vlan100
PHYSDEV=eth3
VLAN_ID=100
BOOTPROTO=static
#IPADDR=10.10.10.10
#NETMASK=255.255.255.0
NM_CONTROLLED=no</pre>

<pre>[root@centralRouter ~]# vi /etc/sysconfig/network-scripts/ifcfg-<b>vlan101</b>
ONBOOT=yes
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
DEVICE=vlan101
PHYSDEV=eth3
VLAN_ID=101
BOOTPROTO=static
#IPADDR=10.10.10.10
#NETMASK=255.255.255.0
NM_CONTROLLED=no</pre>

<p>Перезапустим network сервис:</p>

<pre>[root@centralRouter ~]# systemctl restart network
[root@centralRouter ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@centralRouter ~]# ip -d a
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff promiscuity 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 86359sec preferred_lft 86359sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: &lt;BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:82:05:2d brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>bond_slave state ACTIVE mii_status UP link_failure_count 0 perm_hwaddr 08:00:27:82:05:2d queue_id 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
4: <b>eth2</b>: &lt;BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast master bond0 state UP group default qlen 1000
    link/ether 08:00:27:1f:a0:e3 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>bond_slave state BACKUP mii_status UP link_failure_count 0 perm_hwaddr 08:00:27:1f:a0:e3 queue_id 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
5: <b>eth3</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:95:31:7d brd ff:ff:ff:ff:ff:ff promiscuity 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet6 fe80::a00:27ff:fe95:317d/64 scope link 
       valid_lft forever preferred_lft forever
6: eth4: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:e5:d4:0c brd ff:ff:ff:ff:ff:ff promiscuity 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 192.168.50.12/24 brd 192.168.50.255 scope global noprefixroute eth4
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fee5:d40c/64 scope link 
       valid_lft forever preferred_lft forever
7: <b>bond0</b>: &lt;BROADCAST,MULTICAST,MASTER,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:82:05:2d brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>bond mode active-backup active_slave eth1 miimon 100 updelay 0 downdelay 0 use_carrier 1 arp_interval 0 arp_validate none arp_all_targets any primary_reselect always fail_over_mac active xmit_hash_policy layer2 resend_igmp 1 num_grat_arp 1 all_slaves_active 0 min_links 0 lp_interval 1 packets_per_slave 1 lacp_rate slow ad_select stable tlb_dynamic_lb 1 numtxqueues 16 numrxqueues 16 gso_max_size 65536 gso_max_segs 65535</b> 
    inet 192.168.255.2/30 brd 192.168.255.3 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe82:52d/64 scope link 
       valid_lft forever preferred_lft forever
8: <b>vlan100@eth3</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:95:31:7d brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>vlan protocol 802.1Q id 100 &lt;REORDER_HDR&gt; numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
    inet6 fe80::a00:27ff:fe95:317d/64 scope link 
       valid_lft forever preferred_lft forever
9: <b>vlan101@eth3</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:95:31:7d brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>vlan protocol 802.1Q id 101 &lt;REORDER_HDR&gt; numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
    inet6 fe80::a00:27ff:fe95:317d/64 scope link 
       valid_lft forever preferred_lft forever
[root@centralRouter ~]# </pre>

<h4>testServer1</h4>

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
<b>#IPADDR=10.10.10.1</b>
<b>#NETMASK=255.255.255.0</b>
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>Добавляем сетевой интерфейс для vlan100:</p>

<pre>[root@testServer1 ~]# vi /etc/sysconfig/network-scripts/ifcfg-vlan100
ONBOOT=yes
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
DEVICE=vlan100
PHYSDEV=eth1
VLAN_ID=100
BOOTPROTO=static
IPADDR=10.10.10.1
NETMASK=255.255.255.0
#GATEWAY=10.10.10.10
NM_CONTROLLED=no</pre>

<p>Перезапускаем network сервис:</p>

<pre>[root@testServer1 ~]# systemctl restart network
[root@testServer1 ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@testServer1 ~]# ip -d a
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 86393sec preferred_lft 86393sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:cb:ea:2d brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fecb:ea2d/64 scope link
       valid_lft forever preferred_lft forever
4: eth2: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:26:4d:d8 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.21/24 brd 192.168.50.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe26:4dd8/64 scope link
       valid_lft forever preferred_lft forever
5: <b>vlan100@eth1</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:93:84:cf brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>vlan protocol 802.1Q id 100 &lt;REORDER_HDR&gt; numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
    inet 10.10.10.1/24 brd 10.10.10.255 scope global vlan100
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe93:84cf/64 scope link 
       valid_lft forever preferred_lft forever

[root@testServer1 ~]#</pre>

<p>Аналогично testServer1 настраиваем настройки сетевых интерфейсов   на серверах testServer2, testClient1, testClient2.</p>

<h4>testServer2</h4>

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
<b>#IPADDR=10.10.10.1</b>
<b>#NETMASK=255.255.255.0</b>
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>Добавляем сетевой интерфейс для vlan101:</p>

<pre>[root@testServer2 ~]# vi /etc/sysconfig/network-scripts/ifcfg-vlan101
ONBOOT=yes
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
DEVICE=vlan101
PHYSDEV=eth1
VLAN_ID=101
BOOTPROTO=static
IPADDR=10.10.10.1
NETMASK=255.255.255.0
#GATEWAY=10.10.10.10
NM_CONTROLLED=no</pre>

<p>Перезапускаем network сервис:</p>

<pre>[root@testServer2 ~]# systemctl restart network
[root@testServer2 ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@testServer2 ~]# ip -d a
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 86376sec preferred_lft 86376sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:83:66:ee brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fe83:66ee/64 scope link
       valid_lft forever preferred_lft forever
4: eth2: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:39:c1:06 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.22/24 brd 192.168.50.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe39:c106/64 scope link
       valid_lft forever preferred_lft forever
5: <b>vlan101@eth1</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:58:e0:f3 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>vlan protocol 802.1Q id 101 &lt;REORDER_HDR&gt; numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
    inet 10.10.10.1/24 brd 10.10.10.255 scope global vlan101
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe58:e0f3/64 scope link 
       valid_lft forever preferred_lft forever
[root@testServer2 ~]#</pre>

<h4>testClient1</h4>

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
<b>#IPADDR=10.10.10.254</b>
<b>#NETMASK=255.255.255.0</b>
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>Добавляем сетевой интерфейс для vlan100:</p>

<pre>[root@testClient1 ~]# vi /etc/sysconfig/network-scripts/ifcfg-vlan100
ONBOOT=yes
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
DEVICE=vlan100
PHYSDEV=eth1
VLAN_ID=100
BOOTPROTO=static
IPADDR=10.10.10.254
NETMASK=255.255.255.0
#GATEWAY=10.10.10.10
NM_CONTROLLED=no</pre>

<p>Перезапускаем network сервис:</p>

<pre>[root@testClient1 ~]# systemctl restart network
[root@testClient1 ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@testClient1 ~]# ip -d a
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 86394sec preferred_lft 86394sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b4:62:d8 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:feb4:62d8/64 scope link
       valid_lft forever preferred_lft forever
4: eth2: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:eb:2f:95 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.31/24 brd 192.168.50.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feeb:2f95/64 scope link
       valid_lft forever preferred_lft forever
5: <b>vlan100@eth1</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:c3:fc:bc brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>vlan protocol 802.1Q id 100 &lt;REORDER_HDR&gt; numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
    inet 10.10.10.254/24 brd 10.10.10.255 scope global vlan100
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fec3:fcbc/64 scope link 
       valid_lft forever preferred_lft forever
[root@testClient1 ~]#</pre>

<h4>testClient2</h4>

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
<b>#IPADDR=10.10.10.254</b>
<b>#NETMASK=255.255.255.0</b>
DEVICE=eth1
PEERDNS=no
#VAGRANT-END</pre>

<p>Добавляем сетевой интерфейс для vlan101:</p>

<pre>[root@testClient2 ~]# vi /etc/sysconfig/network-scripts/ifcfg-vlan101
ONBOOT=yes
TYPE=Ethernet
VLAN=yes
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
DEVICE=vlan101
PHYSDEV=eth1
VLAN_ID=101
BOOTPROTO=static
IPADDR=10.10.10.254
NETMASK=255.255.255.0
#GATEWAY=10.10.10.10
NM_CONTROLLED=no</pre>

<p>Перезапускаем network сервис:</p>

<pre>[root@testClient2 ~]# systemctl restart network
[root@testClient2 ~]#</pre>

<p>Смотрим список сетевых интерфейсов:</p>

<pre>[root@testClient2 ~]# ip -d a
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 86392sec preferred_lft 86392sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link
       valid_lft forever preferred_lft forever
3: <b>eth1</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:bd:d2:56 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:febd:d256/64 scope link
       valid_lft forever preferred_lft forever
4: eth2: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:fa:ea:78 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.32/24 brd 192.168.50.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fefa:ea78/64 scope link
       valid_lft forever preferred_lft forever
5: <b>vlan101@eth1</b>: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:cf:60:e4 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    <b>vlan protocol 802.1Q id 101 &lt;REORDER_HDR&gt; numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535</b> 
    inet 10.10.10.254/24 brd 10.10.10.255 scope global vlan101
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fecf:60e4/64 scope link 
       valid_lft forever preferred_lft forever
[root@testClient2 ~]#</pre>

<h4>2. Проверка работы тестового стенда "Bond-Vlan"</h4>

<p><b>&bull; Bond:</b></p>

<p>Установим на все сервера traceroute и tcpdump.</p>

<p>На сервере centralRouter запустим ping до 8.8.8.8, который будет проходить через сервер inetRouter:</p>

<pre>[root@<b>centralRouter</b> ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=32.7 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=29.0 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=61 time=30.2 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=61 time=28.1 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 28.166/30.048/32.744/1.729 ms
[root@centralRouter ~]#</pre>

<p>На сервере inetRouter запустим tcpdump на интерфейсах bond0, eth1, eth2:</p>

<pre>[root@<b>inetRouter</b> ~]# tcpdump -nvvv -e -i <b>bond0</b>
tcpdump: listening on bond0, link-type EN10MB (Ethernet), capture size 262144 bytes
19:01:35.656780 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 25373, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24508, seq 1, length 64
19:01:35.688995 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 20962, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24508, seq 1, length 64
19:01:36.659820 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 26330, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24508, seq 2, length 64
19:01:36.687620 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 20967, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24508, seq 2, length 64
19:01:37.661557 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 26409, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24508, seq 3, length 64
19:01:37.690437 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 20972, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24508, seq 3, length 64
19:01:38.663490 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 26646, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24508, seq 4, length 64
19:01:38.690206 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 20977, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24508, seq 4, length 64
19:01:40.662474 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype ARP (0x0806), length 60: Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.255.1 tell 192.168.255.2, length 46
19:01:40.662531 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype ARP (0x0806), length 42: Ethernet (len 6), IPv4 (len 4), Reply 192.168.255.1 is-at 08:00:27:f3:37:2d, length 28
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[root@inetRouter ~]#</pre>

<pre>[root@<b>inetRouter</b> ~]# tcpdump -nvvv -e -i <b>eth1</b>
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
19:01:35.656780 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 25373, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24508, seq 1, length 64
19:01:35.689004 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 20962, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24508, seq 1, length 64
19:01:36.659820 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 26330, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24508, seq 2, length 64
19:01:36.687645 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 20967, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24508, seq 2, length 64
19:01:37.661557 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 26409, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24508, seq 3, length 64
19:01:37.690467 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 20972, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24508, seq 3, length 64
19:01:38.663490 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 26646, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24508, seq 4, length 64
19:01:38.690231 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 20977, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24508, seq 4, length 64
19:01:40.662474 08:00:27:82:05:2d > 08:00:27:f3:37:2d, ethertype ARP (0x0806), length 60: Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.255.1 tell 192.168.255.2, length 46
19:01:40.662538 08:00:27:f3:37:2d > 08:00:27:82:05:2d, ethertype ARP (0x0806), length 42: Ethernet (len 6), IPv4 (len 4), Reply 192.168.255.1 is-at 08:00:27:f3:37:2d, length 28
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[root@inetRouter ~]#</pre>

<pre>[root@<b>inetRouter</b> ~]# tcpdump -nvvv -e -i <b>eth2</b>
tcpdump: listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
[root@inetRouter ~]#</pre>

<p>Как мы наблюдаем, ping проходит, в данном случае, через интерфейс eth1.</p>

<p>Теперь на сервере inetRouter отключим сетевой интерфейс eth1:</p>

<pre>[root@<b>inetRouter</b> ~]# ip l set dev eth1 down
[root@inetRouter ~]#</pre>

<p>И снова повторим:</p>

<pre>[root@<b>centralRouter</b> ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=61 time=28.1 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=61 time=25.1 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=61 time=27.6 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=61 time=27.9 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 25.148/27.217/28.113/1.204 ms
[root@centralRouter ~]#</pre>

<pre>[root@<b>inetRouter</b> ~]# tcpdump -nvvv -e -i <b>bond0</b>
tcpdump: listening on bond0, link-type EN10MB (Ethernet), capture size 262144 bytes
19:13:19.762685 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 50249, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24517, seq 1, length 64
19:13:19.789540 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 21440, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24517, seq 1, length 64
19:13:20.763962 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 50893, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24517, seq 2, length 64
19:13:20.788805 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 21445, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24517, seq 2, length 64
19:13:21.766694 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 51063, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24517, seq 3, length 64
19:13:21.793039 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 21450, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24517, seq 3, length 64
19:13:22.769066 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 51658, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24517, seq 4, length 64
19:13:22.795772 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 21455, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24517, seq 4, length 64
19:13:24.758662 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype ARP (0x0806), length 60: Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.255.1 tell 192.168.255.2, length 46
19:13:24.758782 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype ARP (0x0806), length 42: Ethernet (len 6), IPv4 (len 4), Reply 192.168.255.1 is-at 08:00:27:bc:fb:e7, length 28
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[root@inetRouter ~]#</pre>

<pre>[root@<b>inetRouter</b> ~]# tcpdump -nvvv -e -i <b>eth1</b>
tcpdump: eth1: That device is not up
[root@inetRouter ~]#</pre>

<pre>[root@<b>inetRouter</b> ~]# tcpdump -nvvv -e -i <b>eth2</b>
tcpdump: listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
19:13:19.762685 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 50249, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24517, seq 1, length 64
19:13:19.789565 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 21440, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24517, seq 1, length 64
19:13:20.763962 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 50893, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24517, seq 2, length 64
19:13:20.788814 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 21445, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24517, seq 2, length 64
19:13:21.766694 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 51063, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24517, seq 3, length 64
19:13:21.793075 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 21450, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24517, seq 3, length 64
19:13:22.769066 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 51658, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.255.2 > 8.8.8.8: ICMP echo request, id 24517, seq 4, length 64
19:13:22.795797 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 61, id 21455, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 192.168.255.2: ICMP echo reply, id 24517, seq 4, length 64
19:13:24.758662 08:00:27:1f:a0:e3 > 08:00:27:bc:fb:e7, ethertype ARP (0x0806), length 60: Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.255.1 tell 192.168.255.2, length 46
19:13:24.758791 08:00:27:bc:fb:e7 > 08:00:27:1f:a0:e3, ethertype ARP (0x0806), length 42: Ethernet (len 6), IPv4 (len 4), Reply 192.168.255.1 is-at 08:00:27:bc:fb:e7, length 28
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[root@inetRouter ~]#</pre>

<p>Как мы видим, после отключения интерфейса eth1 пакеты пошли через интерфейс eth2, то есть соединение bond между серверами inetRouter и centralRouter сохраняется и работает.</p>

<p><b>&bull; Vlan:</b></p>

<p>Запустим на сервере testClient1 ping до 10.10.10.1, чтобы понять, на какой сервер (testServer1 или testServer2) пойдут пакеты:</p>

<pre>[root@<b>testClient1</b> ~]# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=1.38 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.36 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=1.24 ms
64 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=1.30 ms
^C
--- 10.10.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3009ms
rtt min/avg/max/mdev = 1.245/1.325/1.389/0.070 ms
[root@testClient1 ~]#</pre>

<p>На серверах testServer1 и testServer2 запустим команды tcpdump:</p>

<pre>[root@<b>testServer1</b> ~]# tcpdump -nvvv -e -i eth1
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
18:37:38.661666 08:00:27:c3:fc:bc > 08:00:27:93:84:cf, ethertype 802.1Q (0x8100), length 102: vlan 100, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 42672, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.254 > 10.10.10.1: ICMP echo request, id 23202, seq 1, length 64
18:37:38.661805 08:00:27:93:84:cf > 08:00:27:c3:fc:bc, ethertype 802.1Q (0x8100), length 102: vlan 100, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 30947, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.254: ICMP echo reply, id 23202, seq 1, length 64
18:37:39.663366 08:00:27:c3:fc:bc > 08:00:27:93:84:cf, ethertype 802.1Q (0x8100), length 102: vlan 100, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 43056, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.254 > 10.10.10.1: ICMP echo request, id 23202, seq 2, length 64
18:37:39.663459 08:00:27:93:84:cf > 08:00:27:c3:fc:bc, ethertype 802.1Q (0x8100), length 102: vlan 100, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 31902, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.254: ICMP echo reply, id 23202, seq 2, length 64
18:37:40.666895 08:00:27:c3:fc:bc > 08:00:27:93:84:cf, ethertype 802.1Q (0x8100), length 102: vlan 100, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 43762, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.254 > 10.10.10.1: ICMP echo request, id 23202, seq 3, length 64
18:37:40.666985 08:00:27:93:84:cf > 08:00:27:c3:fc:bc, ethertype 802.1Q (0x8100), length 102: vlan 100, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 32111, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.254: ICMP echo reply, id 23202, seq 3, length 64
18:37:41.669854 08:00:27:c3:fc:bc > 08:00:27:93:84:cf, ethertype 802.1Q (0x8100), length 102: vlan 100, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 44416, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.254 > 10.10.10.1: ICMP echo request, id 23202, seq 4, length 64
18:37:41.669928 08:00:27:93:84:cf > 08:00:27:c3:fc:bc, ethertype 802.1Q (0x8100), length 102: vlan 100, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 32706, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.254: ICMP echo reply, id 23202, seq 4, length 64
18:37:43.671228 08:00:27:93:84:cf > 08:00:27:c3:fc:bc, ethertype 802.1Q (0x8100), length 46: vlan 100, p 0, ethertype ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.10.10.254 tell 10.10.10.1, length 28
18:37:43.672540 08:00:27:c3:fc:bc > 08:00:27:93:84:cf, ethertype 802.1Q (0x8100), length 64: vlan 100, p 0, ethertype ARP, Ethernet (len 6), IPv4 (len 4), Reply 10.10.10.254 is-at 08:00:27:c3:fc:bc, length 46
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[root@testServer1 ~]#</pre>

<pre>[root@<b>testServer2</b> ~]# tcpdump -nvvv -e -i eth1
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
[root@testServer2 ~]#</pre>

<p>Как мы наблюдаем, icmp пакеты с сервера testClient1 шли именно на сервер testServer1, то есть по vlan100.</p>

<p>Теперь запустим ping на сервере testClient2 до 10.10.10.1, чтобы понять, на какой сервер (testServer1 или testServer2) пойдут пакеты:</p>

<pre>[root@<b>testClient2</b> ~]# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.409 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.79 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=1.19 ms
64 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=1.49 ms
^C
--- 10.10.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 0.409/1.221/1.790/0.516 ms
[root@testClient2 ~]#</pre>

<p>На серверах testServer1 и testServer2 запустим команды tcpdump:</p>

<pre>[root@<b>testServer1</b> ~]# tcpdump -nvvv -e -i eth1
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
[root@testServer1 ~]#</pre>

<pre>[root@<b>testServer2</b> ~]# tcpdump -nvvv -e -i eth1
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
18:43:52.230819 08:00:27:cf:60:e4 > 08:00:27:58:e0:f3, ethertype 802.1Q (0x8100), length 102: vlan 101, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 64270, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.254 > 10.10.10.1: ICMP echo request, id 23200, seq 1, length 64
18:43:52.230869 08:00:27:58:e0:f3 > 08:00:27:cf:60:e4, ethertype 802.1Q (0x8100), length 102: vlan 101, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 63040, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.254: ICMP echo reply, id 23200, seq 1, length 64
18:43:53.231548 08:00:27:cf:60:e4 > 08:00:27:58:e0:f3, ethertype 802.1Q (0x8100), length 102: vlan 101, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 64451, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.254 > 10.10.10.1: ICMP echo request, id 23200, seq 2, length 64
18:43:53.231631 08:00:27:58:e0:f3 > 08:00:27:cf:60:e4, ethertype 802.1Q (0x8100), length 102: vlan 101, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 63116, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.254: ICMP echo reply, id 23200, seq 2, length 64
18:43:54.233535 08:00:27:cf:60:e4 > 08:00:27:58:e0:f3, ethertype 802.1Q (0x8100), length 102: vlan 101, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 64783, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.254 > 10.10.10.1: ICMP echo request, id 23200, seq 3, length 64
18:43:54.233605 08:00:27:58:e0:f3 > 08:00:27:cf:60:e4, ethertype 802.1Q (0x8100), length 102: vlan 101, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 63498, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.254: ICMP echo reply, id 23200, seq 3, length 64
18:43:55.235338 08:00:27:cf:60:e4 > 08:00:27:58:e0:f3, ethertype 802.1Q (0x8100), length 102: vlan 101, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 65005, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.254 > 10.10.10.1: ICMP echo request, id 23200, seq 4, length 64
18:43:55.235573 08:00:27:58:e0:f3 > 08:00:27:cf:60:e4, ethertype 802.1Q (0x8100), length 102: vlan 101, p 0, ethertype IPv4, (tos 0x0, ttl 64, id 64405, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.254: ICMP echo reply, id 23200, seq 4, length 64
18:43:57.243354 08:00:27:cf:60:e4 > 08:00:27:58:e0:f3, ethertype 802.1Q (0x8100), length 64: vlan 101, p 0, ethertype ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.10.10.1 tell 10.10.10.254, length 46
18:43:57.243463 08:00:27:58:e0:f3 > 08:00:27:cf:60:e4, ethertype 802.1Q (0x8100), length 46: vlan 101, p 0, ethertype ARP, Ethernet (len 6), IPv4 (len 4), Reply 10.10.10.1 is-at 08:00:27:58:e0:f3, length 28
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[root@testServer2 ~]#</pre>

<p>Здесь как мы видим, icmp пакеты с сервера testClient2 уже идут на сервер testServer2, то есть по vlan101.</p>




