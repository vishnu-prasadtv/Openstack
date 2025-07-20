# OpenStack Networking Explained - Using the ML2 plug-in with Open vSwitch (OVS)

The traditional implementation plays a role in the networking aspect of self-service virtual data center infrastructure by enabling regular (non-privileged) users to control virtual networks within a project. It encompasses the following components:


## 1. Project (tenant) networks

- Project networks connect different instances for a specific project.
- Regular users can manage these networks based on the limits set by an administrator.
- Project networks can use different transport methods like VLAN, GRE, or VXLAN, depending on what's been allocated to them.
- They usually use private IP addresses (as defined in RFC1918) and do not connect to outside networks, like the Internet.
- In project networks, IP addresses are referred to as fixed IP addresses.

## 2. External networks

- External networks connect to outside networks like the Internet.
- Only special users with admin rights can manage these networks because they are linked to the physical hardware.
- External networks can use either flat or VLAN methods to send data, depending on the hardware setup, and usually use public IP addresses.

NOTE: A flat network uses the untagged or native VLAN. Like physical networks at layer 2, only one flat network can be connected to each external bridge. Usually, for Production setups, it’s better to use VLANs to carry external network traffic.

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
1. Controller node with:
 - One network interface: management.
2. Network node with four network interfaces:
   - Management network,
   - Project tunnel networks,
   - VLAN project networks,and
   - External (typically the Internet).
      - The Open vSwitch bridge ```br-vlan``` must contain a port on the VLAN interface and
      - Open vSwitch bridge ```br-ex``` must contain a port on the external interface.
3. Compute node with three network interfaces:
 - Management networks.
 - Project tunnel networks, and
 - VLAN project networks.
   NOTE: The Open vSwitch bridge ```br-vlan``` must contain a port on the VLAN interface.

To improve understanding of network traffic flow, the network and compute nodes is having a separate network interface for VLAN project networks. 
- NOTE: Whereas in the production environments, VLAN project networks can use any Open vSwitch bridge with access to a network interface. For example, the br-tun bridge.

In the example configuration, 
- The management network uses `10.0.0.0/24`,
- The project tunnel network uses `10.0.1.0/24`,
- The external network uses `203.0.113.0/24`.
- The VLAN network does not require an IP address range because it only handles layer-2 connectivity.
  - This means that devices on different VLANs cannot communicate directly without going through a router, providing a level of network segmentation and security.
  - The term "layer-2" refers to the data link layer in the OSI (Open Systems Interconnection) model.
  - At L2-layer, devices communicate over a network through MAC (Media Access Control) addresses rather than IP addresses.
  - Layer-2 connectivity involves the transmission of frames that control how data is sent over the physical medium.
  - This means that VLANs can efficiently manage local traffic without the complexities associated with IP addressing schemes.
  - For example, a single VLAN can facilitate communication between devices such as servers and workstations in a local area without requiring those devices to have assigned IP addresses.
  
## Hardware Requirement

<img width="551" height="421" alt="image" src="https://github.com/user-attachments/assets/7a3c03dc-a600-4dcf-8c3a-73a4553c9065" />

## Network Layout

<img width="882" height="397" alt="image" src="https://github.com/user-attachments/assets/fb6b0657-91a6-43cd-bd52-9d0fc21f1686" />

## Service Layout

<img width="595" height="483" alt="image" src="https://github.com/user-attachments/assets/63462675-9173-486d-897d-489edc0fa1df" />

NOTE- For VLAN external and project networks, the physical network infrastructure must support VLAN tagging. For best performance with VXLAN and GRE project networks, the network infrastructure should support jumbo frames.

## Openstack services in:

### Controller node¶
1. Operational SQL server with neutron database and appropriate configuration in the ```neutron.conf``` file. MySQL
2. Operational message queue service with appropriate configuration in the ```neutron.conf``` file. RabbitMQ
3. Operational OpenStack Identity service with appropriate configuration in the ```neutron.conf``` file. Keystone
4. Operational OpenStack Compute controller/management service with appropriate configuration to use neutron in the ```nova.conf``` file. Nova
5. Neutron server service, ML2 plug-in, and any dependencies.

### Network node¶
1. Operational OpenStack Identity service with appropriate configuration in the ```neutron.conf``` file.
2. Open vSwitch service, Open vSwitch agent, L3 agent, DHCP agent, metadata agent, and any dependencies.

### Compute nodes¶
1. Operational OpenStack Identity service with appropriate configuration in the ```neutron.conf``` file.
2. Operational OpenStack Compute controller/management service with appropriate configuration to use neutron in the ```nova.conf``` file.
3. Open vSwitch service, Open vSwitch agent, and any dependencies.
 
# Architecture¶

