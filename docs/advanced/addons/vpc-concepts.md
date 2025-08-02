***Virtual Private Cloud (VPC) - Concepts & Architecture*** 

***Overview*** 

A **Virtual Private Cloud (VPC)** is a logically isolated network that provides full control over IP addressing, subnets, route tables, firewalls, and gateways within a cloud infrastructure. VPC allows the secure and scalable deployment of virtualized resources such as compute, storage, and container services.

The following diagram shows a typical VPC architecture with both public and private subnets to separate internet-facing traffic from internal resources.

*Note: VPC configurations may vary across platforms, but the logical design remains consistent.*


###  VPC → Subnet → Overlay Network → VM: Hierarchical Diagram

#### Component Relationship Table

| Layer | Component         | Example in Harvester        | Description                                                                 |
|-------|-------------------|-----------------------------|-----------------------------------------------------------------------------|
| L3    | VPC               | `vpc-1`                     | Logical network container managing multiple subnets and routing/NAT rules. |
| L3    | Subnet            | `vswitch1-subnet`           | IP segment (CIDR); each subnet is bound to one unique Overlay Network.     |
| L2    | Overlay Network   | `vswitch1`                  | Virtual Layer 2 switch that connects VMs; carries subnet traffic.          |
| L2/L3 | Virtual Machine   | `vm1-vswitch1`              | Attached to an Overlay Network; receives IP/Gateway from its subnet.       |

#### ASCII Diagram

```
                                 [ VPC: vpc-1 ]
                                        │
                  ┌─────────────────────┴─────────────────────┐
                  │                                           │
     [ Subnet: vswitch1-subnet ]                 [ Subnet: vswitch2-subnet ]
       CIDR: 172.20.10.0/24                          CIDR: 172.20.20.0/24
       Gateway: 172.20.10.1                          Gateway: 172.20.20.1
                  │                                           │
     [ Overlay Network: vswitch1 ]               [ Overlay Network: vswitch2 ]
                  │                                           │
         ┌────────┴────────┐                         ┌────────┴────────┐
         │                 │                         │                 │
[VM: vm1-vswitch1] [VM: vm2-vswitch1]                [VM: vm1-vswitch2]
IP: 172.20.10.X     IP: 172.20.10.Y                    IP: 172.20.20.Z



```

This diagram illustrates how VPCs, subnets, overlay networks, and VMs are logically connected in Harvester with Kube-OVN.

## **VPC Components Overview**
We abstract the following elements as key components within a VPC:

|**Component**|**Description**|
| :-: | :-: |
|**VPC**|The top-level network space with a user-defined IP CIDR range|
|**Subnet**|A subdivision of the VPC IP space; can be public or private|
|**Route Table**|Defines traffic routing rules within and outside the VPC|
|**Internet Gateway**|Enables outbound internet access for public subnets|
|**NAT Gateway**|Allows private subnets to access the internet (outbound only)|
|**Security Group**|Virtual firewall that controls inbound/outbound traffic per instance|
|**VPC Peering / VPN**|Optional peering or hybrid connections between different VPCs or on-prem networks|

*Note:You must enable `kubeovn-operator` to deploy Kube-OVN to a Harvester cluster for advanced SDN capabilities such as virtual private cloud (VPC) and subnets for virtual machine workloads.

1. On the Harvester UI, go to **Advanced** > **Add-ons**.

2. Select **kubeovn-operator (Experimental)**, and then select **⋮** > **Enable**.* 

***VPC Creation*** 

***How to Properly Map Subnets to Overlay Networks***

Step 1: Create an Overlay Network

1.Go to the Harvester UI.

2.Navigate to Advanced > Networks.

3.Click “Create” to define a new Overlay Network.

- **Name example: vswitch1**\

- **Type: OverlayNetwork**\

4.Click Create to save.

If you plan to create multiple subnets, create a separate Overlay Network for each one (e.g., vswitch2, vswitch3, etc.).

Step 2: Create a Subnet and Link It to an Overlay Network

1.Go to Virtual Private Cloud > Subnets.

2.Click “Create”.

3.Fill in the Subnet details:

- **Name: vswitch1-subnet**\

- **CIDR: Example 172.20.10.0/24**\

- **Gateway IP: Example 172.20.10.1**\

- **Provider: Select the corresponding Overlay Network from the dropdown (e.g., default/vswitch1)**\

The UI will only show Overlay Networks that have not been used by other subnets — this enforces the 1:1 mapping automatically.

Validation Tips:

- **Once an Overlay Network is assigned to a Subnet, it cannot be reused by another.**\

- **If you try to assign the same Overlay Network to multiple subnets, the system will return an error or prevent selection.**\

