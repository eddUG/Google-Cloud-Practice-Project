
# Virtual Private Networks (VPN)

### Overview
In this lab, you establish VPN tunnels between two networks in separate regions such that a VM in one network can ping a VM in the other network over its internal IP address.

### Objectives
In this lab, you learn how to perform the following tasks:

* Create VPN gateways in each network
* Create VPN tunnels between the gateways
* Verify VPN connectivity


Task 1: Explore the networks and instances
Two custom networks with VM instances have been configured for you. For the purposes of the lab, both networks are VPC networks within a Google Cloud project. However, in a real-world application, one of these networks might be in a different Google Cloud project, on-premises, or in a different cloud.

Explore the networks
Verify that vpn-network-1 and vpn-network-2 have been created with subnets in separate regions.
