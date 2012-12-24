.. toctree::
    :maxdepth:  2

-------------------------------------------------------
Guide d'installation d'OpenStack Folsom sur une machine
-------------------------------------------------------

:Version: 1.0
:Source: https://github.com/obuisson/Guide-Installation-OpenStack-Folsom-Noeud-Unique
:Keywords: OpenStack, Folsom, Nova, Keystone, Glance, Horizon, Cinder, KVM, Ubuntu

Préambule
=========

L'objectif de ce guide est de montrer les différentes étapes nécessaire à l'installation d'OpenStack Folsom sur une machine unique. Afin de simplifier l'installation, le guide utilise un script permettant la création des différents services, utilisateurs, etc... pour Keystone. 


Audience attendue
-----------------

Ce guide s'adresse à des administrateurs systèmes souhaitant installer, configurer et opérer une installation d'OpenStack Folsom. Il est donc nécessaire d'avoir des compétences dans les domaines suivant :

* Ligne de commande UNIX
* Installation de paquets
* Connaissance des concepts réseaux de base
* Connaissance des composants d'OpenStack

Inspiration
-----------

Ce guide est largement inspiré par le Guide en anglais nommé "`Basic OpenStack Folsom Install Guide <http://openstack-folsom-install-guide.readthedocs.org/en/latest/>`" de Zachary VanDuyn

Auteur
------

`Olivier Buisson <http://obn.me/>`_


Démarrage
=========

Ce guide utilise volontairement `nova-network` au lieu de `quantum` afin de le simplifier. En effet, la mise en place de Quantum complexifie beaucoup le temps de mise en place. De plus, les fonctionnalités avancées de Quantum ne sont pas vraiment utile lors de l’installation sur un seul serveur.

Configuration Matériel
----------------------

OpenStack a la capacité de s'installer sur une multiltude de machine physique. De l'ordinateur portable au serveurs d'entreprise, tout est possible. Dans ce guide, l'installation se fera sur un serveur ayant la configuration suivante : 

* **Processeur**: Intel(R) Xeon(R) CPU E31220 @ 3.10GHz 64-bit
* **Mémoire**: 16GB de RAM
* **Disque**: 2x 2 To SATA2 7k2 RAID 1
* **Réseaux** : 2 cartes réseaux 1GB

La configuration réseau est la suivante : 

* **eth0**: Interface réseau public : 88.190.22.60
* **eth1**: Interface réseau privé : 10.32.14.232

Installation d'OpenStack
========================

Mise à jour du système
----------------------

* Après avoir terminé l'installation de la distribution Ubuntu 12.04 LTS, passer en tant qu'utilisateur root jusqu'à la fin du guide::

    sudo su

* Mettre à jour le système::

    apt-get update
    apt-get upgrade
    apt-get dist-upgrade


Ajout du repository Ubuntu Cloud Archive
----------------------------------------

La version 12.04 d'Ubuntu ne contient pas les paquets OpenStack Folsom par défaut. Canonical fournit un repository où se trouve ces paquets.

* Ajout du repository::

    echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main >> /etc/apt/sources.list.d/folsom.list
    apt-get install ubuntu-cloud-keyring 
    apt-get update
    apt-get upgrade

Installation et Configuration de MySQL et RabbitMQ
--------------------------------------------------

* Installation de MySQL::

    apt-get install mysql-server python-mysqldb

* Configuration de MySQL pour accepter les requêtes depuis toutes les interfaces::

    sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
    servier mysql restart

* Installation de RabbitMQ::

    apt-get install rabbitmq-server

Installation et configuration du serveur NTP
--------------------------------------------
* Installation du service NTP::

    apt-get install ntp

* Configuration du serveur NTP::

    sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
    service ntp restart

Installation des services de VLAN et de bridges et configuration de l'IP Forwarding
------------------------------------------------------------------------------------

* Installation des services VLAN et Bridges::

    apt-get install vlan bridge-utils

* Activation de l'IP Forwarding en décommentant net.ipv4.ip_forward=1 dans /etc/sysctl.conf::

    sed -i 's/\#net\.ipv4\.ip_forward/net\.ipv4\.ip_forward/g' /etc/sysctl.conf 

