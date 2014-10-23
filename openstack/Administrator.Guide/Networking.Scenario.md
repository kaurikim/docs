
# Networking Scenarios

This chapter describes two networking scenarios and how the Open vSwitch plug-in and the Linux Bridge plug-in implement these scenarios.

## Open vSwitch

This section describes how the Open vSwitch plug-in implements the Networking abstractions.

이 장에서는 Open vSwitch 플러그인이 어떻게 네트워크 추상화를 구현하였는지 알아볼거임. 

## Configuration

This example uses VLAN segmentation on the switches to isolate tenant networks. 
This configuration labels the physical network associated with the public network 
as physnet1, and the physical network associated with the data network as physnet2, 
which leads to the following configuration options in ovs_neutron_plugin.ini:

```
[ovs]
tenant_network_type = vlan
network_vlan_ranges = physnet1,physnet2:100:110
integration_bridge = br-int
bridge_mappings = physnet2:br-eth1
```
