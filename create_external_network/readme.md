### Create external (or public) network
```
#Como usuario comum:
. admin-openrc

#Create the external network
openstack network create --share --external --provider-physical-network provider --provider-network-type flat provider

#Create the subnetwork to link to external network
#(here may be config. with REAL IP ADDRESS)
openstack subnet create --network provider --allocation-pool start=200.1.2.30,end=200.1.2.40 --dns-nameserver 8.8.4.4 --gateway 200.1.2.1 --subnet-range 200.1.2.0/24 provider
```

