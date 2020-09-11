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

### Create the firewall rule to allow traffic from any sources in the 10.10.0.0/16 range
Create a firewall rule to allow traffic in the 10.10.0.0/16 subnet.

Explore existing firewall rules: 
<pre><code> gcloud compute firewall-rules list </code></pre>

Create a firewall rule to allow traffic in the 10.10.0.0/16 subnet: 
<pre><code> gcloud compute firewall-rules create fw-allow-lb-access --direction=INGRESS --priority=1000 --network=my-internal-app --action=ALLOW --rules=all --source-ranges=10.10.0.0/16 --target-tags=backend-service  </code></pre>

### Create the health check rule
Create a firewall rule to allow health checks:
<pre><code> gcloud compute firewall-rules create fw-allow-health-checks --direction=INGRESS --priority=1000 --network=my-internal-app --action=ALLOW --rules=tcp:80 --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=backend-service </code></pre>


## Task 2: Create a NAT configuration using Cloud Router
The Google Cloud VM backend instances that you setup in Task 3 will not be configured with external IP addresses.

Instead, you will setup the Cloud NAT service to allow these VM instances to receive incoming traffic only through the Cloud NAT, and return traffic from the outbound NAT through the load balancer.

### Create the Cloud Router instance
Create a Cloud Router:
<pre><code> gcloud compute routers create nat-router-us-central1 --network "my-internal-app" --region "us-central1" </code></pre>

Add a configuration to the router:
<pre><code> gcloud compute routers nats create nat-config --router "nat-router-us-central1" --nat-all-subnet-ip-ranges --auto-allocate-nat-external-ips --region=us-central1 </code></pre>


## Task 3. Configure instance templates and create instance groups
A managed instance group uses an instance template to create a group of identical instances. Use these to create the backends of the internal load balancer.

### Configure the instance templates
An instance template is an API resource that you can use to create VM instances and managed instance groups. Instance templates define the machine type, boot disk image, subnet, labels, and other instance properties. Create an instance template for both subnets of the **my-internal-app** network

**For subnet-a:**
<pre><code> gcloud compute instance-templates create instance-template-1 --machine-type=e2-medium --region=us-central1 --subnet=subnet-a --no-address --metadata=startup-script-url=gs://cloud-training/gcpnet/ilb/startup.sh --tags=backend-service </code></pre>

<pre><code> 
The **startup-script-url** specifies a script that is executed when instances are started. 
This script installs Apache and changes the welcome page to include the client IP and the name, region, and zone of the VM instance 
</code></pre>

<pre><code>  The network tag backend-service ensures that the firewall rule to allow traffic from any sources in the 10.10.0.0/16 subnet and the Health Check firewall rule applies to these instances. </code></pre>

**For subnet-b:**
<pre><code> gcloud compute instance-templates create instance-template-2 --machine-type=e2-medium --region=us-central1 --subnet=subnet-b --no-address --metadata=startup-script-url=gs://cloud-training/gcpnet/ilb/startup.sh --tags=backend-service </code></pre>

### Create the managed instance groups
Create a managed instance group in **subnet-a** (us-central1-a) and **subnet-b** (us-central1-b).

**For subnet-a:**
<pre><code> gcloud compute instance-groups managed create instance-group-1 --base-instance-name=instance-group-1 --template=instance-template-1 --size=1 --zone=us-central1-a  </code></pre>
<pre><code> gcloud compute instance-groups managed set-autoscaling "instance-group-1" --zone "us-central1-a" --target-cpu-utilization "0.8" --cool-down-period "45" --min-num-replicas "1" --max-num-replicas "5" --mode "on" </code></pre>

<pre><code> Managed instance groups offer autoscaling capabilities that allow you to automatically add or remove instances from a managed instance group based on increases or decreases in load. Autoscaling helps your applications gracefully handle increases in traffic and reduces cost when the need for resources is lower. Just define the autoscaling policy, and the autoscaler performs automatic scaling based on the measured load. </code></pre>

Repeat the same procedure for instance-group-2 in us-central1-b:

**For subnet-b:**
<pre><code> gcloud compute instance-groups managed create instance-group-2 --base-instance-name=instance-group-2 --template=instance-template-2 --size=1 --zone=us-central1-b </code></pre>

<pre><code> gcloud compute instance-groups managed set-autoscaling "instance-group-2" --zone "us-central1-b" --target-cpu-utilization "0.8" --cool-down-period "45" --min-num-replicas "1" --max-num-replicas "5" --mode "on" </code></pre>

