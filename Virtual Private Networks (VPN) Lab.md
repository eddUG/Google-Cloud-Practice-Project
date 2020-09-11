
# Virtual Private Networks (VPN)

## Overview
In this lab, you establish VPN tunnels between two networks in separate regions such that a VM in one network can ping a VM in the other network over its internal IP address.

## Objectives
In this lab, you learn how to perform the following tasks:

* Create VPN gateways in each network
* Create VPN tunnels between the gateways
* Verify VPN connectivity


## Task 1: Explore the networks and instances
Two custom networks with VM instances have been configured for you. For the purposes of the lab, both networks are VPC networks within a Google Cloud project. However, in a real-world application, one of these networks might be in a different Google Cloud project, on-premises, or in a different cloud.

### Explore the networks 

Verify that **vpn-network-1** and **vpn-network-2** have been created with subnets in separate regions.

    gcloud compute networks subnets list

* Note the **vpn-network-1** network and its **subnet-a** in **us-central1**.
* Note the **vpn-network-2** network and its **subnet-b** in **europe-west1**.


### Explore the firewall rules

<pre><code>  gcloud compute firewall-rules list </code></pre>

* Note the **network-1-allow-ssh** and **network-1-allow-icmp** rules for **vpn-network-1**.
* Note the **network-2-allow-ssh** and **network-2-allow-icmp** rules for **vpn-network-2**.

These firewall rules allow **SSH** and **ICMP** traffic from anywhere.

### Explore the instances and their connectivity

Currently, the VPN connection between the two networks is not established. Explore the connectivity options between the instances in the networks
1. List instance   
<pre><code> gcloud compute instances list </code></pre>    
      
2. Note the external and internal IP addresses for server-2
3. **SSH** to **server-1**
<pre><code> gcloud compute ssh server-1 --zone "us-central1-b" </code></pre>
     
4. To test connectivity to server-2's external IP address, run the following command, replacing server-2's external IP address with the value noted earlier:
<pre><code> ping -c 3 &lt;Enter server-2's external IP address here&gt; </code></pre>
  
    This works because the VM instances can communicate over the internet.

5. To test connectivity to server-2's internal IP address, run the following command, replacing server-2's internal IP address with the value noted earlier:
<pre><code> ping -c 3 &lt;Enter server-2's internal IP address here&gt; </code></pre>
  
    You should see 100% packet loss when pinging the internal IP address because you don't have VPN connectivity yet.
    
6. Exit the SSH terminal

Let's try the same from **server-2**

7. Note the external and internal IP addresses for **server-1**

8. **SSH** to **server-2**
<pre><code> gcloud compute ssh server-2 --zone "europe-west1-b" </code></pre>
     
9. To test connectivity to server-1's external IP address, run the following command, replacing server-1's external IP address with the value noted earlier:
<pre><code> ping -c 3 &lt;Enter server-1's external IP address here&gt; </code></pre>
 
10. To test connectivity to server-1's internal IP address, run the following command, replacing server-1's internal IP address with the value noted earlier:
<pre><code> ping -c 3 &lt;Enter server-1's internal IP address here&gt; </code></pre>
  
    You should see similar results.
  
11. Exit the SSH terminal
<pre><code> 
Why are we testing both **server-1** to **server-2** and **server-2** to **server-1**?

For the purposes of this lab, the path from subnet-a to subnet-b is not the same as the path from subnet-b to subnet-a. You are using one tunnel to pass traffic in each direction. And if both tunnels are not established, you won't be able to ping the remote server on its internal IP address. The ping might reach the remote server, but the response can't be returned.

This makes it much easier to debug the lab during class. In practice, a single tunnel could be used with symmetric configuration. However, it is more common to have multiple tunnels or multiple gateways and VPNs for production work, because a single tunnel could be a single point of failure.</code></pre>


## Task 2: Create the VPN gateways and tunnels

Establish private communication between the two VM instances by creating VPN gateways and tunnels between the two networks

### Reserve two static IP addresses

Reserve one static IP address for each VPN gateway.
<pre><code> gcloud compute addresses create vpn-1-static-ip  --region "us-central1" </code></pre>

Repeat the same for **vpn-2-static-ip**
<pre><code> gcloud compute addresses create vpn-2-static-ip  --region "europe-west1" </code></pre>

