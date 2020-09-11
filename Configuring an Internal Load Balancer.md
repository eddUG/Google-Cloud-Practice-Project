# Configuring an Internal Load Balancer

## Overview
Google Cloud offers Internal Load Balancing for your TCP/UDP-based traffic. Internal Load Balancing enables you to run and scale your services behind a private load balancing IP address that is accessible only to your internal virtual machine instances.

In this lab, you create two managed instance groups in the same region. Then you configure and test an internal load balancer with the instances groups

## Objectives
In this lab, you learn how to perform the following tasks:

* Create internal traffic and health check firewall rules
* Create a NAT configuration using Cloud Router
* Configure two instance templates
* Create two managed instance groups
* Configure and test an internal load balancer


## Task 1. Configure internal traffic and health check firewall rules.
Configure firewall rules to allow internal traffic connectivity from sources in the 10.10.0.0/16 range. This rule allows incoming traffic from any client located in the subnet.

Health checks determine which instances of a load balancer can receive new connections. For HTTP load balancing, the health check probes to your load-balanced instances come from addresses in the ranges **130.211.0.0/22** and **35.191.0.0/16**. Your firewall rules must allow these connections.

### Explore the my-internal-app network

The network **my-internal-app** with **subnet-a** and **subnet-b** and firewall rules for **RDP**, **SSH**, and **ICMP** traffic have been configured for you.
<pre><code> gcloud compute networks subnets list </code></pre>

Notice the **my-internal-app** network with its subnets: **subnet-a** and **subnet-b**.

Each Google Cloud project starts with the **default network**. In addition, the **my-internal-app** network has been created for you as part of your network diagram.

You will create the managed instance groups in **subnet-a** and **subnet-b**. Both subnets are in the **us-central1** region because an internal load balancer is a regional service. The managed instance groups will be in different zones, making your service immune to zonal failures.
