# -*- mode: ruby -*-
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
end

