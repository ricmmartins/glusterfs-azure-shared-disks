# Setting pp a GlusterFS using Azure Shared Disks on Ubuntu Linux 18.04
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
```azurepowershell-interactive
ssh-keygen -t rsa -b 4096
```
 
## Create a resource group
```azurepowershell-interactive
New-AzResourceGroup -Name "myResourceGroup" -Location "EastUS"
```

## Create virtual network resources
### Create a subnet configuration
```azurepowershell-interactive
$subnetConfig = New-AzVirtualNetworkSubnetConfig `
  -Name "mySubnet" `
  -AddressPrefix 192.168.1.0/24
```

### Create a virtual network
```azurepowershell-interactive
$vnet = New-AzVirtualNetwork `
  -ResourceGroupName "myResourceGroup" `
  -Location "EastUS" `
  -Name "myVNET" `
  -AddressPrefix 192.168.0.0/16 `
  -Subnet $subnetConfig
```

### Create a public IP address and specify a DNS name
```azurepowershell-interactive
$pip = New-AzPublicIpAddress `
  -ResourceGroupName "myResourceGroup" `
  -Location "EastUS" `
  -AllocationMethod Static `
  -IdleTimeoutInMinutes 4 `
  -Name "mypublicip01"
```

### Create an inbound network security group rule for port 22
```azurepowershell-interactive
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
```
### Create an inbound network security group rule for port 80
```azurepowershell-interactive
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
```
### Create a network security group
```azurepowershell-interactive
$nsg = New-AzNetworkSecurityGroup `
  -ResourceGroupName "myResourceGroup" `
  -Location "EastUS" `
  -Name "myNetworkSecurityGroup01" `
  -SecurityRules $nsgRuleSSH,$nsgRuleWeb
```
### Create a virtual network card and associate with public IP address and NSG
```azurepowershell-interactive
$nic = New-AzNetworkInterface `
  -Name "myNic01" `
  -ResourceGroupName "myResourceGroup" `
  -Location "EastUS" `
  -SubnetId $vnet.Subnets[0].Id `
  -PublicIpAddressId $pip.Id `
  -NetworkSecurityGroupId $nsg.Id
```

### Create availability set for the virtual machines. 
```azurepowershell-interactive
$set = @{
    Name = 'myAvSet'
    ResourceGroupName = 'myResourceGroup'
    Location = 'eastus'
    Sku = 'Aligned'
    PlatformFaultDomainCount = '2'
    PlatformUpdateDomainCount =  '2'
}
$avs = New-AzAvailabilitySet @set
```
## Create a virtual machine
### Define a credential object
```azurepowershell-interactive
$securePassword = ConvertTo-SecureString ' ' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("azureuser", $securePassword)
```
### Create a virtual machine configuration
```azurepowershell-interactive
$vmConfig = New-AzVMConfig `
  -AvailabilitySetId $avs.Id `
  -VMName "myVM01" `
  -VMSize "Standard_D4s_v3" | `
Set-AzVMOperatingSystem `
  -Linux `
  -ComputerName "myVM01" `
  -Credential $cred `
  -DisablePasswordAuthentication | `
Set-AzVMSourceImage `
  -PublisherName "Canonical" `
  -Offer "UbuntuServer" `
  -Skus "18.04-LTS" `
  -Version "latest" | `
Add-AzVMNetworkInterface `
  -Id $nic.Id
```
### Configure the SSH key
```azurepowershell-interactive
$sshPublicKey = cat ~/.ssh/id_rsa.pub
Add-AzVMSshPublicKey `
  -VM $vmconfig `
  -KeyData $sshPublicKey `
  -Path "/home/azureuser/.ssh/authorized_keys"
```
## Create a Shared Data Disk
```azurepowershell-interactive
$dataDiskConfig = New-AzDiskConfig -Location 'EastUS' -DiskSizeGB 1024 -AccountType Premium_LRS -CreateOption Empty -MaxSharesCount 2
New-AzDisk -ResourceGroupName 'myResourceGroup' -DiskName 'mySharedDisk' -Disk $dataDiskConfig
```