* Prise en compte de la nouvelle configuration::

    sysctl -p

Configuration du réseau
-----------------------

* Configuration des interfaces réseaux via /etc/network/interfaces::

   vi /etc/network/interfaces

   # This file describes the network interfaces available on your system
   # and how to activate them. For more information, see interfaces(5).
   # The loopback network interface
   auto lo
   iface lo inet loopback
   # The primary network interface
   auto eth0
   iface eth0 inet static
           address 88.190.22.60
           netmask 255.255.255.0
           network 88.190.22.0
          broadcast 88.190.22.255
          gateway 88.190.22.1
   auto br100
   iface br100 inet static
           address 10.32.14.232
           netmask 255.255.255.0
           network 10.32.14.0
           broadcast 10.32.14.255
           # dns-* options are implemented by the resolvconf package, if installed
           dns-nameservers 8.8.8.8
           dns-search obn.me
           bridge_ports eth1
           bridge_stp off
           bridge_maxwait 0
           bridge_fd 0

* Création du bridge et relance du réseau::

   brctl addbr br100; /etc/init.d/networking restart


Installation et configuration de KVM
------------------------------------

* Vérification si votre machine supporte la virtualisation avec KVM::

   apt-get install cpu-checker

* Lancer la commande suivante::

   kvm-ok

* Vous devriez avoir une réponse similaire à ça::

   KVM acceleratiopn can be used

* Nous pouvons installer KVM et le configurer::

   apt-get install -y kvm libvirt-bin pm-utils

* Modifier le tableau cgroup_device_acl dans le fichier qemu.conf::

   vi /etc/libvirt/qemu.conf

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet", "/dev/net/tun"
   ]

* Suppression du bridge par défaut::

   virsh net-destroy default
   virsh net-undefine default

Configuration du live migration
-------------------------------

* Activation du live migration en enlevant les commentaires des variables listen_tls = 0, listen_tcp = 1 et auth_tcp = "none" dans le fichier libvirtd.conf. Ne pas toucher aux autres paramètres::

   vi /etc/libvirt/libvirtd.conf

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"

* Editer la variable libvirtd_opts dans le fichier libvirt-bin.conf::

   vi /etc/init/libvirt-bin.conf

   env libvirtd_opts="-d -l"

* Editer la même variable dans le fichier /etc/default/libvirt-bin::

   libvirtd_opts="-d -l"

* Relancer le service libvirt pour prendre en compte les changements::

   service libvirt-bin restart


Installation et configuration de Keystone
-----------------------------------------

* Installation du service d'identité Keystone::

    apt-get install keystone

* Création de la base de donnée MySQL pour Keystone::

    mysql -u root -p
    CREATE DATABASE keystone;
    GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
    quit;

* Modification des attributs de connexion dans /etc/keystone/keystone.conf pour notre base::

    vi /etc/keystone/keystone.conf
    #Modifier le champ connection comme suit
    connection = mysql://keystoneUser:keystonePass@10.32.14.232/keystone

* Relance du service et synchronisation de la base::

    service keystone restart
    keystone-manage db_sync

* Utilisation du script de mseknibilel::

   wget https://raw.github.com/nimbula/OpenStack-Folsom-Install-guide/VLAN/2NICs/Keystone_Scripts/keystone_basic.sh
   wget https://raw.github.com/nimbula/OpenStack-Folsom-Install-guide/VLAN/2NICs/Keystone_Scripts/keystone_endpoints_basic.sh
   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

* Dans le script keystone_basic.sh, modifier la variable $HOST_IP par votre adresse IP
* Dans le script keystone_endpoints_basic.sh, modifier les variables $HOST_IP, $EXT_HOST_IP, & $MYSQL_HOST par votre addresse IP et exécuter les scripts.

* **Note: Vérifier bien les modifications car nettoyer Keystone peut être long et douloureux à faire**::

   vi keystone_basic.sh
   vi keystone_endpoints_basic.sh
   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh


