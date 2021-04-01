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

I'll be using the [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview) once is fully integrated to Azure and with all modules I need already installed.

## Create SSH key pair
```ssh-keygen -t rsa -b 4096```
 
## Create a resource group
New-AzResourceGroup -Name "myResourceGroup" -Location "EastUS"

## Create virtual network resources
# Create a subnet configuration
$subnetConfig = New-AzVirtualNetworkSubnetConfig `
  -Name "mySubnet" `
  -AddressPrefix 192.168.1.0/24

# Create a virtual network
$vnet = New-AzVirtualNetwork `
  -ResourceGroupName "myResourceGroup" `
  -Location "EastUS" `
  -Name "myVNET" `
  -AddressPrefix 192.168.0.0/16 `
  -Subnet $subnetConfig

# Create a public IP address and specify a DNS name
$pip = New-AzPublicIpAddress `
  -ResourceGroupName "myResourceGroup" `
  -Location "EastUS" `
  -AllocationMethod Static `
  -IdleTimeoutInMinutes 4 `
  -Name "mypublicip01"

# Create an inbound network security group rule for port 22
$nsgRuleSSH = New-AzNetworkSecurityRuleConfig `
  -Name "myNetworkSecurityGroupRuleSSH"  `
  -Protocol "Tcp" `
  -Direction "Inbound" `
  -Priority 1000 `
  -SourceAddressPrefix * `
  -SourcePortRange * `
  -DestinationAddressPrefix * `
  -DestinationPortRange 22 `
  -Access "Allow"

# Create an inbound network security group rule for port 80
$nsgRuleWeb = New-AzNetworkSecurityRuleConfig `
  -Name "myNetworkSecurityGroupRuleWWW"  `
  -Protocol "Tcp" `
  -Direction "Inbound" `
  -Priority 1001 `
  -SourceAddressPrefix * `
  -SourcePortRange * `
  -DestinationAddressPrefix * `
  -DestinationPortRange 80 `
  -Access "Allow"

# Create a network security group
$nsg = New-AzNetworkSecurityGroup `
  -ResourceGroupName "myResourceGroup" `
  -Location "EastUS" `
  -Name "myNetworkSecurityGroup01" `
  -SecurityRules $nsgRuleSSH,$nsgRuleWeb


# Create a virtual network card and associate with public IP address and NSG
$nic = New-AzNetworkInterface `
  -Name "myNic01" `
  -ResourceGroupName "myResourceGroup" `
  -Location "EastUS" `
  -SubnetId $vnet.Subnets[0].Id `
  -PublicIpAddressId $pip.Id `
  -NetworkSecurityGroupId $nsg.Id 






