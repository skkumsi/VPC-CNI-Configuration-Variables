### **What are the best practices to configure AWS VPC CNI plugin so that the IP addresses are used efficiently.**

### **Short description**

The Amazon VPC Container Network Interface (CNI) plugin for Kubernetes is deployed to each of your Amazon EKS EC2 worker nodes in a Daemonset with the name 'aws-node'. The plugin consists of two primary components: 
- **L-IPAMD**: Responsible for creating network interfaces and attaching the network interfaces to Amazon EC2 instances, assigning secondary IP addresses to network interfaces, and maintaining a warm pool of IP addresses. 
- **CNI plugin:** – Responsible for wiring the host network (for example, configuring the network interfaces and virtual Ethernet pairs) and adding the correct network interface to the pod namespace.

For our topic today lets focus on ***L-IPAMD***, a.k.a Local IP Address Manager Daemon and how it can be configured to efficiently allocate IPs to the nodes when required.

### **What exactly does L-IPAMD do?**
- The CNI binary is invoked by the kubelet when a new pod gets scheduled or an existing pod is removed from the node.
- The CNI binary then calls L-IPAMD, asking for an IP for the new pod. If there are no available IPs in the cache, it will return an error.
- L-IPAMD keeps track of all ENIs and IPs attached to the instance.
- When the number of pods running on the node exceeds the number of addresses that can be assigned to a single network interface, L-IPAMD starts allocating a new network interface, as long as the maximum number of network interfaces for the instance aren't already attached. 

    **Note**: For more information about how many network interfaces and private IP addresses are supported for each network interface, please refer to documentation [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI).

- There are configuration variables that allow you to change the default behavior for when the plugin creates new network interfaces or attaches additional secondary private IPv4 addresses and they are namely ***WARM_ENI_TARGET***, ***WARM_IP_TARGET***, ***MINIMUM_IP_TARGET*** and the newly added ***WARM_PREFIX_TARGET*** which is required when using [VPC CNI prefix delegation](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html) feature.
- The number of IPs maintained in the warm pool are controlled by these variables.

#### **WARM_ENI_TARGET:** 
-  This variable dictates how many ENIs the L-IPAMD should keep available in its warm pool, so that the pods when scheduled on that node can immediately get an IP assigned to it.
   
    **Example:** For a t3.medium type instance, the maximum number of network interfaces that can be attached is '3' and each interface can have about '6' private IPv4 addresses assigned to it. So a total of 18 IPs can be allocated for a given EC2 instance. 
    
    **Note:** Out of the 18 IPs, one IP is assigned to the Primary network interface of the node and the remaining '17' IPs are available to be assigned to kubernetes pods.

- For instance when you set WARM_ENI_TARGET=1, it essentially means that there will always be an additional network interface along with its IPs (count 6) be attached to the node as long as the maximum number of network interfaces for the instance aren't already attached. Therefore a total of 6 IPs (for a t3.medium type instance) are always made available in the I-PAMD's warm pool.
- Even if '1' out of '6' IPs from the ENI gets assigned to a pod, L-IPAMD attaches a new ENI to the node to satisfy the condition of making a full ENIs' IPs available in the warm pool.

#### **WARM_IP_TARGET:** 
- Setting this variable ensures that you always have the defined number of IPs available in the L-IPAMD's warm pool.
- For instance, setting WARM_IP_TARGET=5 indicates that there will always be '5' IP addresses available in the warm pool to assign to the pods immediately as soon as they get scheduled on that node.
- For the same instance type (t3.medium) discussed above which has '3' pods running on it, assuming WARM_ENI_TARGET is not set, to satisfy the '5' available IPs in warm pool condition, L-IPAMD attaches one more ENI to the node and assigns 2 IPv4 addresses to it.