* Le script keystone_basic.sh n'affiche rien, mais le script keystone_endpoints_basic.sh doit vous renvoyer quelque chose comme ça::

   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |    OpenStack Compute Service     |
   |      id     | 2801693507a44570a7439245b20ea0cd |
   |     name    |               nova               |
   |     type    |             compute              |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |     OpenStack Volume Service     |
   |      id     | b80f524c06464c0c8af80942a1c94f78 |
   |     name    |              cinder              |
   |     type    |              volume              |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |     OpenStack Image Service      |
   |      id     | 9326c1e4d4bc4e748bd8387fa5279bd0 |
   |     name    |              glance              |
   |     type    |              image               |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |        OpenStack Identity        |
   |      id     | 7fd27d54ac7c476cb36ef7d0002b9fda |
   |     name    |             keystone             |
   |     type    |             identity             |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |      OpenStack EC2 service       |
   |      id     | 7ce8ae8b16774c3f82e0eeecea60520a |
   |     name    |               ec2                |
   |     type    |               ec2                |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   | description |   OpenStack Networking service   |
   |      id     | 8777783c2f9f4ae3a3a6a501833ab021 |
   |     name    |             quantum              |
   |     type    |             network              |
   +-------------+----------------------------------+
   +-------------+-------------------------------------------+
   |   Property  |                   Value                   |
   +-------------+-------------------------------------------+
   |   adminurl  | http://10.32.14.232:8774/v2/$(tenant_id)s |
   |      id     |      ecfcff81220c45ce9f13ca000f1c4fa7     |
   | internalurl | http://10.32.14.232:8774/v2/$(tenant_id)s |
   |  publicurl  | http://10.32.14.232:8774/v2/$(tenant_id)s |
   |    region   |                 RegionOne                 |
   |  service_id |      2801693507a44570a7439245b20ea0cd     |
   +-------------+-------------------------------------------+
   +-------------+-------------------------------------------+
   |   Property  |                   Value                   |
   +-------------+-------------------------------------------+
   |   adminurl  | http://10.32.14.232:8776/v1/$(tenant_id)s |
   |      id     |      420959377cde408a865445b0ea743a19     |
   | internalurl | http://10.32.14.232:8776/v1/$(tenant_id)s |
   |  publicurl  | http://10.32.14.232:8776/v1/$(tenant_id)s |
   |    region   |                 RegionOne                 |
   |  service_id |      b80f524c06464c0c8af80942a1c94f78     |
   +-------------+-------------------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |   http://10.32.14.232:9292/v2    |
   |      id     | f2c0b4ea7bed4a8aa2d44b140df73a0d |
   | internalurl |   http://10.32.14.232:9292/v2    |
   |  publicurl  |   http://10.32.14.232:9292/v2    |
   |    region   |            RegionOne             |
   |  service_id | 9326c1e4d4bc4e748bd8387fa5279bd0 |
   +-------------+----------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |  http://10.32.14.232:35357/v2.0  |
   |      id     | ef0fb3dfa5f74f70a2059dd015e7743d |
   | internalurl |  http://10.32.14.232:5000/v2.0   |
   |  publicurl  |  http://10.32.14.232:5000/v2.0   |
   |    region   |            RegionOne             |
   |  service_id | 7fd27d54ac7c476cb36ef7d0002b9fda |
   +-------------+----------------------------------+
   +-------------+-----------------------------------------+
   |   Property  |                  Value                  |
   +-------------+-----------------------------------------+
   |   adminurl  | http://10.32.14.232:8773/services/Admin |
   |      id     |     e5a40371df6e47e79dc78bb61591fc87    |
   | internalurl | http://10.32.14.232:8773/services/Cloud |
   |  publicurl  | http://10.32.14.232:8773/services/Cloud |
   |    region   |                RegionOne                |
   |  service_id |     7ce8ae8b16774c3f82e0eeecea60520a    |
   +-------------+-----------------------------------------+
   +-------------+----------------------------------+
   |   Property  |              Value               |
   +-------------+----------------------------------+
   |   adminurl  |    http://10.32.14.232:9696/     |
   |      id     | 8396e30ecbf14f3d9bd97d489f7407ea |
   | internalurl |    http://10.32.14.232:9696/     |
   |  publicurl  |    http://10.32.14.232:9696/     |
   |    region   |            RegionOne             |
   |  service_id | 8777783c2f9f4ae3a3a6a501833ab021 |
   +-------------+----------------------------------+

