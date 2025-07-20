# OpenStack Networking Explained - Using the ML2 plug-in with Open vSwitch (OVS)

The traditional implementation plays a role in the networking aspect of self-service virtual data center infrastructure by enabling regular (non-privileged) users to control virtual networks within a project. It encompasses the following components:


## 1. Project (tenant) networks

Project networks connect different instances for a specific project. Regular users can manage these networks based on the limits set by an administrator. Project networks can use different transport methods like VLAN, GRE, or VXLAN, depending on what's been allocated to them. They usually use private IP addresses (as defined in RFC1918) and do not connect to outside networks, like the Internet. In project networks, IP addresses are referred to as fixed IP addresses.

## 2. External networks

External networks connect to outside networks like the Internet. Only special users with admin rights can manage these networks because they are linked to the physical hardware. External networks can use either flat or VLAN methods to send data, depending on the hardware setup, and usually use public IP addresses.

**Note**: A flat network uses the untagged or native VLAN. Like physical networks at layer 2, only one flat network can be connected to each external bridge. Usually, for Production setups, it’s better to use VLANs to carry external network traffic.

## 3. Routers

Routers typically connect project and external networks. 
- By default, they implement SNAT to provide outbound external connectivity for instances on project networks. Each router uses an IP address in the external network allocation for SNAT.
- Routers also use DNAT to provide inbound external connectivity for instances on project networks.
- Networking involves using IP addresses on routers to provide external access to instances in project networks, known as floating IP addresses.
- Routers can also connect project networks that belong to the same project.

## 4. Supporting services

Other supporting services include 
1. **DHCP**- The DHCP service manages IP addresses for instances on project networks.
2. **Metadata**- The metadata service provides an API for instances on project networks to obtain metadata such as SSH keys.

The example configuration creates one flat external network and one VXLAN project (tenant) network. However, this configuration also supports VLAN external networks, VLAN project networks, and GRE project networks.

## Prerequisites¶


### Infrastructure¶
1. One controller node with one network interface: management.
2. One network node with four network interfaces: management, project tunnel networks, VLAN project networks, and external (typically the Internet). The Open vSwitch bridge br-vlan must contain a port on the VLAN interface and Open vSwitch bridge br-ex must contain a port on the external interface.
3. At least one compute node with three network interfaces: management, project tunnel networks, and VLAN project networks. The Open vSwitch bridge br-vlan must contain a port on the VLAN interface.

To improve understanding of network traffic flow, the network and compute nodes contain a separate network interface for VLAN project networks. In production environments, VLAN project networks can use any Open vSwitch bridge with access to a network interface. For example, the br-tun bridge.

In the example configuration, the management network uses 10.0.0.0/24, the tunnel network uses 10.0.1.0/24, and the external network uses 203.0.113.0/24. The VLAN network does not require an IP address range because it only handles layer-2 connectivity.

