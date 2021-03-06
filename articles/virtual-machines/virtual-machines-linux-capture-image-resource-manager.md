<properties
	pageTitle="Capture a Linux VM to use as a template | Microsoft Azure"
	description="Learn how to capture an image of a Linux-based Azure virtual machine (VM) created with the Azure Resource Manager deployment model."
	services="virtual-machines"
	documentationCenter=""
	authors="dlepow"
	manager="timlt"
	editor=""
	tags="azure-resource-manager"/>

<tags
	ms.service="virtual-machines"
	ms.workload="infrastructure-services"
	ms.tgt_pltfrm="vm-linux"
	ms.devlang="na"
	ms.topic="article"
	ms.date="11/05/2015"
	ms.author="danlep"/>


# How to capture a Linux virtual machine to use as a Resource Manager template

[AZURE.INCLUDE [learn-about-deployment-models](../../includes/learn-about-deployment-models-rm-include.md)] [classic deployment model](virtual-machines-linux-capture-image.md).


This article shows you how to use the Azure Command-Line Interface (CLI) to capture an Azure virtual machine running Linux so you can use it as an Azure Resource Manager template to create other virtual machines. This template specifies the OS disk and data disks attached to the virtual machine. It doesn't include the virtual network resources you'll need to create an Azure Resource Manager VM, so in most cases you'll need to set those up separately before you create another virtual machine that uses the template.

## Before you begin

These steps assume that you've already created an Azure virtual machine in the Azure Resource Manager deployment model and configured the operating system, including attaching any data disks and making other customizations like installing applications. If you haven't done this yet, see these instructions for using the Azure CLI in Azure Resource Manager mode:

- [Deploy and manage virtual machines by using Azure Resource Manager templates and the Azure CLI](virtual-machines-deploy-rmtemplates-azure-cli.md)

For example, you might create a resource group named *MyResourceGroup* in the Central US region. Then use an **azure vm quick-create** command similar to the following to deploy an Ubuntu 14.04 LTS VM in the resource group.

 	azure vm quick-create -g MyResourceGroup -n <your-virtual-machine-name> "centralus" -y Linux -Q canonical:ubuntuserver:14.04.2-LTS:14.04.201507060 -u <your-user-name> -p <your-password>