* Création du fichier contenant les identifiants de connexions::

   vi creds

* Copier le texte suivant::

   export OS_NO_CACHE=1
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://10.32.14.232:5000/v2.0/"

* Charger le fichier::

   source creds

* Vérifions si Keystone fonctionne avec un test simple::

   apt-get install curl openssl
   curl http://10.32.14.232:35357/v2.0/endpoints -H 'x-auth-token: ADMIN' | python -m json.tool

* Vous devriez avoir un retour qui ressemble à ça::

   {
   "endpoints": [
   {
   "adminurl": "http://10.32.14.232:8776/v1/$(tenant_id)s", 
   "id": "0fe6ddf16ce344989adb22a644befa48", 
   "internalurl": "http://10.32.14.232:8776/v1/$(tenant_id)s", 
   "publicurl": "http://10.32.14.232:8776/v1/$(tenant_id)s", 
   "region": "RegionOne", 
   "service_id": "d0a8dbeac60845aaa1fa043c23177d5e"
   }, 
   {
   "adminurl": "http://10.32.14.232:35357/v2.0", 
   "id": "7811cbbf4c3042f1a6b97d19a9ceace5", 
   "internalurl": "http://10.32.14.232:5000/v2.0", 
   "publicurl": "http://10.32.14.232:5000/v2.0", 
   "region": "RegionOne", 
   "service_id": "00685df9e085427a97837892622ca4b2"
   }, 
   {
   "adminurl": "http://10.32.14.232:8774/v2/$(tenant_id)s", 
   "id": "826a7b77f108414ea4be8eb06d3b0c96", 
   "internalurl": "http://10.32.14.232:8774/v2/$(tenant_id)s", 
   "publicurl": "http://10.32.14.232:8774/v2/$(tenant_id)s", 
   "region": "RegionOne", 
   "service_id": "1a7bd347252049d9921703d45c1182dc"
   }, 
   {
   "adminurl": "http://10.32.14.232:9696/", 
   "id": "b0974d6c9bbb4f2cab281f3ff5bcd412", 
   "internalurl": "http://10.32.14.232:9696/", 
   "publicurl": "http://10.32.14.232:9696/", 
   "region": "RegionOne", 
   "service_id": "fc2b6886fd8241448d4f3b0c9a960bf0"
   }, 
   {
   "adminurl": "http://10.32.14.232:9292/v2", 
   "id": "c49d46bc5a62445ea60dc568abc954bb", 
   "internalurl": "http://10.32.14.232:9292/v2", 
   "publicurl": "http://10.32.14.232:9292/v2", 
   "region": "RegionOne", 
   "service_id": "574f359c07fc449ab6b0b4fad42b2df9"
   }, 
   {
   "adminurl": "http://10.32.14.232:8773/services/Admin", 
   "id": "d50733db9848451596c84b782906cba1", 
   "internalurl": "http://10.32.14.232:8773/services/Cloud", 
   "publicurl": "http://10.32.14.232:8773/services/Cloud", 
   "region": "RegionOne", 
   "service_id": "a0fdb3cd3a234cada512ba0a75a6df56"
   }
   ]
   }

Installation et configuration de Glance
---------------------------------------

* Installation du service de stockage des images (Glance)::

    apt-get install glance

* Création de la base de donnée pour Glance::

   mysql -u root -p
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
   quit;

* Remplacer la section existante filter:authtoken dans le fichier /etc/glance/glance-api-paste.ini avec::

   vi /etc/glance/glance-api-paste.ini

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.32.14.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass
 
* Remplacer la section existante filter:authtoken dans le fichier /etc/glance/glance-registry-paste.ini avec::

   vi /etc/glance/glance-registry-paste.ini

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.32.14.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Ouvrer le fichier /etc/glance/glance-api.conf et mettre à jour les paramètres suivants::

   vi /etc/glance/glance-api.conf

   sql_connection = mysql://glanceUser:glancePass@10.32.14.232/glance

   [paste_deploy]
   flavor = keystone

