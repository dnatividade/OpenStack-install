# Create external (or public) network
##### In this example, our network has public IPs available

## CONTROLLER NODE
```
#Como usuario comum:
. admin-openrc

#Create the external network
#(network need to be "flat" type)
#--share = network shared with other projects
#--provider-physical-network = NAME_CHOOSE into /etc/neutron/plugins/ml2/ml2_conf.ini file:
#[ml2_type_flat]
#flat_networks = NAME_CHOOSE
openstack network create --share --external --provider-physical-network provider --provider-network-type flat provider

#Create the subnetwork to link to external network
#(here will be config. with REAL IP ADDRESS)
#--subnet-range = network address
#--gateway network gateway
#--dns-nameserver = any DNS server
#--allocation-pool start/end = DHCP server pool
openstack subnet create --network provider --allocation-pool start=200.1.2.30,end=200.1.2.40 --dns-nameserver 8.8.4.4 --gateway 200.1.2.1 --subnet-range 200.1.2.0/24 provider
```

