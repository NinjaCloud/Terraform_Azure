### Launching your first Azure Virtual Machine using Terraform

#### Prerequisite
Complete LAB 1 (Azure CLI logged in and `ARM_SUBSCRIPTION_ID` exported). Create an SSH key pair on the `TerraformServer`. Terraform will place the public key on the new VM. Press `Enter` for all prompts.
```
ssh-keygen -t rsa -b 4096
```
```
ls ~/.ssh
```

#### Create the configuration
```
cd ~
mkdir VM-lab && cd VM-lab/
```
```
vi example.tf
```
Add the given lines, by pressing "INSERT"

##### Note:
1. Replace the `location` with your allocated `Region`.
2. Replace `"Yourname"` in all the resource names with Your Name.
3. If `Standard_B1s` is not available in your region, find a free size with `az vm list-skus --location eastus --size Standard_B --output table` and use it.
```
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "example" {
  name     = "Yourname-RG-1"
  location = "eastus"
}

resource "azurerm_virtual_network" "example" {
  name                = "Yourname-vnet-1"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
}

resource "azurerm_subnet" "example" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "example" {
  name                = "Yourname-pip-1"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_security_group" "example" {
  name                = "Yourname-nsg-1"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name

  security_rule {
    name                       = "Allow-SSH"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_interface" "example" {
  name                = "Yourname-nic-1"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.example.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.example.id
  }
}

resource "azurerm_network_interface_security_group_association" "example" {
  network_interface_id      = azurerm_network_interface.example.id
  network_security_group_id = azurerm_network_security_group.example.id
}

resource "azurerm_linux_virtual_machine" "example" {
  name                  = "Yourname-VM-1"
  resource_group_name   = azurerm_resource_group.example.name
  location              = azurerm_resource_group.example.location
  size                  = "Standard_B1s"
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.example.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  tags = {
    Name = "Yourname-VM-1"
  }
}
```
Save the file using "ESCAPE + :wq!"
```
terraform init
```
```
terraform fmt
```
```
terraform validate
```
```
terraform plan
```
The plan should show `8 to add`.
```
terraform apply
```
```
#List the files
ls 
```
To see what is saved in `terraform.tfstate` use the below command.

```
cat terraform.tfstate
```
Go to the Azure Portal > `Resource groups` > `Yourname-RG-1` and see the resources. You can also check from the CLI:
```
az vm list-ip-addresses --resource-group Yourname-RG-1 --output table
```
Now, Let's Replace the image with `Debian 12` and see what happens.

`Note:` Only the `source_image_reference` block changes. To find other images use `az vm image list --output table`.
```
vi example.tf
```
Replace the `source_image_reference` block, by pressing "INSERT"
```
  source_image_reference {
    publisher = "Debian"
    offer     = "debian-12"
    sku       = "12-gen2"
    version   = "latest"
  }
```

Save the file using "ESCAPE + :wq!"

Now in Terraform Plan Verify the Changes. Look for the `# forces replacement` marker next to the image attributes.
```
terraform plan
```
```
terraform apply
```
Once Applied then go to the portal and see that the Previous VM is being deleted and a new VM is getting created.

Also verify that the OS is `Debian 12`. Connect to the new VM using the public IP from the earlier command.
```
ssh azureuser@<public-ip-address>
```
```
cat /etc/os-release
```
```
exit
```
`Note:` If SSH shows `REMOTE HOST IDENTIFICATION HAS CHANGED`, it is because the VM was recreated. Remove the old key with `ssh-keygen -R <public-ip-address>` and connect again.

Again when you cat the `terraform.tfstate` file it shows the new image details.
```
cat terraform.tfstate
```
Use the `terraform destroy` command for cleaning the infrastructure used in this lab
```
terraform destroy
```
Finally, verify that the resources are deleted in the Portal, or run the below command, which should return `false`.
```
az group exists --name Yourname-RG-1
```
```
cd ~
rm -rf VM-lab
```