* Ouvrer le fichier /etc/glance/glance-registry.conf et mettre à jour les paramètres suivants::

   vi /etc/glance/glance-registry.conf

   sql_connection = mysql://glanceUser:glancePass@10.32.14.232/glance

   [paste_deploy]
   flavor = keystone

* Relance des services glance-api et glance-registry::

   service glance-api restart; service glance-registry restart

* Synchronisation de la base de donnée de Glance::

   glance-manage db_sync

* **Note: Vous aurez probablement un avertissement concernant 'useexisting'. C'est normal, pas d'inquiétude.**

* Relance des service de nouveau afin de prendre en compte les modifications::

   service glance-api restart; service glance-registry restart

* Faisons un test de notre installation Glance en installant une image cirros cloud du miroir Launchpad::

   mkdir images
   cd images
   wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
   glance image-create --name GlanceTest --is-public true --container-format bare --disk-format qcow2 < cirros-0.3.0-x86_64-disk.img

* La dernière commande doit afficher une réponse similaire à ça::

   +-----------------+------------------------------------+
   | Property | Value |
   +----------------+------------------------------------+
   | checksum | 50bdc35edb03a38d91b1b071afb20a3c |
   | container_format | bare |
   | created_at | 2012-12-04T21:52:49 |
   | deleted | False |
   | deleted_at | None |
   | disk_format | qcow2 |
   | id | 9f045abf-3aa4-40d9-a9e1-7ab7bfa3e1ef |
   | is_public | True |
   | min_disk | 0 |
   | min_ram | 0 |
   | name | GlanceTest |
   | owner | b302c28c0f0e4d2f8f4d99553fc3971f |
   | protected | False |
   | size | 9761280 |
   | status | active |
   | updated_at | 2012-12-04T21:52:50 |
   +----------------+------------------------------------+

* Vérifions que l'image a bien été installée::

   glance image-list

* Vous devrez avoir une réponse similaire à ça::

   +--------------------------------------+-------------+-------------+------------------+---------+--------+
   | ID                                   | Name        | Disk Format | Container Format | Size    | Status |
   +--------------------------------------+-------------+-------------+------------------+---------+--------+
   | 74cec29b-76a1-4e89-8060-f0e2623ae5bf | NimbulaTest | qcow2       | bare             | 9761280 | active |
   +--------------------------------------+-------------+-------------+------------------+---------+--------+


Installation et configuration de Nova
-------------------------------------

* Installation de Nova et autres::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-network nova-compute-kvm

* Nous allons enlever le endpoint et le service Quantum car le script utilisé pour Keystone ajoute la configuration de quantum au lieu de nova-network. Avoir les 2 entrées risque de causer des problèmes pour la suite

* Récupération de l'identifiant du endpoint::

   keystone endpoint-list | grep 9696

* Vous devriez avoir un retour similaire à ça::

   | b0974d6c9bbb4f2cab281f3ff5bcd412 | RegionOne | http://10.32.14.232:9696/ | http://10.32.14.232:9696/ | http://10.32.14.232:9696/ | 

* Récupérer l'ID et suppression du endpoint::

   keystone endpoint-delete b0974d6c9bbb4f2cab281f3ff5bcd412

* Récupération de l'identifiant du service pour Quantum::

   keystone service-list | grep quantum

* Vous devriez avoir un retour similaire à ça::

   | 9e3b400f6531414c93262644f20cfda1 | quantum  |   network    | OpenStack Networking Service |

* Récupérer l'ID et suppression du service::

   keystone service-delete 9e3b400f6531414c93262644f20cfda1

* Création de la base de donnée pour Nova::

   mysql -u root -p
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
   quit;

* Modification de la section authtoken dans le fichier /etc/nova/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.32.14.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova

* Modification du fichier /etc/nova/nova-compute.conf::

   vi /etc/nova/nova-compute.conf

   [DEFAULT]
   libvirt_type=kvm