#### **MINIMUM_IP_TARGET:**
- Setting this variable ensures that a minimum of defined number of IPs are assigned to the node when it comes up initially. This value also includes the *WARM_IP_TARGET* value if configured. It is generally used in conjunction with WARM_IP_TARGET.
- For instance, setting a MINIMUM_IP_TARGET=4 indicates that, when the node comes up for the first time it should have 4 IPs assigned to its ENI by default, even when there are no pods running on the node.
- If a WARM_IP_TARGET=1 is set along with the MINIMUM_IP_TARGET=4, with no pods running on the node, still a warm pool of 4 IPs is maintained by L-IPAMD as MINIMUM_IP_TARGET also takes into account the value set by WARM_IP_TARGET. So at this point you have a total of '4' IPv4 addresses assigned to the ENI.
- Now, if you provision '5' pods on the above discussed node, the L-IPAMD tries to assign one more IPv4 address to the ENI to satisfy the WARM_IP_TARGET of '1'.
- The MINIMUM_IP_TARGET comes into picture only during the initial stages of a node when it comes up for the first time. Once the MINIMUM_IP_TARGET number of pods are provisioned, L-IPAMD only tries to satisfy WARM_IP_TARGET condition. 

#### **WARM_PREFIX_TARGET** (available for CNI version 1.9.0+):
- This variable is set when CNI Prefix Delegation feature is enabled for your cluster.
- After enabling this feature, instead of assigning secondary IPv4 addresses to the ENI, a /28 prefix (16 continuous IP address block) is added to the ENI, thereby increasing the pod density on a given node.
- Setting this variable ensures that you have a defined number of prefixes (/28 CIDR blocks) added to the instance's network interface at all times.
- For instance, setting WARM_PREFIX_TARGET=1 indicates that by default there will be one prefix added to the network interface when there are no pods running. When you start provisioning the first few pods, a second prefix gets added to the network interface to satisfy the WARM_PREFIX_TARGET condition. 
- For a t3.medium node with prefix delegation feature enabled, instead of adding 6 IPv4 addresses to a single ENI, '5' prefixes (# of IPv4s per ENI - 1) which is a total of 5 * 16 = 80 IPv4 addresses can be assigned, thereby increasing the pod density on the node.


### Best practices to follow when setting these configuration variables for efficient usage of IP addresses in your Subnet:

- When setting WARM_ENI_TARGET, be aware of the worker node instance type, its Maximum network interfaces and Private IPv4 addresses per interface. 

    **Example:** Setting WARM_ENI_TARGET=3 for m5.xlarge node implies that we are requesting L-IPAMD to assign 3 ENIs to the node at all times, thereby assigning 45 IPs (15 IPs per ENI). These 45 IPs are now blocked by this node and cannot be used for pods that are getting scheduled on other worker nodes. This can lead to quick exhaustion of IPs in the subnet.

- However, if you do expect your application to scale out drastically, setting WARM_ENI_TARGET might be a suitable option to quickly accommodate the newly scheduled pods.

- For clusters with low pod churn try to use WARM_IP_TARGET, so that only the required number of IPs are assigned to the network interface instead of blocking the ENIs' full IPs.

- Also it is not advised to use WARM_IP_TARGET for large clusters as it will add a lot more EC2 calls eventually resulting is API throttling issues.

- If you know the minimum number of pods that you are going to run per node, try to use MINIMUM_IP_TARGET, so that the required number of IPs are readily assigned and available on the node and can be given to the pod as soon as it gets scheduled without any delays. Set this variable along with the WARM_IP_TARGET, to make sure there are extra free IPs available on the node for future pods.

- When using the VPC CNI's Prefix Delegation feature, always make sure WARM_PREFIX_TARGET variable is set to a value greater than or equal to '1'. If set to '0' you will run into following error.

    > **Error:**
    Setting WARM_PREFIX_TARGET = 0 is not supported while WARM_IP_TARGET/MINIMUM_IP_TARGET is not set. Please configure either one of the WARM_{PREFIX/IP}_TARGET or MINIMUM_IP_TARGET env variables

- For smaller subnets, make use of WARM_IP_TARGET along with WARM_PREFIX_TARGET when using VPC Prefix delegation feature to avoid over allocating prefixes and thereby its IPs to the ENI. You could very quickly exhaust your free IPs when using this feature with small subnets.