User Reminder:
Always ensure that each Subnet is backed by a unique Overlay Network. Mapping multiple Subnets to the same Overlay Network can cause ambiguous routing, traffic collisions, or isolation issues.

- **Every new Subnet should be tied to a dedicated Overlay Network.**\

- **This design also helps maintain clarity when configuring features like NAT, Private Subnet isolation, and VPC Peering.**\





***Test steps:***

**1.Creat Virtual Machine Networks**

Name: vswitch1 ,  vswitch2

Type: OverlayNetwork

***Subnet and Overlay Network: 1:1 Mapping Explanation*** 

In Harvester, when working with Virtual Private Cloud (VPC) networking:

- **An Overlay Network represents a virtual Layer 2 switch that encapsulates and forwards traffic between virtual machines.**\

- **A Subnet defines an IP range (CIDR block) within the VPC — but it must be associated with exactly one Overlay Network to actually carry traffic.**\

Each Subnet must be mapped to only one Overlay Network, and each Overlay Network can be used by only one Subnet.

This 1:1 relationship ensures:

- **Clear and predictable routing behavior**\

- **Isolation between Subnets**\

- **Avoidance of routing conflicts or traffic leakage**\


**2.Creat VPC**

Name: vpc-1

After creation, you’ll have an isolated network space ready for subnet creation. 

**3.Create Subnets** 

| Subnet Name      | CIDR           | Provider          | Gateway IP   |
|------------------|----------------|-------------------|--------------|
| vswitch1-subnet  | 172.20.10.0/24 | default/vswitch1  | 172.20.10.1  |
| vswitch2-subnet  | 172.20.20.0/24 | default/vswitch2  | 172.20.20.1  |



**4.Creat VM**

**Name: vm1-vswitch1**

**Basic**

CPU:1

Memory:2

SSH key:Enter your SSH Key, for example:\
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

**Volumes**

Image:Enter your cloudimg, for example: noble-server-cloudimg-amd64

**Networks**

Network: default/vswitch1

**Advanced Options**
```
users:

`  `- name: ubuntu

`    `groups: [ sudo ]

`    `shell: /bin/bash

`    `sudo: ALL=(ALL) NOPASSWD:ALL

`    `lock\_passwd: false

`    `ssh\_authorized\_keys:

`      `- ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

chpasswd:

`  `list: |

`    `ubuntu:test1234

`  `expire: false

ssh\_pwauth: true
```
**Name: vm2-vswitch1**

**Basic**

CPU:1

Memory:2

SSH key:Enter your SSH Key, for example:\
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

**Volumes**

Image:Enter your cloudimg, for example: noble-server-cloudimg-amd64

**Networks**

Network: default/vswitch1

**Advanced Options**
```
users:

`  `- name: ubuntu

`    `groups: [ sudo ]

`    `shell: /bin/bash

`    `sudo: ALL=(ALL) NOPASSWD:ALL

`    `lock\_passwd: false

`    `ssh\_authorized\_keys:

`      `- ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

chpasswd:

`  `list: |

`    `ubuntu:test1234

`  `expire: false

ssh\_pwauth: true
```
**Name: vm1-vswitch2**

**Basic**

CPU:1

Memory:2

SSH key:Enter your SSH Key, for example:\
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

**Volumes**

Image:Enter your cloudimg, for example: noble-server-cloudimg-amd64

**Networks**

Network: default/vswitch1

**Advanced Options**
```
users:

`  `- name: ubuntu

`    `groups: [ sudo ]

`    `shell: /bin/bash

`    `sudo: ALL=(ALL) NOPASSWD:ALL

`    `lock\_passwd: false

`    `ssh\_authorized\_keys:

`      `- ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

chpasswd:

`  `list: |

`    `ubuntu:test1234

`  `expire: false

ssh\_pwauth: true
```
**Note: Once the VM is running, you will see the Node displaying the NTP server -> 0.suse.pool.ntp.org and the IP address.** 

**5.**

Open the **serial console** of **vm1-vswitch1 (172.20.10.6)** and ping **vm1-vswitch2 (172.20.20.3)**.

It shows: **ping: connect: Network is unreachable.**

**Adds a default route :**
```
#sudo ip route add default via 172.20.10.1 dev enp1s0
```
**note: For any network traffic that doesn't match a more specific route, send it to the gateway 172.20.10.1 using the network interface enp1s0.** 

Open the **serial console** of **vm1-vswitch2 (172.20.20.3)** and ping **vm1-vswitch1 (172.20.10.6)**.

It shows: **ping: connect: Network is unreachable.**

**Adds a default route :**
```
#sudo ip route add default via 172.20.10.1 dev enp1s0
```
**note: For any network traffic that doesn't match a more specific route, send it to the gateway** 172.20.20.1 **using the network interface** enp1s0**.** 

