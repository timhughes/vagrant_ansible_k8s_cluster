# -*- mode: ruby -*-
# vi: set ft=ruby :

node_configs = [
  {role: 'control', qty: 3},
  {role: 'worker', qty: 3},
]

# Max instances per role. Used for spacing ip addresses
role_spacer = 10


Vagrant.configure("2") do |config|

  config.vm.box = "centos/7"

  # Deliberatly disabled as it is not needed. Change `disabled: true`
  # to `disabled: false` if you would like to use NFS shares or you can
  # change to some other vagrant share system.
  config.vm.synced_folder ".",
    "/vagrant",
    type: "nfs",
    nfs_udp: false,
    disabled: true

  config.vm.provider :libvirt do |libvirt|
    libvirt.cpus = 2
    libvirt.memory = 4096
    libvirt.nested = true
    libvirt.graphics_type = 'spice'
    libvirt.video_type = 'virtio'
    # Use QEMU system instead of session connection
    libvirt.qemu_use_session = false
    libvirt.suspend_mode = 'managedsave'
  end

  ansible_groups = {}


  config.vm.define :lb, autostart: false do |node|
    #node.vm.box = "centos/7"
    node.vm.network :private_network,
      :libvirt__network_name => 'kube-internal',
      :libvirt__dhcp_enabled => false,
      :ip => "192.168.200.2"
    node.vm.provider :libvirt do |libvirt|
      libvirt.cpus = 2
      libvirt.memory = 2048
    end
    node.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/lb.yml"
    end
  end




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
        node.vm.network "private_network",
          :libvirt__network_name => 'kube-internal',
          ip: "192.168.200.#{id}"
        node.vm.provision "shell", inline: 'sed -i "/$HOSTNAME/d" /etc/hosts'
        if node_conf[:role].eql?('worker')
          node.vm.provider :libvirt do |libvirt|
            libvirt.storage :file, :size => '40G'
          end
        end

        node.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/playbook.yml"
          ansible.groups = ansible_groups
        end
      end
    end
  end
end