* Remplacer le fichier /etc/nova/nova.conf par la configuration suivante::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   scheduler_driver=nova.scheduler.simple.SimpleScheduler
   s3_host=10.32.14.232
   ec2_host=10.32.14.232
   ec2_dmz_host=10.32.14.232
   rabbit_host=10.32.14.232
   cc_host=10.32.14.232
   metadata_host=10.32.14.232
   metadata_listen=0.0.0.0
   nova_url=http://10.32.14.232:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.32.14.232/nova
   ec2_url=http://10.32.14.232:8773/services/Cloud
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
    
   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://10.32.14.232:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=10.32.14.232:9292
   image_service=nova.image.glance.GlanceImageService
    
   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://10.32.14.232:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.32.14.232
   vncserver_listen=0.0.0.0
    
   # NETWORK
   network_manager=nova.network.manager.FlatDHCPManager
   force_dhcp_release=True
   dhcpbridge_flagfile=/etc/nova/nova.conf
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   # Change my_ip to match each host
   my_ip=10.32.14.232
   public_interface=br100
   vlan_interface=eth1
   flat_network_bridge=br100
   flat_interface=eth1
   #Note the different pool, this will be used for instance range
   fixed_range=10.33.14.0/24
    
   # Compute #
   compute_driver=libvirt.LibvirtDriver
    
   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* Synchronisation de la base de donnée Nova::

   nova-manage db sync

* **Note: Vous devriez avoir des retours de DEBUG mentionnant 'nova.db.sqlalchemy.migration'. C'est normal, pas d'inquiétude.**

* Relance des différents services de Nova::

   cd /etc/init.d/; for i in $(ls nova-*); do sudo service $i restart; done

* Vérification que les services sont ok::

   nova-manage service list

* Vous devriez avoir un retour similaire à ça::

   Binary           Host                                 Zone             Status     State Updated_At
   nova-cert        folsom-1                             nova             enabled    :-)   2012-12-22 18:30:58
   nova-consoleauth folsom-1                             nova             enabled    :-)   2012-12-22 18:30:57
   nova-scheduler   folsom-1                             nova             enabled    :-)   2012-12-22 18:31:05
   nova-network     folsom-1                             nova             enabled    :-)   2012-12-22 18:31:09
   nova-compute     folsom-1                             nova             enabled    :-)   2012-12-22 00:26:00


* **Note: Vous devriez avoir des retours de DEBUG mentionnant 'nova.db.sqlalchemy.migration'. C'est normal, pas d'inquiétude.**

Installation et configuration de Cinder
---------------------------------------

* Installation de Cinder::

   apt-get install cinder-api cinder-scheduler cinder-volume

* Création de la base de donnée pour Cinder::

   mysql -u root -p
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
   quit;

* Configuration du fichier api-paste.ini avec le modèle suivant::

   vi /etc/cinder/api-paste.ini

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 10.32.14.232
   service_port = 5000
   auth_host = 10.32.14.232
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* Configuration du fichier /etc/cinder/cinder.conf avec le modèle suivant::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@10.32.14.232/cinder
   api_paste_confg = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

* Synchronisation de la base de donnée Cinder::

   cinder-manage db sync

* Si votre serveur n'a pas été installé avec LVM, vous pouvez créer le groupe de volume nommé cinder-volumes de la manière suivantes::

   cd /var/lib/cinder
   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2

* Une fois dans fdisk, taper les commandes suivantes::

   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

* Création du volume physique et du groupe de volume::

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

* Ajout du volume dans rc.local afin de ne pas perdre le volume lors d'un reboot du système
* Ajouter la ligne suivante avant le exit 0 dans le fichier /etc/rc.local::

   losetup /dev/loop2 /var/lib/cinder/cinder-volumes

* Compilation des modules nécessaire pour iscsitarget::

   apt-get install iscsitarget iscsitarget-source iscsitarget-dkms

* Activation de iscsitarget au démarrage::

   vi /etc/default/iscsitarget

   ISCSITARGET_ENABLE=true

