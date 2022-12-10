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

Créez un répertoire de provisionnement dans lequel nous placerons tous les fichiers liés à Ansible. 

Créons un fichier hosts sous le répertoire de provisioning :

```
 controller ansible_connection=local
 lb         ansible_host=172.17.177.21 ansible_ssh_private_key_file=/vagrant/.vagrant/machines/lb/virtualbox/private_key
 node1      ansible_host=172.17.177.22 ansible_ssh_private_key_file=/vagrant/.vagrant/machines/node1/virtualbox/private_key
 node2      ansible_host=172.17.177.23 ansible_ssh_private_key_file=/vagrant/.vagrant/machines/node2/virtualbox/private_key
 
 [nginx]
 lb
 node[1:2]
```

Ce fichier indique à Ansible comment se connecter aux hôtes. 
  * Il répertorie les adresses IP que nous avons définies dans Vagrantfile et les clés privées pour se connecter à chaque hôte. 
  * Vagrant place les clés privées sous les chemins .vagrant/machines/<nom de la machine>/virtualbox/private_key. 
  * Nous définissons également un groupe nginx composé de l'équilibreur de charge et des deux nœuds Web.


Le prochain fichier à créer est le Playbook Ansible (playbook.yml) qui indique à Ansible quelles tâches exécuter dans quels hôtes :

```
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

```

Pour faire simple, nous procédons de manière linéaire : 
 - utilisez d'abord apt pour installer nginx dans le groupe nginx à partir de l'inventaire (lb, node1 et node2). 
 - Ensuite, nous copions un message de bienvenue sur node1 et node2 (/var/www/html/index.html). 
 - Remplacez ensuite la configuration par défaut de nginx dans l'équilibreur de charge (/etc/nginx/sites-enabled/default) avec le contenu de nginx.conf, et redémarrez le service pour charger la nouvelle configuration.

Ensuite, dans le répertoire de provisioning, créez un répertoire de fichiers et créez le fichier nginx.conf à l'intérieur :
 
```
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
```

Ce fichier configure nginx dans le nœud lb en tant qu'équilibreur de charge pour les deux nœuds Web (Il s'agit par défaut d'un round robin)

Enfin, nous allons créer le fichier ansible.cfg dans le répertoire de provisioning pour permettre à ssh de se connecter aux nœuds contrôlés :

```
 [defaults]
host_key_checking = no
```
 
Vous devriez avoir une structure de répertoire comme celle-ci :


```
vagrant
-- Vagrantfile
-- provisioning/
  -- ansible.cfg
  -- hosts
  -- playbook.yml
  -- files/
    -- nginx.conf
```

Il est temps de commencer ! Ouvrez un terminal où Vagrantfile est placé et entrez :

```
 vagrant up
```
 
Attendez que Vagrant crée 4 machines virtuelles, installe Ansible dans le nœud du contrôleur et exécute le playbook pour configurer l'équilibreur de charge et les deux nœuds Web.

Allez maintenant sur http://localhost:8080 et vous verrez le message de bienvenue de node1 :

Rechargez la page plusieurs fois et vous verrez le message changer au fur et à mesure que l'équilibreur de charge transmet les requêtes au nœud1 et au nœud2 alternativement.

source: https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwiDsZyB8en7AhVZSKQEHVDtBFcQFnoECA0QAQ&url=https%3A%2F%2Forlando-ramirez.com%2F2020%2F12%2F06%2Fprovisioning-virtual-machines-with-ansible-and-vagrant%2F&usg=AOvVaw2UuqBybzLceipulTAruij-
