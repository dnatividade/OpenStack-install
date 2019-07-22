# Create Port Forwarding or DNAT
##### OpenStack use Floating IP to do this

## CONTROLLER NODE

draft...
```
#List VMs
openstack server list

#Identifies the PORT_ID that represents the VM NIC to which the floating IP should map
openstack port list -c ID -c "Fixed IP Addresses" --server INSTANCE_ID


#Lists routers
openstack router list

#Shows information for a specified router
openstack router show ROUTER_ID

#Shows all internal interfaces for a router
openstack port list --router  ROUTER_ID
#openstack port list --router  ROUTER_NAME


#Create Floating IP
#openstack floating ip create --port INTERNAL_VM_PORT_ID EXT_NET_ID
openstack floating ip create --port INTERNAL_VM_PORT_ID EXT_NET_ID


#Lists floating IPs 	
openstack floating ip list

#Finds floating IP for a specified VM port. 	
openstack floating ip list --port INTERNAL_VM_PORT_ID
```
draft...
