<h3>### VLAN ###</h3>

<p>Строим бонды и вланы</p>

<h4>Описание домашнего задания</h4>

<p>в Office1 в тестовой подсети появляется сервера с доп интерфесами и адресами в internal сети testLAN</p>
<ul type="disc">
<li>testClient1 - 10.10.10.254</li>
<li>testClient2 - 10.10.10.254</li>
<li>testServer1- 10.10.10.1</li>
<li>testServer2- 10.10.10.1<br />
равести вланами<br />
testClient1 <-> testServer1<br />
testClient2 <-> testServer2<br />
между centralRouter и inetRouter<br />
"пробросить" 2 линка (общая inernal сеть) и объединить их в бонд<br />
проверить работу c отключением интерфейсов</p>

<p>Формат сдачи ДЗ - vagrant + ansible</p>













<h4>1. Работа со стендом и настройка DNS</h4>

<p>В домашней директории создадим директорию dns, в котором будут храниться настройки виртуальных машин:</p>

<pre>[user@localhost otus]$ mkdir ./dns
[user@localhost otus]$</pre>

<p>Перейдём в директорию dns:</p>

<pre>[user@localhost otus]$ cd ./dns/
[user@localhost dns]$</pre>

<p>Скачаем стенд https://github.com/erlong15/vagrant-bind и изучим содержимое файлов:</p>

<pre>[user@localhost dns]$ ls -l
total 12
drwxrwxr-x. 2 user user 4096 фев 16  2020 provisioning
-rw-rw-r--. 1 user user 3100 сен 29 00:00 README.md
-rw-rw-r--. 1 user user  820 фев 16  2020 Vagrantfile
[user@localhost dns]$</pre>


<p>Откроем Vagrantfile, добавим ВМ client2 и внесём некоторые корректировки, такие как вместо <i>ansible.sudo = "true"</i> запишем <i>ansible.<b>become</b> = "true"</i>:</p>

<pre>[user@localhost dns]$ vi ./Vagrantfile</pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"

  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "vvv"
    ansible.playbook = "provisioning/playbook.yml"
    ansible.<b>become</b> = "true"
  end


  config.vm.provider "virtualbox" do |v|
	  v.memory = 256
  end

  config.vm.define "ns01" do |ns01|
    ns01.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "dns"
    ns01.vm.hostname = "ns01"
  end

  config.vm.define "ns02" do |ns02|
    ns02.vm.network "private_network", ip: "192.168.50.11", virtualbox__intnet: "dns"
    ns02.vm.hostname = "ns02"
  end

  config.vm.define "client" do |client|
    client.vm.network "private_network", ip: "192.168.50.15", virtualbox__intnet: "dns"
    client.vm.hostname = "client"
  end

  <b>config.vm.define "client2" do |client2|
    client2.vm.network "private_network", ip: "192.168.50.16", virtualbox__intnet: "dns"
    client2.vm.hostname = "client2"
  end</b>

end</pre>

<p>После того, как добавили виртуальную машину client2, разберём содержимое каталога provisioning:</p>

<pre>ls -l ./provisioning</pre>