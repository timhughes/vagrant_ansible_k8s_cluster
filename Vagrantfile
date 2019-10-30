# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  config.vm.box = "centos/7"
  config.vm.synced_folder ".", "/vagrant", type: "nfs", nfs_udp: false

  config.vm.provider :libvirt do |libvirt|
    libvirt.cpus = 2
    libvirt.memory = 2048
    libvirt.nested = true
    libvirt.graphics_type = 'spice'
    libvirt.video_type = 'virtio'

  end
  node_configs = [
    {role: 'master', qty: 1},
    {role: 'worker', qty: 3},
  ]


  ansible_groups = {}

  #config.vm.network "private_network",
  #    type: 'dhcp',
  #    libvirt__network_name: 'metallb',
  #    libvirt__network_address: '172.28.128.0',
  #    libvirt__dhcp_last: '172.28.128.100' # keep the rest for the load balancer

  j = 10
  node_configs.each do |node_conf|
    (1..(node_conf[:qty])).each do |i|
      id = i.to_s.rjust(2, '0')
      config.vm.define "#{node_conf[:role]}-#{id}" do |node|
        hostname = "#{node_conf[:role]}-#{id}"
        if ansible_groups.key?(node_conf[:role])
          ansible_groups[node_conf[:role]] << hostname
        else
          ansible_groups[node_conf[:role]] = [hostname]
        end
        node.vm.hostname = hostname
        node.vm.network "private_network", ip: "192.168.200.#{j+id}"
        node.vm.provision "shell", inline: 'sed -i "/$HOSTNAME/d" /etc/hosts'
      end
    end
    j += 10
  end
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/playbook.yml"
    ansible.groups = ansible_groups
  end


end
