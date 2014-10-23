
Plug-ins typically have requirements for particular software that must be run on each node that handles data packets. 

* plug-ins 는 각 노드에서 일반적으로 데이터 패킷을 처리학 위해서 구동되어야 하는 특정 S/W 를 필요로 한다. 

This includes any node that runs nova-compute 
and nodes that run dedicated OpenStack Networking service agents such as neutron-dhcp-agent, neutron-l3-agent, neutron-metering-agent or neutron-lbaas-agent.

* 에이젼트는 nova-compute 가 구동되고, 
* 특정 목적의 OpenStack Networking service (neutron-l3-agent, neutron-metering-agent or neutron-lbaas-agent) 가 구동되는 노드에 위치할 수 있다.

A data-forwarding node typically has a network interface with an IP address on the management network and another interface on the data network.


This section shows you 
    * how to install and configure a subset of the available plug-ins, 
    * which might include the installation of switching software (for example, Open vSwitch) 
    * and as agents used to communicate with the neutron-server process running elsewhere in the data center.

    
### Configure data-forwarding nodes

#### Node set up: OVS plug-ien
    
    <!> This section also applies to the ML2 plug-in when you use Open vSwitch 
    as a mechanism driver.

    If you use the Open vSwitch plug-in, you must install Open vSwitch and the 
    neuron-plugin-openvswitch-agent agent on each data-forwarding node:

* Open vSwitch plug-in 을 사용하는 경우, 
    * 반드시 각 data-forwarding node 에 Open vSwitch 와  openvswitch-agent 를 설치 


#### Node Set UP:  NSX plug-in

* NSX Plug-in 을 사용하는 경우,
    * 각 data-forwarding node 에 Open vSwitch 를 설치
    * 각 node 에 추가적인 agent 를 설치할 필요는 없다.


### Configure DHCP agent

The OpenStack Networking Service has a widely used API extension to allow administrators and tenants to create routers to interconnect L2 networks, and floating IPs to make ports on private networks publicly accessible.
