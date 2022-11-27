# lab-vagrant-ansible

##Ansible

As an Infrastructure as Code (IaC) tool, Ansible has a similarity to Vagrant. But Ansible is much more powerful and is widely used in production environments to manage baremetal and virtualized hosts running Linux, Unix or Windows, both on-premises and in the cloud.

Ansible works by connecting from a Control Node where Ansible is installed, to Managed Nodes where the configuration is applied. Ansible does not need to be installed in the Managed Nodes, it simply connects to them via SSH for Linux and Unix hosts and Windows Remote Management (WinRM) for Windows hosts. The only requirements are Python in Linux and Unix hosts and PowerShell in Windows hosts.

## Vagrant avec Ansible

Vagrant supports several provisioners including Ansible. There are two different Ansible provisioners in Vagrant: Ansible and Ansible Local. The Ansible provisioner runs Ansible from your guest, while Ansible Local installs Ansible in a VM provisioned by Vagrant (Control Node) and uses it to configure other VMs (Managed Nodes). Since Ansible cannot run on Windows and I want to keep the requisites in your guest machine limited to Vagrant and VirtualBox, we’re going to use Ansible Local.

![image](https://user-images.githubusercontent.com/45850849/204154464-a5cb99c5-a86a-49da-93ef-5bffe41f081f.png)

Vagrantfile

Choose an empty directory and create the following Vagrantfile:
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
	
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

First we define a load balancer (lb) node and connect it to private_network with an IP address. We also forward port 8080 in our host machine to port 80 in the VM, so we can access it through our browser.

Then we define two web nodes (node1 and node2) and join them to private_network with an IP address. These nodes have no port forward so they are not accessible through our browser.

Finally we define the Ansible controller (controller) that is going to be used by Vagrant to configure the other nodes. We join it to private_network with an IP. We use the ansible_local provisioner as discussed before, indicating that we want to run the playbook on all hosts (ansible.limit = "all") and indicate the path to the playbook, inventory and ansible.cfg files. Finally we override the default configuration for the synced_folder, using a umask to remove permissions from all users except vagrant. This is necessary otherwise both Ansible and ssh will complain for security reasons and fail.
Ansible configuration

Create a provisioning directory where we’ll place all Ansible related files. By default Vagrant autogenerates an inventory that is placed in the guest VM under the path /tmp/vagrant-ansible/inventory/vagrant_ansible_local_inventory, but since we have no name resolution we cannot use it. Instead create a hosts file under the provisioning directory:

controller ansible_connection=local
 lb         ansible_host=172.17.177.21 ansible_ssh_private_key_file=/vagrant/.vagrant/machines/lb/virtualbox/private_key
 node1      ansible_host=172.17.177.22 ansible_ssh_private_key_file=/vagrant/.vagrant/machines/node1/virtualbox/private_key
 node2      ansible_host=172.17.177.23 ansible_ssh_private_key_file=/vagrant/.vagrant/machines/node2/virtualbox/private_key
 [nginx]
 lb
 node[1:2]

This file is telling Ansible how to connect to the hosts. It lists the IP addresses that we defined in Vagrantfile and the private keys to connect to every host. Vagrant places the private keys under .vagrant/machines/<machine name>/virtualbox/private_key paths. We also define an nginx group which consists of the load balancer and both web nodes.

The next file to create is the Ansible Playbook (playbook.yml) which tells Ansible which tasks to execute in which hosts:

---
- hosts: nginx
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
      become: yes

- hosts: node1
  tasks:
    - name: Copy hello from node 1
      ansible.builtin.copy:
        dest: /var/www/html/index.html
        content: 'Hello from Node 1!'
      become: yes

- hosts: node2
  tasks:
    - name: Copy hello from node 2
      ansible.builtin.copy:
        dest: /var/www/html/index.html
        content: 'Hello from Node 2!'
      become: yes

- hosts: lb
  tasks:
    - name: Copy nginx.conf to load balancer
      ansible.builtin.copy:
        src: nginx.conf
        dest: /etc/nginx/sites-enabled/default
      become: yes
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
      become: yes
      

To keep it simple we go in a linear fashion: first use apt to install nginx in the nginx group from the inventory (lb, node1 and node2). Then we copy a welcome message to node1 and node2 (/var/www/html/index.html). Then override the nginx default configuration in the load balancer (/etc/nginx/sites-enabled/default) with the content of nginx.conf, and restart the service to load the new configuration.

Next, inside the provisioning directory, create a files directory and create the nginx.conf file inside of it:

upstream hello {
    server 172.17.177.22;
    server 172.17.177.23;
}

server {
    listen 80;

    location / {
        proxy_pass http://hello;
    }
}

This file configures nginx in the lb node as a load balancer for the two web nodes. It defaults to round robin.

Finally we’ll create the ansible.cfg file inside the provisioning directory to allow ssh to connect to the controlled nodes:

[defaults]
host_key_checking = no

You should have a directory structure like this:

It’s time to start it! Open a terminal where Vagrantfile is placed and enter:

vagrant up

Wait while Vagrant creates 4 virtual machines, installs Ansible in the controller node and runs the playbook to configure the load balancer and both web nodes.

Now go to http://localhost:8080 and you will see the welcome message from node1:

Reload the page several times and you will see the message change as the load balancer forwards the requests to node1 and node2 alternatively.
Wrapping up

Enter vagrant halt to stop the VMs and save some resources, or vagrant destroy -f to delete them, concluding this demo.

Vagrant is a very nice way to test with virtual machines. It can create a single VM or several VMs connected by virtual networks.

Integration with Ansible allows to test Ansible playbooks in your machine. It also supports other provisioners like Chef, Puppet, Docker and more, enabling the development of complex setups in a virtual environment, without the need for real servers.
