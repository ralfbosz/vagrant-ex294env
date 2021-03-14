# number of worker nodes
NUM_WORKERS = 4
# number of extra disks per worker
NUM_DISKS = 1
# size of each disk in gigabytes
DISK_GBS = 10

MASTER_IP = "192.168.55.200"
WORKER_IP_BASE = "192.168.55.2" # 200, 201, ...

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false
  config.vm.box_check_update = false
  config.vm.box = "generic/centos8"
  
  config.vm.provider :libvirt do |libvirt|
    libvirt.cpu_mode = 'host-passthrough'
    libvirt.graphics_type = 'none'
    libvirt.memory = 1024
    libvirt.cpus = 1
    libvirt.qemu_use_session = false
  end

  config.vm.define "repo" do |repo|
    repo.vm.hostname = "repo.ansi.example.com"
    repo.vm.provision :shell, :inline => "sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config; sudo systemctl restart sshd;", run: "always"
    repo.vm.provision :shell, :inline => "dnf install epel-release -y; sudo dnf install -y sshpass httpd sshpass vsftpd createrepo", run: "always"
    repo.vm.provision :shell, :inline => " dnf install ansible -y", run: "always"
    repo.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: [".git/", "*.vdi"]
    repo.vm.network "private_network", ip: "192.168.55.199"
  
    repo.vm.provider :libvirt do |libvirt|
      libvirt.memory = "512"
    end
  end

  (1..NUM_WORKERS).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}"
      node.vm.network :private_network, ip: "#{WORKER_IP_BASE}" + i.to_s.rjust(2, '0')
      (1..NUM_DISKS).each do |j|
        node.vm.provider :libvirt do |libvirt|
          libvirt.cpus=1
          if i == 3
            libvirt.memory=512
            libvirt.storage :file, :size => "#{DISK_GBS}G"
          elsif i >= 4
            libvirt.memory=512
          end
        end
      end
    end
  end

  config.vm.define "control" do |control|
    control.vm.hostname = "control"
    config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__exclude: [".git/", "*.vdi"]
    control.vm.network :private_network, ip: MASTER_IP
    config.vm.provider :libvirt do |libvirt|
      libvirt.memory = 2048
      libvirt.cpus = 1
    end
    control.vm.provision :shell, :inline => "dnf install epel-release -y"
    control.vm.provision :shell, :inline => "dnf install ansible -y"
    control.vm.provision :shell, :inline => 'ssh-keygen -f /home/vagrant/.ssh/id_rsa -N ""'
    control.vm.provision :shell, :inline => "chown -R vagrant: /home/vagrant/.ssh"
    control.vm.provision :shell, :inline => "cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys"
    control.vm.provision :ansible_local do |ansible|
      ansible.playbook = "/vagrant/playbooks/master.yml"
      ansible.install = false
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "/vagrant/inventory"
      ansible.config_file = "/vagrant/ansible.cfg"
      ansible.limit = "all"
    end
  end
end