### Verify the backends
Verify that VM instances are being created in both subnets:
<pre><code> gcloud compute instances list </code></pre>   

Notice two instances that start with instance-group-1 and instance-group-2.
These instances are in separate zones, and their internal IP addresses are part of the **subnet-a** and **subnet-b** CIDR blocks.

Create a utility VM to access the backends' HTTP sites:
<pre><code> gcloud compute instances create utility-vm --zone=us-central1-f --machine-type=f1-micro --subnet=subnet-a --private-network-ip=10.10.20.50 --no-address </code></pre> 

| :memo: | Note that the internal IP addresses for the backends are 10.10.20.2 and 10.10.30.2. If these IP addresses are different, replace them in the two curl commands below |
|--------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------|


**SSH** to **utility-vm:**
<pre><code> gcloud compute ssh utility-vm --zone "us-central1-f" </code></pre> 

To verify the welcome page for instance-group-1-xxxx, run the following command:
<pre><code> curl 10.10.20.2 </code></pre> 
 
To verify the welcome page for instance-group-2-xxxx, run the following command:
<pre><code> curl 10.10.30.2 </code></pre>
 
Exit the SSH terminal


## Task 4. Configure the internal load balancer
Configure the internal load balancer to balance traffic between the two backends (**instance-group-1** in us-central1-a and **instance-group-2** in us-central1-b), as illustrated in the network diagram:
![alt text](https://github.com/eddUG/Google-Cloud-Practice-Project/blob/master/images/load.balancer.png)

### Create load balancer
1. Create a health check:
<pre><code> gcloud compute health-checks create tcp my-lib-health-check --port=80 </code></pre>

2. Define the backend service with the **gcloud compute backend-services create** command.
<pre><code> gcloud compute backend-services create my-ilb-backend-service  --load-balancing-scheme=internal --protocol=TCP --health-checks=my-ilb-health-check --region=us-central1 </code></pre>

<pre><code> 
Defining **--load-balancing-scheme=internal** makes this load balancer internal. 
This choice requires the backends to be in a single region (us-central1) and does not allow offloading TCP processing to the load balancer </code></pre>

<pre><code> 
Health checks determine which instances can receive new connections. 
This HTTP health check polls instances every 5 seconds, waits up to 5 seconds for a response, and treats 2 successful or 2 failed attempts as healthy or unhealthy, respectively. </code></pre>

3. Add backends to the backend service with the **gcloud compute backend-services add-backend** command.
<pre><code> gcloud compute backend-services add-backend my-ilb-backend-service --instance-group=instance-group-1 --instance-group-zone=us-central1-a --region=us-central1 </code></pre>
<pre><code> gcloud compute backend-services add-backend my-ilb-backend-service --instance-group=instance-group-2 --instance-group-zone=us-central1-b --region=us-central1 </code></pre>

The backend service monitors instance groups and prevents them from exceeding configured usage
 
4. Configure the frontend: An internal forwarding rule and internal IP address for the frontend of the load balancer 
<pre><code> gcloud compute forwarding-rules create my-ilb-forwarding-rule --backend-service=my-ilb-backend-service --load-balancing-scheme=internal --subnet=subnet-b --address=10.10.30.5 --ports=80 --region=us-central1 </code></pre>

The frontend forwards traffic to the backend.


## Task 5. Test the internal load balancer
Verify that the *my-ilb* IP address forwards traffic to **instance-group-1** in us-central1-a and **instance-group-2** in us-central1-b.

### Access the internal load balancer

1. **SSH** to **utility-vm:**
<pre><code> gcloud compute ssh utility-vm --zone "us-central1-f" </code></pre>

2. To verify that the internal load balancer forwards traffic, run the following command:
<pre><code> curl 10.10.30.5 </code></pre>
 
<pre><code> As expected, traffic is forwarded from the internal load balancer (10.10.30.5) to the backend </code></pre>

3. Run the same command a couple of times:
<pre><code> 
curl 10.10.30.5
curl 10.10.30.5
curl 10.10.30.5
curl 10.10.30.5
curl 10.10.30.5
curl 10.10.30.5
curl 10.10.30.5
curl 10.10.30.5
curl 10.10.30.5
curl 10.10.30.5
</code></pre>

You should be able to see responses from **instance-group-1** in us-central1-a and **instance-group-2** in us-central1-b. If not, run the command again


## Task 6. Review

In this lab, you created two managed instance groups in the us-central1 region and a firewall rule to allow HTTP traffic to those instances and TCP traffic from the Google Cloud health checker. Then you configured and tested an internal load balancer for those instance groups.
 
