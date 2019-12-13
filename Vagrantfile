# -*- mode: ruby -*-
# vi: set ft=ruby :


node_configs = [
  {role: 'master', qty: 1 },
  {role: 'worker', qty: 3},
]

# Max instances per role. Used for spacing ip addresses
role_spacer = 10


Vagrant.configure("2") do |config|

  config.vm.box = "centos/7"
  config.vm.synced_folder ".",
    "/vagrant",
    type: "nfs",
    nfs_udp: false

  config.vm.provider :libvirt do |libvirt|
    libvirt.cpus = 2
    libvirt.memory = 4096
    libvirt.nested = true
    libvirt.graphics_type = 'spice'
    libvirt.video_type = 'virtio'
  end

  ansible_groups = {}

  node_configs.each_with_index  do |node_conf, i|
    role_num = i+1 # dont want multiply by zero later
    if node_conf[:qty] > role_spacer
      puts "ERROR: Role #{node_conf[:role]} has 'qty' of  #{node_conf[:qty]} greater than #{role_spacer} (role_spacer)"
      abort
    end
    (1..(node_conf[:qty])).each do |node_num|
      id = role_spacer*role_num+node_num
      config.vm.define "#{node_conf[:role]}-#{id}" do |node|
        hostname = "#{node_conf[:role]}-#{id}"
        if ansible_groups.key?(node_conf[:role])
          ansible_groups[node_conf[:role]] << hostname
        else
          ansible_groups[node_conf[:role]] = [hostname]
        end
        node.vm.hostname = hostname
        node.vm.network "private_network", ip: "192.168.200.#{id}", libvirt__forward_mode: 'route'
        node.vm.provision "shell", inline: 'sed -i "/$HOSTNAME/d" /etc/hosts'
      end
    end
  end
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/playbook.yml"
    ansible.groups = ansible_groups
  end
end
