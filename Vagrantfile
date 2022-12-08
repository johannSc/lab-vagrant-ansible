Vagrant.configure("2") do |config|
    config.vm.box = "bento/ubuntu-18.04"
 
    config.vm.define "lb" do |machine|
        machine.vm.network "private_network", ip: "172.17.177.21"
        machine.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"
    end
 
    config.vm.define "node1" do |machine|
        machine.vm.network "private_network", ip: "172.17.177.22"
    end
 
    config.vm.define "node2" do |machine|
        machine.vm.network "private_network", ip: "172.17.177.23"
    end
 
    config.vm.define "controller" do |machine|
        machine.vm.network "private_network", ip: "172.17.177.11"
 
        machine.vm.provision "ansible_local" do |ansible|
            ansible.playbook = "provisioning/playbook.yml"
            ansible.limit = "all"
            ansible.inventory_path = "provisioning/hosts"
            ansible.config_file = "provisioning/ansible.cfg"
        end
 
        machine.vm.synced_folder ".", "/vagrant", mount_options: [ "umask=077" ]
    end
end