Note both IP addresses for the next step. 

### Create the vpn-1 gateway and tunnel1to2
<pre><code>
gcloud compute target-vpn-gateways create "vpn-1" --region "us-central1" --network "vpn-network-1"

gcloud compute forwarding-rules create "vpn-1-rule-esp" --region "us-central1" --address "35.223.239.47" --ip-protocol "ESP" --target-vpn-gateway "vpn-1"

gcloud compute forwarding-rules create "vpn-1-rule-udp500" --region "us-central1" --address "35.223.239.47" --ip-protocol "UDP" --ports "500" --target-vpn-gateway "vpn-1"

gcloud compute forwarding-rules create "vpn-1-rule-udp4500" --region "us-central1" --address "35.223.239.47" --ip-protocol "UDP" --ports "4500" --target-vpn-gateway "vpn-1"

gcloud compute vpn-tunnels create "tunnel1to2" --region "us-central1" --peer-address "35.189.225.245" --shared-secret "gcprocks" --ike-version "2" --local-traffic-selector "0.0.0.0/0" --target-vpn-gateway "vpn-1"

gcloud compute routes create "tunnel1to2-route-1" --network "vpn-network-1" --next-hop-vpn-tunnel "tunnel1to2" --next-hop-vpn-tunnel-region "us-central1" --destination-range "10.1.3.0/24"
</code></pre>

### Create the vpn-2 gateway and tunnel2to1
<pre><code>
gcloud compute target-vpn-gateways create "vpn-2" --region "europe-west1" --network "vpn-network-2"

gcloud compute forwarding-rules create "vpn-2-rule-esp" --region "europe-west1" --address "35.189.225.245" --ip-protocol "ESP" --target-vpn-gateway "vpn-2"

gcloud compute forwarding-rules create "vpn-2-rule-udp500" --region "europe-west1" --address "35.189.225.245" --ip-protocol "UDP" --ports "500" --target-vpn-gateway "vpn-2"

gcloud compute forwarding-rules create "vpn-2-rule-udp4500" --region "europe-west1" --address "35.189.225.245" --ip-protocol "UDP" --ports "4500" --target-vpn-gateway "vpn-2"

gcloud compute vpn-tunnels create "tunnel2to1" --region "europe-west1" --peer-address "35.223.239.47" --shared-secret "gcprocks" --ike-version "2" --local-traffic-selector "0.0.0.0/0" --target-vpn-gateway "vpn-2"

gcloud compute routes create "tunnel2to1-route-1" --network "vpn-network-2" --next-hop-vpn-tunnel "tunnel2to1" --next-hop-vpn-tunnel-region "europe-west1" --destination-range "10.5.4.0/24"
</code></pre>

## Task 3: Verify VPN connectivity

Verify server-1 to server-2 connectivity

1. **SSH** to **server-1**: 
<pre><code> gcloud compute ssh server-1 --zone "us-central1-b" </code></pre>

2. To test connectivity to server-2's internal IP address, run the following command:
<pre><code> ping -c 3 &lt;Enter server-2's internal IP address here&gt; </code></pre>
  
3. Exit the SSH terminal

4. **SSH** to **server-2**: 
<pre><code> gcloud compute ssh server-2 --zone "europe-west1-b" </code></pre>
     
5. To test connectivity to server-1's internal IP address, run the following command:
<pre><code> ping -c 3 &lt;Enter server-1's internal IP address here&gt; </code></pre>
  
### Remove the external IP addresses

Now that you verified VPN connectivity, you can remove the instances' external IP addresses. For demonstration purposes, just do this for the **server-1** instance.
<pre><code> gcloud compute instances delete-access-config server-1 --access-config-name "External NAT" --zone "us-central1-b" </code></pre>

6. Feel free to **SSH** to **server-2** and verify that you can still ping the **server-1** instance's internal IP address
<pre><code> gcloud compute ssh server-2 --zone "europe-west1-b" </code></pre>
<pre><code> ping -c 3 &lt;Enter server-1's internal IP address here&gt; </code></pre>

## Task 4: Review
In this lab, you configured a VPN connection between two networks with subnets in different regions. Then you verified the VPN connection by pinging VMs in different networks using their internal IP addresses.

You configured the VPN gateways and tunnels using the Cloud Shell. 
