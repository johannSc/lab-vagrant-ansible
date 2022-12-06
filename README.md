# lab-vagrant-ansible

## Ansible

En tant qu'outil IaC, Ansible présente une similitude avec Vagrant. Mais Ansible est beaucoup plus puissant et est largement utilisé dans les environnements de production pour gérer les hôtes de type bare-metal, exécutant Linux / Unix / Windows, à la fois en local mais aussi dans le cloud.

Ansible fonctionne en se connectant à partir d'un nœud de contrôle où il est installé, à des nœuds gérés où la configuration est appliquée. 
Ansible n'a pas besoin d'être installé dans les nœuds gérés, il s'y connecte simplement via SSH pour les hôtes Linux et Unix et la gestion à distance Windows (WinRM) pour les hôtes Windows. Les seules exigences sont Python dans les hôtes Linux et Unix et PowerShell dans les hôtes Windows.

## Vagrant avec Ansible

Vagrant prend en charge plusieurs fournisseurs, dont Ansible. Il existe deux approvisionneurs Ansible différents dans Vagrant : Ansible et Ansible Local. 

  * L'approvisionneur Ansible exécute Ansible à partir de votre invité
  * Ansible Local installe Ansible dans une machine virtuelle provisionnée par Vagrant (nœud de contrôle) et l'utilise pour configurer d'autres machines virtuelles (nœuds gérés). 
  
  => Étant donné qu'Ansible ne peut pas s'exécuter sur Windows et que je souhaite que les éléments requis sur votre machine invitée soient limités à Vagrant et VirtualBox, nous allons utiliser Ansible Local.

![image](https://user-images.githubusercontent.com/45850849/204154464-a5cb99c5-a86a-49da-93ef-5bffe41f081f.png)


### Vagrantfile

```
Vagrant.configure("2") do |config|
    config.vm.box = "generic/debian10"
 
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

```

  * Nous définissons d'abord une équilibre de charge (lb= load-balancer) et le connectons à private_network avec une adresse IP. Nous 'nattons' également le port 8080 de notre machine hôte vers le port 80 de la machine virtuelle, afin que nous puissions y accéder via notre navigateur.

  * Ensuite, nous définissons deux nœuds Web (node1 et node2) et les joignons à private_network avec une adresse IP. Ces nœuds n'ont pas de transfert de port, ils ne sont donc pas accessibles via notre navigateur.

  * Puis nous définissons le contrôleur Ansible (controller) qui va être utilisé par Vagrant pour configurer les autres nœuds. Nous le joignons à private_network avec une adresse IP. Nous utilisons le fournisseur ansible_local comme indiqué précédemment, indiquant que nous voulons exécuter le playbook sur tous les hôtes (ansible.limit = "all") et indiquons le chemin vers les fichiers playbook, inventaire et ansible.cfg. 
  
  * Enfin, nous remplaçons la configuration par défaut pour le synced_folder, en utilisant un umask pour supprimer les autorisations de tous les utilisateurs sauf vagrant. Ceci est nécessaire sinon Ansible et ssh se plaindront pour des raisons de sécurité et échoueront.
Configuration possible

Créez un répertoire de provisionnement dans lequel nous placerons tous les fichiers liés à Ansible. Par défaut, Vagrant génère automatiquement un inventaire qui est placé dans la machine virtuelle invitée sous le chemin /tmp/vagrant-ansible/inventory/vagrant_ansible_local_inventory, mais comme nous n'avons pas de résolution de nom, nous ne pouvons pas l'utiliser. Créez plutôt un fichier hosts sous le répertoire de provisioning :

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
