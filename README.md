# OpenStack Instalation and Configuration  ![Alt Text](http://object-storage-ca-ymq-1.vexxhost.net/swift/v1/6e4619c416ff4bd19e1c087f27a43eea/www-assets-prod/Uploads/openstack-vert.jpg)

Step-by-step OpenStack configuration

Previous version (unformated) ...

![OpenStack-topologia](https://user-images.githubusercontent.com/43869367/61165794-7056e400-a4fb-11e9-96f1-d55ede6db0bd.png)


## CONTROLLER NODE and COMPUTE NODES
```
#Configurar arquivo /etc/hosts
#em hosts em todos os computadores
###########################################################################
10.0.0.11	controller
10.0.0.31	compute1
10.0.0.41	compute2
10.0.0.51	compute3
10.0.0.61	compute4
###########################################################################

#Em todos os nos instale o serviço de NTP
apt install chrony
===========================================================================
```
## CONTROLLER NODE
```
### CONTROLLER ###
#Config NTP Server
#no arquivo /etc/chrony.conf 
#adicione as linhas abaixo: 
server 200.160.7.186 iburst
allow 10.0.0.0/24

#Reincie o serviço
service chrony restart

#Teste a configuração
chronyc sources
===========================================================================
```
## COMPUTE NODES
```
### COMPUTE NODES ###
#Config NTP Client
#comente todas as linhas do arquivo /etc/chrony.conf 
#e adicione a linha abaixo: 
server controller iburst

#Reincie o serviço
service chrony restart

#Teste a configuração
chronyc sources
===========================================================================
```
## CONTROLLER NODE
```
### CONTROLLER ##
#Instalar OpenStack client
add-apt-repository cloud-archive:rocky
apt update && apt dist-upgrade
apt install python-openstackclient
===========================================================================

### CONTROLLER ###
#Instalar Banco de Dados
apt install mariadb-server python-pymysql

#Configure o banco de dados
#Adicione as linhas abaixo no arquivo: /etc/mysql/mariadb.conf.d/99-openstack.cnf
###########################################################################
[mysqld]
bind-address = 10.0.0.11

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
###########################################################################

#Reinicie o banco de dados
service mysql restart

#Configure uma senha para "root" no banco de dados:	
mysql_secure_installation
[informe: senha atual (em branco), nova senha, nova senha]
===========================================================================

### CONTROLLER ###
#Configurar Message queue
apt install rabbitmq-server

#Criar usuario no rabbitmqctl a ajustar permissões
rabbitmqctl add_user openstack RABBIT_PASS
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
===========================================================================

### Instalar e configurar componentes 
#MEMCACHED#
apt install memcached python-memcache

#Alterar o arquivo /etc/memcached.conf
#(comente a linha abaixo)
###########################################################################
#-l 127.0.0.1
###########################################################################

#Reinicie o serviço
service memcached restart

#ETCD#
apt install etcd

#Editar o arquivo /etc/default/etcd
###########################################################################
ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://10.0.0.11:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.11:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.11:2379"
###########################################################################

#Finalizar instalação do etcd
systemctl enable etcd
systemctl start etcd
===========================================================================
===========================================================================
```
## OPENSTACK MODULES INSTALLATION
```
############################################
### Instalação dos modulos do CONTROLLER ###
############################################

#################################
## Identity Service - Keystone ##
#################################

#Instalação
apt install python-pyasn1

#Criar bases
mysql -u root -p
(enter root password)
> CREATE DATABASE keystone;
> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
> USE mysql;
> UPDATE user SET plugin='mysql_native_password' WHERE User='root';
> FLUSH PRIVILEGES;
> exit

#Instalar keystone e Apache
apt install keystone apache2 libapache2-mod-wsgi

#Edite o arquivo: /etc/keystone/keystone.conf
###########################################################################
[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
provider = fernet
###########################################################################

#Populate the Identity service database
su -s /bin/sh -c "keystone-manage db_sync" keystone

#Initialize Fernet key repositories:
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

#Bootstrap the Identity service
keystone-manage bootstrap --bootstrap-password ADMIN_PASS --bootstrap-admin-url http://controller:5000/v3/ --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne

#Configurar Apache: /etc/apache2/apache2.conf
###########################################################################
ServerName controller
###########################################################################

#Reiniciar Apache
service apache2 restart
===========================================================================

#Entrar com usuario comum (não root)
#no diretorio home, criar um arquivo chamado "admin-openrc", com o conteudo:
###########################################################################
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
###########################################################################

#Ativar as variáveis de ambiente do arquivo criado:
. admin-openrc
#(o mesmo que: source admin-openrc)
===========================================================================

## Criar dominio, project, users e roles ## (ainda como usuário comum)
#criar dominio
openstack domain create --description "An Example Domain" example

#Criar 2 projects: "service" e "myproject"
openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" myproject

#Criar usuário "myuser" (informe a senha em seguida)
openstack user create --domain default --password-prompt myuser

#Criar role "myrole"
openstack role create myrole

#Adicionar "myrole" em "myproject" com "myuser"
openstack role add --project myproject --user myuser myrole
===========================================================================

### Verify operation ### (ainda como usuario comum)
unset OS_AUTH_URL OS_PASSWORD

#Digitar o comando abaixo (para testar) e em seguida informar a senha ADMIN_PASS
openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue
===========================================================================

### Create OpenStack client environment scripts ###

#Criar um script para o usuario "demo"
#Entrar com usuario comum (não root)
#no diretorio home, criar um arquivo chamado "demo-openrc", com o conteudo:
###########################################################################
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=MYUSER_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
###########################################################################

#Testando "demo-openrc"
. admin-openrc
openstack token issue

#$$$$$$$$$ Os passos abaixo não foram executados $$$$$$$$$$$$
https://docs.openstack.org/keystone/rocky/getting-started/index.html
===========================================================================

############################
## Image Service - Glance ##
############################

#Criar bases
#(ainda como uuario comum)
mysql -u root -p
(informe a senha de root do MariaDB)
> CREATE DATABASE glance;
> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
> exit

#Carregue o script "admin-openrc"
. admin-openrc

#Criar as credenciais de serviço
openstack user create --domain default --password-prompt glance

#Adicionar role "admin" para o usuario "glance" no project "service"
openstack role add --project service --user glance admin

#Criar um serviço "glance"
openstack service create --name glance --description "OpenStack Image" image

#Create the Image service API endpoints
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
===========================================================================

#Instalar os pacotes do Glance
apt install glance

#Altere o arquivo: /etc/glance/glance-api.conf
#(remove any other options in the [keystone_authtoken] section)
###########################################################################
[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
###########################################################################

#Altere o arquivo: /etc/glance/glance-registry.conf
#(remove any other options in the [keystone_authtoken] section)
#(will be DEPRECATED)
###########################################################################
[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
flavor = keystone
###########################################################################

#Populate the image service database
#(ignore any deprecation messages in this output)
su -s /bin/sh -c "glance-manage db_sync" glance

#Finalizar a instalação
service glance-registry restart
service glance-api restart
===========================================================================

#Verificação e teste
#Como usuario comum
. admin-openrc

#Baixar imagem Cirros
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img

#Subir
openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public

#Verificar
openstack image list

#$$$$$$$$$ Os passos abaixo não foram executados $$$$$$$$$$$$
https://docs.openstack.org/glance/rocky/configuration/index.html
===========================================================================

##############################
### Compute Service - Nova ###
##############################

#*********************#
#** CONTROLLER NODE **#
#*********************#

### Install and configure controller node ###

#Criar bases
mysql -u root -p
> CREATE DATABASE nova_api;
> CREATE DATABASE nova;
> CREATE DATABASE nova_cell0;
> CREATE DATABASE placement;

> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';
> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';
> exit
===========================================================================

#Com usuario comum
. admin-openrc

#Create the Compute service credentials
openstack user create --domain default --password-prompt nova

#Add the admin role to the nova use
openstack role add --project service --user nova admin

#Criar nova service entry
openstack service create --name nova --description "OpenStack Compute" compute

#Create the Compute API service endpoints
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1

#Create a Placement service
openstack user create --domain default --password-prompt placement

#Add the Placement user to the service project with the admin role
openstack role add --project service --user placement admin

#Create the Placement API entry in the service catalog
openstack service create --name placement --description "Placement API" placement

#Create the Placement API service endpoints
openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
===========================================================================

### Install and configure components ###
#Instalar os pacotes
#(como root)
apt install nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler nova-placement-api

#Configure o arquivo: /etc/nova/nova.conf
###########################################################################
[api_database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

[placement_database]
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 10.0.0.11
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
###########################################################################

#Populate the nova-api and placement databases
#(ainda como root)
su -s /bin/sh -c "nova-manage api_db sync" nova

#Register the cell0 database
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

#Create the cell1 cell
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

#Populate the nova database:
su -s /bin/sh -c "nova-manage db sync" nova

#Verify nova cell0 and cell1 are registered correctly
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova

#Restart the Compute services
service nova-api restart
service nova-consoleauth restart
service nova-scheduler restart
service nova-conductor restart
service nova-novncproxy restart
===========================================================================

###############
### Neutron ###
###############
#(Using: Networking Option 2: Self-service networks)

## Controller node ##

mysql -u root -p
(entre com a senha de root)
> CREATE DATABASE neutron;
> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
> exit

#como usuario comum
. admin-openrc

openstack user create --domain default --password-prompt neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network

openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
===========================================================================

#Instalar componentes
#como root
apt install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent

#Configure the server component
#Configure o arquivo: /etc/neutron/neutron.conf
###########################################################################
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"

[database]
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
###########################################################################
===========================================================================

#Configure the Modular Layer 2 (ML2) plug-in
#Configure o arquivos: /etc/neutron/plugins/ml2/ml2_conf.ini
###########################################################################
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000

[securitygroup]
enable_ipset = true
###########################################################################
===========================================================================

#Configure the Linux bridge agent
#edite o arquivos: /etc/neutron/plugins/ml2/linuxbridge_agent.ini
###########################################################################
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME_EXTERNAL_IFACE

[vxlan]
enable_vxlan = true
local_ip = IP_CONTROLLER_NODE
l2_population = true

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
###########################################################################

#Garanta que as opções abaixo, estão setados em 1 no arquivo "/etc/sysctl.conf"
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

#Garanta que o modulo "br_netfilter" está carregado no kernel, com o comando:
lsmod |grep br_netfilter
===========================================================================

#Configure the layer-3 agent
#edite o arquivo: /etc/neutron/l3_agent.ini
###########################################################################
[DEFAULT]
interface_driver = linuxbridge
###########################################################################

#Configure o DHCP agente
#edite o arquivo: /etc/neutron/dhcp_agent.ini
###########################################################################
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
###########################################################################
===========================================================================

#Configure the metadata agent
#edite o arquivo: /etc/neutron/metadata_agent.ini
###########################################################################
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
###########################################################################

#Adicione as sessões abaixo no arquivo: /etc/nova/nova.conf
###########################################################################
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET

[scheduler]
#search and add new nodes every x seconds
discover_hosts_in_cells_interval = 300
###########################################################################
===========================================================================

#Populate the database
#(ainda como root)
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
===========================================================================

#Restart services:
service nova-api restart

service neutron-server restart
service neutron-linuxbridge-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart

service neutron-l3-agent restart
===========================================================================

###################################
### Dashboard Service - Horizon ###
###################################

### Controller node ###

#Instalar serviço
apt install openstack-dashboard

#Edite o arquivo: /etc/openstack-dashboard/local_settings.py
#com os paramentros abaixo:
###########################################################################
OPENSTACK_HOST = "controller"
# ...
ALLOWED_HOSTS = ['*']
#(inseguro deixar assim, mas para testes, thats ok! ;)
# ...
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
# ...
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
# ...
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
# ...
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
# ...
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
# ...
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
# ...
TIME_ZONE = "America/Sao_Paulo"
###########################################################################

#Edite o arquivo: /etc/apache2/conf-available/openstack-dashboard.conf
###########################################################################
WSGIApplicationGroup %{GLOBAL}
###########################################################################

#Reinicie o Apache2
service apache2 reload
===========================================================================
===========================================================================
```
## COMPUTE NODES
```
#*******************#
#** COMPUTE NODEs **#
#*******************#

######################
###  NOVA SERVICES ###
######################

#Install components
apt install nova-compute

#Configure o arquivo: /etc/nova/nova.conf
#(deixe-o como abaixo)
###########################################################################
[DEFAULT]
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova

transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 10.0.0.31
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[api_database]
connection = sqlite:////var/lib/nova/nova_api.sqlite

[cells]
enable = False

[database]
connection = sqlite:////var/lib/nova/nova.sqlite

[devices]
[ephemeral_storage_encryption]
[filter_scheduler]

[glance]
api_servers = http://controller:9292

[keystone]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
os_region_name = openstack

region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
###########################################################################

#Determinar se o computador suporta hardware acceleration for virtual machines
egrep -c '(vmx|svm)' /proc/cpuinfo
#SE o resultado for ZERO, então configure o arquivo: /etc/nova/nova-compute.conf
###########################################################################
[libvirt]
# ...
virt_type = qemu
###########################################################################

#Restart o serviço
service nova-compute restart
===========================================================================


#########################
###  Neutron SERVICES ###
#########################

#Install components
apt install neutron-linuxbridge-agent

#Edit o arquivo: /etc/neutron/neutron.conf
#(deixe-o desta forma)
###########################################################################
[DEFAULT]
core_plugin = ml2
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone

[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"

[database]
connection = sqlite:////var/lib/neutron/neutron.sqlite

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
###########################################################################

#Edit o arquivo: /etc/nova/
#e insire a sessão abaixo
###########################################################################
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
###########################################################################

#Finalizar a instalação
service nova-compute restart
service neutron-linuxbridge-agent restart
===========================================================================
===========================================================================
```
## CONTROLLER NODE
### Force search and add new nodes
```
#*********************#
#** CONTROLLER NODE **#
#*********************#

#Add compute node
#(como usuario comum)
. admin-openrc
openstack compute service list --service nova-compute

## DISCOVER COMPUTE HOSTS ## **********************************************
#(como root)
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
===========================================================================
```


##### SOURCE: https://docs.openstack.org/install-guide/openstack-services.html



## Continua....