- The classic architecture provides basic virtual networking components in your environment.
- Routing among project and external networks resides completely on the network node.
- Even though it is more simpler to deploy than other architectures, performing all functions on a single network node creates a single-point of failure and potential performance issues. To avoid this, consider deploying DVR or L3 HA architectures in production environments to provide redundancy and increase performance.

## General Architecture

<img width="914" height="624" alt="image" src="https://github.com/user-attachments/assets/6f441854-cbb0-49c0-b696-7b2296d1e419" />

### Network Node Overview

The network node contains the following network components:

1. Open vSwitch agent manages:
   - Virtual switches and connectivity among them,
   - Interaction via *virtual ports* with other network components such as
      - Namespaces
      - Linux bridges
      - Underlying interfaces.
2. DHCP agent managing the ```qdhcp``` namespaces.
   - The ```qdhcp``` namespaces provide DHCP services for instances using project networks.
3. The L3 agent managing the ```qrouter``` namespaces.
   - The ```qrouter``` namespaces provide routing between project and external networks and among project networks.
   - They also route metadata traffic between instances and the metadata agent.
4. Metadata agent handling metadata operations for instances.

<img width="643" height="576" alt="image" src="https://github.com/user-attachments/assets/7c2b8927-85a9-4b03-81e1-9acb1d0dfb56" />

#### Network Node Components

<img width="925" height="763" alt="image" src="https://github.com/user-attachments/assets/041c7856-49d7-4604-9502-afaafdaa8d21" />


### Compute Node Overview

The compute nodes contain the following network components:

1. Open vSwitch agent managing"
   - Virtual switches, and onnectivity among them, and
   - Interaction via virtual ports with other network components such as 
        - Namespaces,
        - Linux bridges, and
        - Underlying interfaces.
2. Linux bridges handling security groups.
    - Due to limitations with Open vSwitch and iptables, the Networking service uses a Linux bridge to manage security groups for instances.

<img width="647" height="548" alt="image" src="https://github.com/user-attachments/assets/faf6f541-599d-470e-a74c-d66e4d344d6c" />

#### Compute Node Components

<img width="828" height="635" alt="image" src="https://github.com/user-attachments/assets/948fedda-5177-4a37-9bcb-286d882a5180" />

# Packet flow

NOTE:
- North-south network traffic travels between an instance and external network, typically the Internet.
- East-west network traffic travels between instances.

## Case 1: North-south for instances with a fixed IP address¶

For instances with a fixed IP address, the network node routes north-south network traffic between project and external networks.

- External network
  - Network `203.0.113.0/24`
  - IP address allocation from `203.0.113.101` to `203.0.113.200`
  - Project network router interface `203.0.113.101` TR
- Project network
  - Network `192.168.1.0/24`
  - Gateway `192.168.1.1` with MAC address TG
- Compute node 1
  - Instance 1 `192.168.1.11` with MAC address I1
 
- Instance 1 resides on compute node 1 and uses a project network.
- The instance sends a packet to a host on the external network.

#### Compute Node1 - The following steps are involved: Case-1
1. The instance 1 ```tap``` interface (1) forwards the packet to the Linux bridge ```qbr```. The packet contains destination MAC address *TG* because the destination resides on another network.
2. Security group rules (2) on the Linux bridge ```qbr``` handle state tracking for the packet.
3. The Linux bridge ```qbr``` forwards the packet to the Open vSwitch integration bridge ```br-int```.
4. The Open vSwitch integration bridge `br-int` adds the internal tag for the project network.
5. For VLAN project networks:
    - The Open vSwitch integration bridge `br-int` forwards the packet to the Open vSwitch VLAN bridge `br-vlan`.
    - The Open vSwitch VLAN bridge `br-vlan` replaces the internal tag with the actual VLAN tag of the project network.
    - The Open vSwitch VLAN bridge `br-vlan` forwards the packet to the network node via the VLAN interface.
6. For VXLAN and GRE project networks:
    - The Open vSwitch integration bridge br-int forwards the packet to the Open vSwitch tunnel bridge br-tun.
    - The Open vSwitch tunnel bridge br-tun wraps the packet in a VXLAN or GRE tunnel and adds a tag to identify the project network.
    - The Open vSwitch tunnel bridge br-tun forwards the packet to the network node via the tunnel interface.

<img width="1001" height="490" alt="image" src="https://github.com/user-attachments/assets/b64a1010-6151-47d8-b28a-25ccf6f0c931" />

#### Network node - The following steps are involved: Case-1

1. For VLAN project networks:
    - The VLAN interface forwards the packet to the Open vSwitch VLAN bridge `br-vlan`.
    - The Open vSwitch VLAN bridge `br-vlan` forwards the packet to the Open vSwitch integration bridge `br-int`.
    - The Open vSwitch integration bridge `br-int` replaces the actual VLAN tag of the project network with the internal tag.