Use vm1-vswitch2 (172.20.20.3) to ping vm1-vswitch1 (172.20.10.6) to verify connectivity. 

Use vm1-vswitch1 (172.20.10.6) to ping vm1-vswitch2 (172.20.20.3) to verify connectivity. 

**If the VM wants to send traffic to an unknown network (not in its local subnet), it will forward that traffic to the specified gateway IP using the specified network interface.** 

vm1-vswitch1 will send traffic via 172.20.10.1 through enp1s0.

vm1-vswitch2 will send traffic via 172.20.20.1 through enp1s0.

**This setup allows traffic to be forwarded properly through their gateways, enabling end-to-end connectivity.** 

\------------------------------------------------------------------------------------------------------------------------
### ***Private Subnet***

- **Purpose**
The main goal of the Private Subnet feature is to provide stronger network isolation within the same VPC by restricting traffic between the private subnet and other subnets. This enhances security by preventing unauthorized access to sensitive or critical resources inside the VPC.

- **Behavior**
When Private Subnet is enabled, the subnet cannot communicate with other subnets in the same VPC by default, even though they belong to the same VPC.
Traffic between the private subnet and other subnets is only allowed if you explicitly add their CIDR blocks to the private subnet’s Allow Subnets list.

- **Practical Effect**

    - **Prevents access from other subnets in the same VPC to the private subnet, enabling fine-grained network segmentation (micro-segmentation).**

    - **Enhances internal security isolation and reduces potential attack surface.**

    - **Allows controlled, selective cross-subnet communication by configuring allowed subnets.**

***Testing Summary:***

1.Enable Private Subnet on vswitch1-subnet.

2.Ping from vm1-vswitch1 to vm1-vswitch2 (different subnet, not allowed) — ping fails.

3.Add 172.20.20.0/24 (vswitch2-subnet) to Allow Subnets of vswitch1-subnet.

4.Ping again — communication succeeds.

In essence, the Private Subnet acts as an internal firewall within the VPC, only permitting cross-subnet traffic when explicitly allowed, thus enforcing stricter security boundaries.

***Test steps:***

- **Open the **VPC** page, go to **vswitch1-subnet -> Edit Config**, and enable the **Private Subnet** setting.**

- **Open the **serial console** of **vm1-vswitch1 (172.20.10.6)** and ping **vm1-vswitch2 (172.20.20.3)**. At this point, the ping fails.**

- **Go back to **vswitch1-subnet -> Edit Config**, and add **172.20.20.0/24** to the **Allow Subnets** field.**

- **Open the **serial console** of **vm1-vswitch1 (172.20.10.6)** again and ping **vm1-vswitch2 (172.20.20.3)**. This time, the ping succeeds.**

- **This verifies that the **Private Subnet** feature is working as expected.**

\------------------------------------------------------------------------------------------------------------------------

***NatOutgoing***

**Test steps:**

**1.Creat Virtual Machine Networks**

Name: vswitch-external

Type: OverlayNetwork

**2.Creat VPC**

Create a subnet within the Virtual Private Cloud named 'ovn-cluster'. 

| Subnet Name      | CIDR           | Provider               | Gateway IP   |
|------------------|----------------|------------------------|--------------|
| external-subnet  | 172.20.30.0/24 | default/vswitch-external | 172.20.30.1 |

**3.Creat VM**

Name: vm-external

**Basic**

CPU:1

Memory:2

SSH key:Enter your SSH Key, for example:\
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

**Volumes**

Image:Enter your cloudimg, for example: noble-server-cloudimg-amd64

**Networks**

Network: default/vswitch-external

**Advanced Options**
```
users:

`  `- name: ubuntu

`    `groups: [ sudo ]

`    `shell: /bin/bash

`    `sudo: ALL=(ALL) NOPASSWD:ALL

`    `lock\_passwd: false

`    `ssh\_authorized\_keys:

`      `- ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

chpasswd:

`  `list: |

`    `ubuntu:test1234

`  `expire: false

ssh\_pwauth: true
```
4\.Open the serial console of vm-external (172.20.30.2) and ping 8.8.8.8.

It shows: ping: connect: Network is unreachable.

Adds a default route :
```
#sudo ip route add default via 172.20.30.1 dev enp1s0
```
Still no response from 8.8.8.8 when pinged again. 

5\.Navigate to the Virtual Private Cloud Page\
Log in to the management interface and go to the Virtual Private Cloud page. Locate the subnet resource named external-subnet.

- **Edit the YAML Configuration**\
  Click on the options menu next to external-subnet and select **Edit YAML**.
- **Update the** natOutgoing **Parameter**\
  In the YAML editor, find the following line : natOutgoing: false\
  Change it to: natOutgoing: true