After the VM is provisioned and running, you might want to attach and mount a data disk. See instructions [here](virtual-machines-linux-tutorial.md#attach-and-mount-a-disk).

To perform other customizations, you'll need to connect to the VM using an SSH client of your choice. For details, see [Connect to your Azure Linux VM using ssh](virtual-machines-linux-tutorial-portal-rm.md#connect-to-your-azure-linux-vm-using-strongsshstrong).


## Capture the VM

1. When you are ready to capture the VM, connect to it using your SSH client.

2. In the SSH window, type the following command. Note that the output from **waagent** may vary slightly depending on the version of this utility:

	`sudo waagent -deprovision+user`

	This command will attempt to clean the system and make it suitable for re-provisioning. This operation performs the following tasks:

	- Removes SSH host keys (if Provisioning.RegenerateSshHostKeyPair is 'y' in the configuration file)
	- Clears nameserver configuration in /etc/resolv.conf
	- Removes the `root` user's password from /etc/shadow (if Provisioning.DeleteRootPassword is 'y' in the configuration file)
	- Removes cached DHCP client leases
	- Resets host name to localhost.localdomain
	- Deletes the last provisioned user account (obtained from /var/lib/waagent) and associated data.

	>[AZURE.NOTE] Deprovisioning deletes files and data in an effort to "generalize" the image. Only run this command on a VM that you intend to capture as an image. It does not guarantee that the image is cleared of all sensitive information or is suitable for redistribution to third parties.

3. Type **y** to continue. You can add the **-force** parameter to avoid this confirmation step.

4. Type **exit** to close the SSH client.

	>[AZURE.NOTE] The next steps assume you have already [installed the Azure CLI](../xplat-cli-install.md) on your client computer.

5. From your client computer, open the Azure CLI and login to your Azure subscription. For details, read [Connect to an Azure subscription from the Azure CLI](../xplat-cli-connect.md).

6. Make sure you are in Resource Manager mode:

	`azure config mode arm`

7. Stop the VM which you already deprovisioned by using the following command:

	`azure vm stop –g <your-resource-group-name> -n <your-virtual-machine-name>`

8. Generalize the VM with the following command:

	`azure vm generalize –g <your-resource-group-name> -n <your-virtual-machine-name>`

9. Now capture the image and a local file template with the following command:

	`azure vm capture –g <your-resource-group-name> -n <your-virtual-machine-name> -p <your-vhd-name-prefix> -t <your-template-file-name.json>`

	This command creates a generalized OS image, using the VHD name prefix you specify for the VM disks. The image VHD files get created by default in the same storage account that the original VM used. The **-t** option creates a local JSON file template you can use to create a new VM from the image.

>[AZURE.TIP] To find the location of an image, open the JSON file template. In the **storageProfile**, find the **uri** of the **image** located in the **system** container. For example, the uri of the OS disk image is similar to `https://clixxxxxxxxxxxxxxxxxxxx.blob.core.windows.net/system/Microsoft.Compute/Images/vhds/your-prefix-osDisk.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.vhd`.

## Deploy a new VM from the captured image
Now use the image with a template to create a new Linux VM. These steps show you how to use the Azure CLI and the JSON file template you created with the `azure vm capture` command to create the VM in a new virtual network.

### Create network resources

To use the template, you first need to set up a virtual network and NIC for your new VM. We recommend you create a new resource group for these resources. Run commands similar to the following, substituting names for your resources and an appropriate Azure location ("centralus" in these commands):

	azure group create <your-new-resource-group-name> -l "centralus"

	azure network vnet create <your-new-resource-group-name> <your-vnet-name> -l "centralus"

	azure network vnet subnet create <your-new-resource-group-name> <your-vnet-name> <your-subnet-name>

	azure network public-ip create <your-new-resource-group-name> <your-ip-name> -l "centralus"

	azure network nic create <your-new-resource-group-name> <your-nic-name> -k <your-subnetname> -m <your-vnet-name> -p <your-ip-name> -l "centralus"

To deploy a VM from the image by using the JSON you saved during capture, you'll need the Id of the NIC. Obtain it by running the following command.

	azure network nic show <your-new-resource-group-name> <your-nic-name>

The **Id** in the output is a string similar to this.

	/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/<your-new-resource-group-name>/providers/Microsoft.Network/networkInterfaces/<your-nic-name>



### Create a new deployment
Now run the following command to create your VM from the captured VM image and the template JSON file you saved.

	azure group deployment create –g <your-new-resource-group-name> -n <your-new-deployment-name> -f <your-template-file-name.json>

You are prompted to supply a new VM name, the admin user name and password, and the Id of the NIC you created previously.

	info:    Executing command group deployment create
	info:    Supply values for the following parameters
	vmName: mynewvm
	adminUserName: myadminuser
	adminPassword: ********
	networkInterfaceId: /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resource Groups/mynewrg/providers/Microsoft.Network/networkInterfaces/mynewnic

You will see output similar to the following for a successful deployment.

	+ Initializing template configurations and parameters
	+ Creating a deployment
	info:    Created template deployment "dlnewdeploy"
	+ Waiting for deployment to complete
	data:    DeploymentName     : mynewdeploy
	data:    ResourceGroupName  : mynewrg
	data:    ProvisioningState  : Succeeded
	data:    Timestamp          : 2015-10-29T16:35:47.3419991Z
	data:    Mode               : Incremental
	data:    Name                Type          Value


	data:    ------------------  ------------  -------------------------------------

	data:    vmName              String        mynewvm


	data:    vmSize              String        Standard_D1


	data:    adminUserName       String        myadminuser


	data:    adminPassword       SecureString  undefined


	data:    networkInterfaceId  String        /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/mynewrg/providers/Microsoft.Network/networkInterfaces/mynewnic
	info:    group deployment create command OK

### Verify the deployment

Now SSH to the virtual machine you created to verify the deployment and start using the new VM. To connect via SSH, find the IP address of the VM you created by running the following command:

	azure network public-ip show <your-new-resource-group-name> <your-ip-name>

The public IP address is listed in the command output. By default you connect to the Linux VM by SSH on port 22.

## Create additional VMs with the template

Use the captured image and template to deploy additional VMs using the steps outlined in the preceding section.

* Ensure that your VM image is in the same storage account that will host your VM's VHD
* Copy the template JSON file and enter a unique value for the **uri** of each VM's VHD
* Create a new NIC in either the same or a different virtual network
* Create a deployment in the resource group in which you set up the virtual network, using the modified template JSON file

If you want the network set up automatically when you create a VM from the image, use the [101-vm-from-user-image template](https://github.com/Azure/azure-quickstart-templates/tree/master/101-vm-from-user-image) from GitHub. This template creates a VM from your custom image and the necessary virtual network, public IP address, and NIC resources. For a walkthrough of using the template in the Azure preview portal, see [How to create a virtual machine from a custom image using an ARM template](http://codeisahighway.com/how-to-create-a-virtual-machine-from-a-custom-image-using-an-arm-template/).

## Use the azure vm create command

You'll generally want to use a Resource Manager template to create a VM from the image. However, you can create the VM _imperatively_ by using the **azure vm create** command with the **--os-disk-vhd** (**-d**) parameter.  

Do the following before running **azure vm create** with the image:

1.	Create a new resource group, or identify an existing resource group for the deployment.

2.	Create a public IP address resource and a NIC resource for the new VM. For steps to create a virtual network, public IP address, and NIC by using the CLI, see earlier in this article. (**azure vm create** can also create a new NIC but you will need to pass additional parameters for a virtual network and subnet.)

3.	Ensure that you copy the image VHD to a blob container location that doesn't have folders (virtual directories). By default the image you captured is stored in nested folders in a storage blob container (URI similar to `https://clixxxxxxxxxxxxxxxxxxxx.blob.core.windows.net/system/Microsoft.Compute/Images/vhds/your-prefix-osDisk.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.vhd`. The **azure vm create** command currently can create a VM only from an OS disk VHD stored at the top level of a blob container. For example, you might copy the image VHD to `https://yourstorage.blob.core.windows.net/vhds/your-prefix-OsDisk.vhd`.  

Then run a command similar to the following.

	azure vm create -g <your-resource-group-name> -n <your-new-vm-name> -l eastus -y Linux -o <your-storage-account-name> -d "https://yourstorage.blob.core.windows.net/vhds/your-prefix-OsDisk.vhd" -z Standard_A1 -u <your-admin-name> -p <your-admin-password> -f <your-nic-name>
	
For additional command options, run `azure help vm create`.

## Next steps

To manage your VMs with the CLI, see the tasks in [Deploy and manage virtual machines by using Azure Resource Manager templates and the Azure CLI](virtual-machines-deploy-rmtemplates-azure-cli.md).