2. For VXLAN and GRE project networks:
    - The tunnel interface forwards the packet to the Open vSwitch tunnel bridge `br-tun`.
    - The Open vSwitch tunnel bridge `br-tun` unwraps the packet and adds the internal tag for the project network.
    - The Open vSwitch tunnel bridge `br-tun` forwards the packet to the Open vSwitch integration bridge `br-int`.
3. The Open vSwitch integration bridge `br-int` forwards the packet to the `qr` interface (3) in the router namespace qrouter. The `qr` interface contains the project network gateway IP address TG.
4. The iptables service (4) performs SNAT on the packet using the `qg` interface (5) as the source IP address. The `qg` interface contains the project network router interface IP address TR.
5. The router namespace `qrouter` forwards the packet to the Open vSwitch integration bridge `br-int` via the `qg` interface.
6. The Open vSwitch integration bridge `br-int` forwards the packet to the Open vSwitch external bridge `br-ex`.
7. The Open vSwitch external bridge `br-ex` forwards the packet to the external network via the external interface.

NOTE: Return traffic follows similar steps in reverse.

<img width="1013" height="783" alt="image" src="https://github.com/user-attachments/assets/56a13f00-ed99-4fb3-91b6-b209556bea88" />


## Case 2: North-south for instances with a floating IP address

For instances with a floating IP address, the network node routes north-south network traffic between project and external networks.

- External network
    Network `203.0.113.0/24`
    IP address allocation from `203.0.113.101` to `203.0.113.200`
    Project network router interface `203.0.113.101` TR
- Project network
    Network `192.168.1.0/24`
    Gateway `92.168.1.1` with MAC address TG
- Compute node 1
    Instance 1 `192.168.1.11` with MAC address I1 and floating IP address `203.0.113.102` F1
- Instance 1 resides on compute node 1 and uses a project network.
- The instance receives a packet from a host on the external network.

#### Network node - The following steps are involved: Case-2

1. The external interface forwards the packet to the Open vSwitch external bridge `br-ex`.
2. The Open vSwitch external bridge `br-ex` forwards the packet to the Open vSwitch integration bridge `br-int`.
3. The Open vSwitch integration bridge forwards the packet to the `qg` interface (1) in the router namespace `qrouter`. The `qg` interface contains the instance 1 floating IP address F1.
4. The iptables service (2) performs DNAT on the packet using the `qr` interface (3) as the source IP address. The `qr` interface contains the project network router interface IP address TR1.
5. The router namespace qrouter forwards the packet to the Open vSwitch integration bridge `br-int`.
6. The Open vSwitch integration bridge `br-int` adds the internal tag for the project network.
7. For VLAN project networks:
    - The Open vSwitch integration bridge `br-int` forwards the packet to the Open vSwitch VLAN bridge `br-vlan`.
    - The Open vSwitch VLAN bridge `br-vlan` replaces the internal tag with the actual VLAN tag of the project network.
    - The Open vSwitch VLAN bridge `br-vlan` forwards the packet to the compute node via the VLAN interface.
8 For VXLAN and GRE project networks:
    - The Open vSwitch integration bridge `br-int` forwards the packet to the Open vSwitch tunnel bridge `br-tun`.
    - The Open vSwitch tunnel bridge `br-tun` wraps the packet in a VXLAN or GRE tunnel and adds a tag to identify the project network.
    - The Open vSwitch tunnel bridge `br-tun` forwards the packet to the compute node via the tunnel interface.

<img width="993" height="690" alt="image" src="https://github.com/user-attachments/assets/75ff00b7-f8c5-454b-82f5-34f1a7291efb" />

  
#### Compute Node1 - The following steps are involved: Case-2

1. For VLAN project networks:
    - The VLAN interface forwards the packet to the Open vSwitch VLAN bridge `br-vlan`.
    - The Open vSwitch VLAN bridge `br-vlan` forwards the packet to the Open vSwitch integration bridge `br-int`.
    - The Open vSwitch integration bridge `br-int` replaces the actual VLAN tag the project network with the internal tag.
2. For VXLAN and GRE project networks:
    - The tunnel interface forwards the packet to the Open vSwitch tunnel bridge `br-tun`.
    - The Open vSwitch tunnel bridge `br-tun` unwraps the packet and adds the internal tag for the project network.
    - The Open vSwitch tunnel bridge `br-tun` forwards the packet to the Open vSwitch integration bridge `br-int`.
3. The Open vSwitch integration bridge `br-int` forwards the packet to the Linux bridge `qbr`.
4. Security group rules (4) on the Linux bridge `qbr` handle firewalling and state tracking for the packet.
5. The Linux bridge `qbr` forwards the packet to the tap interface (5) on instance 1.

<img width="989" height="587" alt="image" src="https://github.com/user-attachments/assets/0a417e1a-5681-46cb-8208-1e9e3c8cdc69" />