- **Save the Changes**\
  After reviewing to ensure all other settings are correct, click **Save** or **Update** to apply the changes.

**Verify the Configuration**

Go back to the serial console of the VM named vm-external (IP: 172.20.30.2), and run a ping to 8.8.8.8 to check if the connection is successful. 

\------------------------------------------------------------------------------------------------------------------------

***VPC peering***

- **Enable Private Communication Between VPCs**\
  VPC Peering allows virtual machines in different VPCs to communicate with each other **using private IP addresses**, as if they were on the same internal network.
- **Maintain Network Isolation with Controlled Access**\
  Even though the VPCs can communicate, they are still logically and administratively **isolated**, which is useful for organizing workloads by team, function, or environment (e.g., dev, prod).
- **Improve Performance and Reduce Costs**\
  Since traffic stays within the internal cloud network, it's **faster**, **more secure**, and typically **cheaper** than going over public internet or VPNs.
- **Enhanced Security**\
  Traffic between VPCs via peering doesn't traverse the public internet, reducing exposure and risk. Access can also be tightly controlled with route tables and firewall rules.

**Test steps:**

**1.Creat Virtual Machine Networks**

Name: vswitch3 ,  vswitch4

Type: OverlayNetwork

**2.Creat VPC**

Name: vpcpeer-1 ,vpcpeer-2

After creation, you’ll have an isolated network space ready for subnet creation. 

**3.Create Subnets** 

**vpcpeer-1**

| Subnet Name | CIDR        | Provider         | Gateway IP |
|-------------|-------------|------------------|------------|
| subnet1     | 10.0.0.0/24 | default/vswitch3 | 10.0.0.1   |



**vpcpeer-2**

| Subnet Name | CIDR        | Provider         | Gateway IP |
|-------------|-------------|------------------|------------|
| subnet2     | 20.0.0.0/24 | default/vswitch4 | 20.0.0.1   |


**4.Edit Confic**

**vpcpeer-1**

VPC peering \
| Local Connect IP  | Remote VPC  |
|-------------------|-------------|
| 169.254.0.1/30    | vpcpeer-2   |



Static Routes

| CIDR         | Next Hop IP   |
|--------------|--------------|
| 20.0.0.0/16  | 169.254.0.2  |



**vpcpeer-2**

VPC peering \
| Local Connect IP | Remote VPC    |
|------------------|--------------|
| 169.254.0.2/30   | vpcpeer-1    |



Static Routes

| CIDR         | Next Hop IP   |
|--------------|--------------|
| 10.0.0.0/16  | 169.254.0.1  |

**5.Creat VM**

**Name: vm1-vpcpeer1**

**Basic**

CPU:1

Memory:2

SSH key:Enter your SSH Key, for example:\
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

**Volumes**

Image:Enter your cloudimg, for example: noble-server-cloudimg-amd64

**Networks**

Network: default/vswitch3

**Advanced Options**
```
users:

`  `- name: ubuntu

`    `groups: [ sudo ]

`    `shell: /bin/bash

`    `sudo: ALL=(ALL) NOPASSWD:ALL

`    `lock\_passwd: false

`    `ssh\_authorized\_keys:

`      `- ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

chpasswd:

`  `list: |

`    `ubuntu:test1234

`  `expire: false

ssh\_pwauth: true
```
**Name: vm2-vpcpeer2**

**Basic**

CPU:1

Memory:2

SSH key:Enter your SSH Key, for example:\
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

**Volumes**

Image:Enter your cloudimg, for example: noble-server-cloudimg-amd64

**Networks**

Network: default/vswitch4

**Advanced Options**
```
users:

`  `- name: ubuntu

`    `groups: [ sudo ]

`    `shell: /bin/bash

`    `sudo: ALL=(ALL) NOPASSWD:ALL

`    `lock\_passwd: false

`    `ssh\_authorized\_keys:

`      `- ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxyz123456789abcdefgh... Tony Tseng

chpasswd:

`  `list: |

`    `ubuntu:test1234

`  `expire: false

ssh\_pwauth: true
```
note: An 'Unschedulable' error typically indicates insufficient memory. Please stop other virtual machines before attempting to start this one again. 

**6.**

- Open the serial console of vm1-vpcpeer1 (10.0.0.2) and adds a default route :
```
  #sudo ip route add default via 172.20.10.1 dev enp1s0
```
- Check if 20.0.0.2 can be pinged successfully. 
- Open the serial console of vm1-vpcpeer1 (20.0.0.2) and adds a default route :
```
  #sudo ip route add default via 20.0.0.1 dev enp1s0
```
- Check if 10.0.0.2 can be pinged successfully. 

**VPC Peering allows secure, private, and efficient communication between two separate VPCs without exposing them to the public internet.** 
