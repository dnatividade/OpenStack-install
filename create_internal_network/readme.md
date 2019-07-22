# Create internal (or private) network

## CONTROLLER NODE

##### For this configuration, I used Horizon Web interface:
- User "admin" -> Project -> Network
- Click in [+ Create Network]

- Choose a "Network Name", as: private;
- Check:
-- [v] Enable Admin State
-- [ ]  Shared
-- [v] Create Subnet
- Click in [Next]

- Choose a "Subnet Name", as: sub_private;
- Put a "Network Address", as: 10.20.0.0/24
- IP Version: IPv4
- Gateway IP: [blanck]
- Click in [Next]

- Check:  [v] Enable DHCP
- "Allocation Pools": 10.20.0.100,10.20.0.200
- "DNS Name Servers": 8.8.4.4
- Click in [Create]