* Relance du service iscsitarget::

   service iscsitarget restart

* Relance des services cinder::

   service cinder-api restart
   service cinder-scheduler restart
   service cinder-volume restart

* Vérification du bon fonctionnement de cinder en créant un volume test de 1G::

   cinder create --display-name  test 1

* Vous devez avoir un retour similaire à ça::

   +---------------------+--------------------------------------+
   |       Property      |                Value                 |
   +---------------------+--------------------------------------+
   |     attachments     |                  []                  |
   |  availability_zone  |                 nova                 |
   |      created_at     |      2012-12-23T15:01:48.762336      |
   | display_description |                 None                 |
   |     display_name    |                 test                 |
   |          id         | 0fed655b-dfbd-43c8-9c36-e0696401eb29 |
   |       metadata      |                  {}                  |
   |         size        |                  1                   |
   |     snapshot_id     |                 None                 |
   |        status       |               creating               |
   |     volume_type     |                 None                 |
   +---------------------+--------------------------------------+

* Vérification que tout est ok avec la commande cinder list. Si le volume a un statut "available", c'est que tout est ok::

   cinder list
   +--------------------------------------+-----------+--------------+------+-------------+-------------+
   |                  ID                  |   Status  | Display Name | Size | Volume Type | Attached to |
   +--------------------------------------+-----------+--------------+------+-------------+-------------+
   | 0fed655b-dfbd-43c8-9c36-e0696401eb29 | available |     test     |  1   |     None    |             |
   +--------------------------------------+-----------+--------------+------+-------------+-------------+


Installation et configuration de Horizon::
------------------------------------------


* La fin est proche, installation de l'interface Horizon::

   apt-get install openstack-dashboard memcached

Création d'un réseau pour les VM
--------------------------------

* Création du réseau pour les VM::

   nova-manage network create --label=TestNetwork --fixed_range_v4=10.33.14.0/24 --bridge=br100 --num_networks=1 --multi_host=T

Création d'une VM de test
-------------------------

* Création du clé SSH pour l'accès au VM::

   ssh-keygen

* Importation de la clé public dans OpenStack::

   nova keypair-add --pub_key ~/.ssh/id_rsa.pub mykey

* Création des règles de filtrage réseau::

   nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
   nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

* Pour créer une VM, il est nécessaire de spécifier une taille de VM et l'image à utiliser, il faut donc récupérer les infos nécessaire::

   nova flavor-list
   +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
   | ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public | extra_specs |
   +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
   | 1  | m1.tiny   | 512       | 0    | 0         |      | 1     | 1.0         | True      | {}          |
   | 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      | {}          |
   | 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      | {}          |
   | 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      | {}          |
   | 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      | {}          |
   +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
   
   nova image-list
   +--------------------------------------+------------+--------+--------+
   | ID                                   | Name       | Status | Server |
   +--------------------------------------+------------+--------+--------+
   | 0a2f7eb1-c6c7-4159-abce-500c19a26cc5 | GlanceTest | ACTIVE |        |
   +--------------------------------------+------------+--------+--------+

* Repérer l'ID souhaité pour la taille de la VM ainsi que pour l'image

* Lancement de la première VM de test::

   nova boot --flavor 1 --image 0a2f7eb1-c6c7-4159-abce-500c19a26cc5 --security-groups default --key-name mykey ma_vm_de_test

* Au bout de quelques minutes, vous pouvez vérifier que votre VM est disponible (le statut doit être ACTIVE) via la commande nova-list::

   nova-list

  
Notes
=====

Si vous souhaitez faire des commentaires sur ce guide ou si vous voulez l'améliorer, envoyez un mail à olivier (at) obn.me
Le guide est présent sur GitHub. 

D'autres guides verront le jour prochainement. La liste des guides en préparation sont :

* Guide d'installation d'OpenStack Folsom sur plusieurs machines
* Guide d'utilisation de Keystone, Glance, Nova
* Guide de la haute disponibilité d'OpenStack Folsom
* Guide de l'utilisation de Quantum

Si vous souhaitez d'autres guides, envoyez moi un mail à olivier (at) obn.me 

