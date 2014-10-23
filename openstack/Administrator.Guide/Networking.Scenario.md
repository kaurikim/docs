
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


tenant network 간의 단절의 위해서 switches 의 vlan segmentation 를 사용하는 예시를 들것음.
public network 을 위한 physnet1, data network 을 위한 physnet2 라는 명칭을 ovs_neutron_plugin.ini 파일에 설정함.


```
[ovs]
tenant_network_type = vlan
network_vlan_ranges = physnet1,physnet2:100:110
integration_bridge = br-int
bridge_mappings = physnet2:br-eth1
```

## Scenario 1: one tenant, two networks, one router

The first scenario has two private networks (net01, and net02), 
each with one subnet (net01_subnet01: 192.168.101.0/24, net02_subnet01, 192.168.102.0/24). 
Both private networks are attached to a router that connects them to the public network (10.64.201.0/24).

![](http://docs.openstack.org/admin-guide-cloud/content/figures/14/a/a/common/figures/under-the-hood-scenario-1.png "one tenant, two networks, one router")

Under the service tenant, create the shared router, define the public network, and set it as the default gateway of the router

service 텐넌트에 shared route 를 생성하고 이를 public network 로 정의, 해당 라우터를 default gateway 로 설정한다. 

```
$ tenant=$(keystone tenant-list | awk '/service/ {print $2}') 
$ neutron router-create router01
$ neutron net-create --tenant-id $tenant public01 \
    --provider:network_type flat \
    --provider:physical_network physnet1 \
    --router:external True
$ neutron subnet-create --tenant-id $tenant --name public01_subnet01 \ --gateway 10.64.201.254 public01 10.64.201.0/24 --disable-dhcp
$ neutron router-gateway-set router01 public01
```

Under the demo user tenant, create the private network net01 and corresponding subnet, 
and connect it to the router01 router. Configure it to use VLAN ID 101 on the physical switch.

demo user 텐넌트를 위한 사설 망 net01을 생성하고, 
     vlan id 101 로 지정된 관련 subnet 을 route01 라우터에 연결

```
$ tenant=$(keystone tenant-list|awk '/demo/ {print $2}' 
$ neutron net-create --tenant-id $tenant net01 \
    --provider:network_type vlan \
    --provider:physical_network physnet2 \
    --provider:segmentation_id 101
$ neutron subnet-create --tenant-id $tenant --name net01_subnet01 net01 192. 168.101.0/24
$ neutron router-interface-add router01 net01_subnet01
```

Similarly, for net02, using VLAN ID 102 on the physical switch:
같은 방식으로, net02 사설망을 생성 후, vlan id 102 를 가지는 subnet 을 생성

```
$ neutron net-create --tenant-id $tenant net02 \ --provider:network_type vlan \
          --provider:physical_network physnet2 \
          --provider:segmentation_id 102
          $ neutron subnet-create --tenant-id $tenant --name net02_subnet01 net02 192. 168.102.0/24
          $ neutron router-interface-add router01 net02_subnet01
```

### Compute Host config

The following figure shows how to configure various Linux networking devices on the compute host:

아래 그림은 compute host 에서 어떻게 네트워크 디바이스를 구성을 보여준다.

![](http://docs.openstack.org/admin-guide-cloud/content/figures/14/a/a/common/figures/under-the-hood-scenario-1-ovs-compute.png "Types of network devices")

    <!> There are four distinct type of virtual networking devices: 
    TAP devices, veth pairs, Linux bridges, and Open vSwitch bridges. 
    For an Ethernet frame to travel from eth0 of virtual machine vm01 
    to the physical network, it must pass through nine devices inside 
    of the host: TAP vnet0, Linux bridge qbrNNN, veth pair (qvbNNN, qvoNNN), 
    Open vSwitch bridge br-int, veth pair (int-br-eth1, phy-br-eth1), 
    and, finally, the physical network interface card eth1.

