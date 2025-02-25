     _________________  _____       _           _              _____ 
    /  ___|  _  \  _  \/  __ \     | |         | |            |____ |                Developed By
    \ `--.| | | | | | || /  \/     | |     __ _| |__    __   __   / /         --------------------------
     `--. \ | | | | | || |         | |    / _` | '_ \   \ \ / /   \ \         Rutger Blom  &  Luis Chanu
    /\__/ / |/ /| |/ / | \__/\  _  | |___| (_| | |_) |   \ V /.___/ /         NSX vExpert     VCDX #246
    \____/|___/ |___/   \____/ (_) \_____/\__,_|_.__/     \_/ \____/ 


## Table of Contents

* [Description](#Description)
* [Requirements](#Requirements)
  * [Recommendations](#Recommendations)
* [Preparations](#Preparations)
* [Networking](#Networking)
* [IP Address Assignments](#IP-Address-Assignments)
* [Usage](#Usage)
* [Known Items](#Known-Items)
* [More Information](#More-Information)
* [Credits](#Credits)


## Description

This repository contains Ansible scripts that perform fully automated deployments of complete nested VMware SDDC Pods. Each Pod contains:
* A [VyOS](https://www.vyos.io/) Router
* vCenter Server
* ESXi Hosts
* NSX-T Local Manager
* NSX-T Edge Nodes
* vRealize Log Insight
* A DNS/NTP Server (multi-Pod)

![Physicaloverview](images/SDDC-Lab-pod2phys.png)

The primary use case is consistent and speedy provisioning of nested VMware SDDC lab environments.

## Requirements
The following are the requirements for successful Pod deployments:

* A physical ESXi host running version 6.7 or higher.
* A virtual machine with a modern version of Ubuntu (used as the Ansible controller)
* The default deployment settings require DNS name resolution. You can leverage an existing DNS server, but it must be configured with the required forward and reverse zones and support dynamic updates.
* Access to VMware product installation media.
* For deploying NSX-T you will need an NSX-T license (Check out [VMUG Advantage](https://www.vmug.com/membership/vmug-advantage-membership) or the [NSX-T Product Evaluation Center](https://my.vmware.com/web/vmware/evalcenter?p=nsx-t-eval)).
* If IPv6 deployment is enabled (Deploy.Setting.IPv6 = True):
  * Pod.BaseNetwork.IPv6 must be a fully expanded /56 IPv6 network prefix.  By default, [RFC4193](https://tools.ietf.org/html/rfc4193) ULA fd00::/56 prefix is used as a placeholder.
  * Router Version should be set to "Latest" (default)
  * It is recommended that the physical layer-3 switch be configured with OSPFv3 enabed on the Lab-Routers segment
  * The Ansible controller must be IPv6 enabled, and have IPv6 transit to the DNS server
  * DNS server must be IPv6 enabled
  * DNS server must have IPv6 forward and reverse zones
  * Within each Pod, only the following components are currently configured with IPv6:
    * Nested VyOS Router (All interfaces)
    * NSX-T Segments
    * NSX-T eBGP Peering with the Router

### Recommendations
The following are recommendations based on our experience with deploying Pods:

* Use a physical layer-3 switch with appropriate OSPF/BGP configuration matching the OSPF/BGP settings in your config.yml file. Dynamic routing between your Pods and your physical network will make for a better experience.
* Hardware configuration of the physical ESXi host(s):
  * 2 CPUs (10 cores per CPU)
  * 320 GB RAM
  * 1 TB storage capacity (preferably SSD). Either DAS or 10 Gbit NFS/iSCSI.  More space required if multiple labs are deployed.
* Virtual hardware configuration of the Ansible controller VM:
  * 1 vCPU (4 vCPUs recommended)
  * 8 GB RAM (16GB RAM recommended)
  * Hard disk
    * 64 GB for Linux boot disk
    * 300 GB for /Software repository (Recommend this be on it's own disk)
  * VMware Paravirtual SCSI controller
  * VMXNET 3 network adapter
* Deploy the pre-configured DNS server for DNS name resolution within Pods instead of using your own.

## Preparations

* Configure your physical network:
  * Create an Lab-Routers VLAN used as transit segment between your layer-3 switch and the Pod [VyOS](https://www.vyos.io/) router.
  * Connfigure routing (OSPFv2/OSPFv3/BGP/static) on the Lab-Routers segment.
  * Add the Pod VLANs to your layer-3 switch in case you are deploying the Pod to a vSphere cluster. 

* Install the required software on your Ansible controller:
  * sudo apt install python3 python3-pip xorriso git
  * sudo pip3 install ansible pyvim pyvmomi netaddr jmespath dnspython
  * ansible-galaxy collection install community.general community.vmware ansible.posix
  * git clone https://github.com/rutgerblom/SDDC.Lab.git 

* Copy/rename the sample files:
  * cp config_sample.yml config.yml
  * cp licenses_sample.yml licenses.yml
  * cp software_sample.yml software.yml
  * cp templates_sample.yml templates.yml

* Modify **config.yml** and **licenses.yml** according to your needs and your environment

* Create the Software Library directory structure using:
  * sudo ansible-playbook utils/util_CreateSoftwareDir.yml

* Add installation media to the corresponding directories in the Software Library (/Software)

## Upgrade Considerations
Consider the following when upgrading SDDC.Lab to a newer version.

* v2 to v3
  * Clone the v3 branch to its own directory. For example: git clone https://github.com/rutgerblom/SDDC.Lab.git SDDC.Lab_v3
  * As additional PIP and Ansible modules are required by v3, please follow the instructions in the "Preparations" section to ensure all of the required software is installed on the Ansible controller.
  * Use copies of the v3 sample files and update these with your settings. Don't copy any v2 files into the v3 directory.
  * Remove the VyOS ISO file from your software library and let the router deployment script download the latest version of the rolling release.


## Networking
The network configuration is where many users experience issues with the setup of the SDDC.Lab solution.  For that reason, the focus of this section is to give a deep dive into how the SDDC.Lab solution "connects" to the physical network, and what networking components it requires.  We will also give overviews of how the network connectivity is different if you're running:
* 1 Physical Server
* 2 Physical Servers
* 3 (or more) Physical Servers

### Logical Networking Overview
Before we dive into the physical network environment, it's important to understand logically how everything is configured and connected.  This is the **KEY** to understanding how SDDC.Lab works from a networking perspective.

Each SDDC.Lab that is deployed is referred to as a Pod.  Every Pod is assigned a number between 10-240 which is evenly divisble by 10.  So, valid Pod numbers include 10, 20, 30, ..., 220, 230, and 240.  The Pod number drives **ALL** networking elements used by that lab, includng, VLAN IDs, IP networks (IPv4 and IPv6), and Autonomous System Numbers (ASNx).  This ensures that no duplicate networking components exist between any of the Pods.

At the heart of each Pod is a software-based [VyOS](https://vyos.io/) router, which we call the Pod-Router.  The Pod-Router provides these main functions:
1. Connectivity to the physical environment
2. The gateway interfaces for the various SDDC.Lab networks

Connectivity to the physical environment is achieved via the Pod-Router's eth0 interface.  This interface is a an untagged interface, and is connected to the "Lab-Routers" portgroup (discussed later).  It's over this interface that the Pod-Router peers with other deployed Pods and the physical environment.

The Pod-Router provides gateways services for all of the SDDC.Lab's networks via it's eth1 interface.  Eth1 is configured as a tagged interface, and a unique layer-3 sub-interface (VyOS calls them vif's) is created for each of the SDDC.Lab networks.  These sub-interfaces act as the IPv4/IPv6 gateway for its respective SDDC.Lab network.

Each Pod is comprised of ten (10) SDDC.Lab networks, numbered 0 through 9.  These SDDC.Lab network numbers are added to the Pod number to create unique VLAN IDs for each Pod.  This explains why the Pod Numbers are divisible by 10.  Below are the SDDC.Lab networks that are deployed within each Pod (Pod Number 100 shown):

| Pod.Number | Network Number | VLAN ID |    Description    |
|------------|----------------|---------|-------------------|
|    100     |       0        |   100   | Management (ESXi, NSX-T Edges, etc.) |
|    100     |       1        |   101   | vMotion |
|    100     |       2        |   102   | vSAN |
|    100     |       3        |   103   | IPStorage |
|    100     |       4        |   104   | Overlay Transport (i.e. GENEVE Traffic) |
|    100     |       5        |   105   | Service VM Management Interfaces |
|    100     |       6        |   106   | NSX-T Edge Uplink #1 |
|    100     |       7        |   107   | NSX-T Edge Uplink #2 |
|    100     |       8        |   108   | Remote Tunnel Endpoint (RTEP) |
|    100     |       9        |   109   | VM Network |

In order to be able to deploy multiple Pods, VLAN ID's 10-249 should be reserved for SDDC.Lab use.

Here is a network diagram showing the Pod Logical Networking described above:
![PodLogicalNetworkingOverview](images/Pod_Logical_Networking_Overview_001.png)

### Physical Networking Overview
When we refer to physical networking, we are referring to the "NetLab-L3-Switch" in the network diagram shown above, along with the "Lab-Routers" portgroup to which it connects.  The "Lab-Routers" portgroup is the central "meet-me" network segment which all Pods connect to.  It's here where the various Pods establish OSPFv2, OSPFv3, and BGP neighbor peerings and share routes.  Because OSPF uses multicast to discovery neighbors, there is no additional configuration required for it.  BGP on the other hand, by default, is only configured to peer with the "NetLab-L3-Switch".  This behavior can be changed by adding additional neighbors to the Pod Configuration file, but that is left up to the user to setup.  The routing protocols configured on the "NetLab-L3-Switch" should be configured to originate and advertise the default route to the [VyOS](https://vyos.io/) routers on the "Lab-Routers" segment.

To ensure VLAN IDs created on the "NetLab-L3-Switch" do not pollute other networking environments, and to protect the production network from the SDDC.Lab environment, it's **HIGHLY** recommended that the following guidelines are followed:
1. The VLAN IDs used by the SDDC.Lab environment (i.e. VLANs 10-249) should not be stretched to other switches outside of the SDDC.Lab environment.  Those VLANs should be local to the SDDC.Lab.
2. The uplink from the "NetLab-L3-Switch" to the core network should be a **ROUTED** connection.
3. Reachability between the SDDC.Lab environment and the Core Network should be via static routes, and **NOT** use any dynamic routing protocol.  This will ensure that SDDC.Lab routes don't accidentally "leak" into the production network.

All interfaces, both virtual and physical, that are connected to the "Lab-Routers" segment should be configured with a 1500 byte MTU.  This ensures that OSPF neighbors can properly establish peering relationships.  That said, the "NetLab-L3-Switch" must be configured to support jumbo mtu frames, as multiple SDDC.Lab segments require jumbo frame support.  Jumbo frame size of 9000 (or higher) is suggested.

### Physical Network Considerations - One ESXi Server
When only one physical ESXi server is being used to run Pod workloads, as all workloads will be running on the same vSwitch on that host, there is no requirement to configure an Uplink on the SDDCLab_vDS switch to the physical environment.

### Physical Network Considerations - Two ESXi Servers
When exactly two physical ESXi servers are being used to run Pod workloads, you can use a cross-over cable to connect the SDDCLab_vDS vswitches together via their Uplinks.  This cable between the two servers to connect the SDDCLab_vDS Uplink interface from each server.

### Physical Network Considerations - Three or more ESXi Servers
When three or more physical ESXi servers are being used to run Pod workloads, you have two options:
1. Use a single "NetLab-L3-Switch" and connect all servers to it (Suggested)
2. If the number of available ports on the "NetLab-L3-Switch" is limited, you can use two switches as is shown in the Pod Logical Networking Overview above.  In this configuration, a layer-2 only switch is used for the SDDCLab_vDS vswitch, and a layer-3 switch is used to connect to the "Lab-Routers" segment.

## IP Address Assignments
When a Pod is deployed, various components are deployed as part of that Pod.  Each of those components are connected to the Pod's Management subnet.  Here is a listing of those components along with their respective host IP address:

| IPv4 Address | Component | Description | Part of Default Deployment |
|-------------------|-----------|-------------|----------------------------|
| 1 | Gateway | VyOS Router | Yes |
| 2 | Reserved | Reserved for Future Use | |
| 3 | Reserved | Reserved for Future Use | |
| 4 | Reserved | Reserved for Future Use | |
| 5 | vCenter | vCenter Appliance | Yes |
| 6 | vRLI | vRealize Log Insight Appliance | Yes |
| 7 | GM VIP | NSX-T Global Manager VIP (Future) | No |
| 8 | GM1 | NSX-T Global Manager Node 1 (Future) | No |
| 9 | GM2 | NSX-T Global Manager Node 2 (Future) | No |
| 10 | GM3 | NSX-T Global Manager Node 3 (Future) | No |
| 11 | LM VIP | NSX-T Local Manager VIP | No |
| 12 | LM1 | NSX-T Local Manager Node 1 | Yes |
| 13 | LM2 | NSX-T Local Manager Node 2 | No |
| 14 | LM3 | NSX-T Local Manager Node 3 | No |
| 15 | CSM | NSX-T Cloud Services Manager | No |
| 16 | vRNI Platform | vRealize Network Insight Platform Appliance | No |
| 17 | vRNI Collector | vRealize Network Insight Collector Node | No |
| 18 | Reserved | Reserved for Future Use | |
| 19 | Reserved | Reserved for Future Use | |
| 20 | Reserved | Reserved for Future Use | |
| 21 | Mgmt-1 | Nested ESXi Host in Management Cluster | No |
| 22 | Mgmt-2 | Nested ESXi Host in Management Cluster | No |
| 23 | Mgmt-3 | Nested ESXi Host in Management Cluster | No |
| 31 | ComputeA-1 | Nested ESXi Host in Compute-A Cluster | Yes |
| 32 | ComputeA-2 | Nested ESXi Host in Compute-A Cluster | Yes |
| 33 | ComputeA-3 | Nested ESXi Host in Compute-A Cluster | Yes |
| 91 | Edge-1 | Nested ESXi Host in Edge Cluster | Yes |
| 92 | Edge-2 | Nested ESXi Host in Edge Cluster | Yes |
| 93 | Edge-3 | Nested ESXi Host in Edge Cluster | Yes |
| 253 | EdgeVM-02 | NSX-T Edge Transport Node 2 | Yes |
| 254 | EdgeVM-01 | NSX-T Edge Transport Node 1 | Yes |

## Usage

To deploy a Pod:
1. Generate a Pod configuration with:  
**ansible-playbook playbooks/createPodConfig.yml**

1. Start a Pod deployment per the instructions. For example:  
**sudo ansible-playbook -e "@/home/ubuntu/Pod-230-Config.yml" deploy.yml**

Deploying an SDDC Pod will take somewhere between 1 and 1.5 hours depending on your environment and Pod configuration.

Similary you remove a Pod with:  
**sudo ansible-playbook -e "@/home/ubuntu/Pod-230-Config.yml" undeploy.yml**

## Known Items
Here are some known items to be aware of:
1. If you attempt to deploy a pod, and receive a message indicating "Error rmdir /tmp/Pod-###/iso: [Errno 39] Directory not empty: '/tmp/Pod-###/iso'", that's because a previous pod deployment failed (for whatever reason), and some files remained in the /tmp/Pod-### directory.  To resolve this issue, delete the entire /tmp/Pod-### directory, and then re-deploy the Pod.  If an ISO image is still mounted (which you can check by running 'mount'), then you will need to unmount the ISO image before you can delete the /tmp/Pod-### directory.  In all examples, the "###" of Pod-### is the 3-digit Pod Number.

2. The DNS IPv6 reverse zone used is determined by the network used for BaseNetwork.IPv6:\
   a) If it begins with "fd", then the zone used is **d.f.ip6.arpa**\
   b) Otherwise, the zone used is a standard IPv6 reverse DNS zone for the configured /56 network

   This is important understand if you need to configure conditional forwarding to reach your SDDC.Lab environment.

3. SDDC.Lab v3 requires Ansible version 2.10.1 (or later).  Thus, if you are upgrading from SDDC.Lab v2, make 
   sure you upgrade your Ansible to the latest version.  To see your current Ansible version, run the following
   command: ansible --version

## More Information
For detailed installation, preparation, and deployment steps, please see the "[Deploying your first SDDC.Lab Pod](FirstPod.md)" document.

## Credits
A big thank you to [Yasen Simeonov](https://www.linkedin.com/in/yasen/). His project at https://github.com/yasensim/vsphere-lab-deploy was the inspiration for this project. Another big thank you to my companion and lead developer [Luis Chanu](https://www.linkedin.com/in/luischanu/) (VCDX #246) for pushing this project forward all the time. Last but not least thank you vCommunity for trying this out and providing valuable feedback.