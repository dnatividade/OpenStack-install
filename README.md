# OpenStack-install
Step-by-step OpenStack configuration

Previous version (unformated) ...

![OpenStack-topologia](https://user-images.githubusercontent.com/43869367/61165794-7056e400-a4fb-11e9-96f1-d55ede6db0bd.png)



===========================================================================
#Configurar arquivo de hosts em todos os computadores
#/etc/hosts
10.0.0.11	controller
10.0.0.31	compute1
10.0.0.41	compute2
10.0.0.51	compute3
10.0.0.61	compute4

#Em todos os nos instale o servidor NTP
apt install chrony
===========================================================================

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

### CONTROLLER ##
#Instalar OpenStack
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

#Criar usuario no rabbitmqctl
rabbitmqctl add_user openstack RABBIT_PASS
===========================================================================

#Instalar e configurar componentes

#MEMCACHED#
apt install memcached python-memcache

#Alterar o arquivo /etc/memcached.conf
-l 10.0.0.11

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

#finalizar instalação do etcd
systemctl enable etcd
systemctl start etcd
===========================================================================
===========================================================================

### Instalação dos modulos do CONTROLLER ###

#################################
## Identity Service - Keystone ##
#################################
apt install python-pyasn1

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
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

#...

[token]
# ...
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

#Executar o arquivo criado:
source admin-openrc
===========================================================================

## Criar dominio, project, users e roles ## (ainda como usuário comum)
#criar dominio
openstack domain create --description "An Example Domain" example

#criar 2 projects: "service" e "myproject"
openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" myproject

#criar usuário "myuser" (informe a senha em seguida)
openstack user create --domain default --password-prompt myuser

#criar role "myrole"
openstack role create myrole

#adicionar "myrole" em "myproject" com "myuser"
openstack role add --project myproject --user myuser myrole
===========================================================================

### Verify operation ### (ainda como usuario comum)
unset OS_AUTH_URL OS_PASSWORD

#digitar o comando abaixo (para testar) e em seguida informar a senha ADMIN_PASS
openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue
===========================================================================

### Create OpenStack client environment scripts ###

#Criar um script "demo"
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

#Testando "admin-openrc"
. admin-openrc
openstack token issue

#### (os passos abaixo não foram executados ####
https://docs.openstack.org/keystone/rocky/getting-started/index.html
===========================================================================

############################
## Image Service - Glance ##
############################

#ainda como uuario comum...
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
#(comment out or remove any other options in the [keystone_authtoken] section)
###########################################################################
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

# ...

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

# ...

[paste_deploy]
# ...
flavor = keystone

# ...

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
###########################################################################

#Altere o arquivo: /etc/glance/glance-registry.conf
#(comment out or remove any other options in the [keystone_authtoken] section)
#(will be DEPRECATED)
###########################################################################
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

# ...

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

# ...

[paste_deploy]
# ...
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

#### (os passos abaixo não foram executados) ####
https://docs.openstack.org/glance/rocky/configuration/index.html
===========================================================================

##############################
### Compute Service - Nova ###
##############################

Continuar aqui....

