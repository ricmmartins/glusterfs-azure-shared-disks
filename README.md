# Setting-up a GlusterFS using Azure Shared Disks on Ubuntu Linux 18.04
A guide on howto create a redundant storage pool using GlusterFS using Azure Shared Disks 


Build your workload in cloud doesn't mean that your workload never will fail. When you start to draft your architecture, consider that all can fail. This is what will ensure your availability. Keep in mind: 

> "Design for failure and nothing fails."

So here I'll show to you how to create a redundtant storage pool using [GlusterFS](https://www.gluster.org/) and [Azure Shared Disks](https://docs.microsoft.com/en-us/azure/virtual-machines/disks-shared). GlusterFS is a network-attached storage filesystem that allows you to pool storage resources of multiple machines. Azure shared disks is a new feature for Azure managed disks that allows you to attach a managed disk to multiple virtual machines (VMs) simultaneously. 

Our setup will consists in:

* An Azure Resource Group containing the resources
   * An Azure VNET and a Subnet
   * An Availability Set into a Proximity Placement Group
   * 2 Linux VMs (Ubuntu 18.04)
      * 2 Public IPs (one for each VM)
      * 2 Network Security Groups (1 per VM Network Interface Card)
   * A Shared Data Disk attached to the both VMs


 






