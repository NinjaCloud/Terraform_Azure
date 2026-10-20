## Using Output Feature 

#### Prerequisite
Complete LAB 1 and LAB 2 (Azure CLI logged in, `ARM_SUBSCRIPTION_ID` exported and the SSH key at `~/.ssh/id_rsa.pub` created).

### Task-1: Using output feature of Terraform to get the IP Address of Azure VM
```
mkdir output-variable-lab && cd output-variable-lab
```
```
ls
```
Create the instance.tf file
```
vi instance.tf
```
Add the given lines, by pressing "INSERT" and replace the `location` with your allocated `Region`.
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
  name     = "output-lab-rg"
  location = "eastus"
}

resource "azurerm_virtual_network" "example" {
  name                = "output-lab-vnet"
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
  name                = "output-lab-pip"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_interface" "example" {
  name                = "output-lab-nic"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.example.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.example.id
  }
}

resource "azurerm_linux_virtual_machine" "example" {
  name                  = "output-lab-vm"
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
}
```
Save the changes
```
vi output.tf
```
```
output "Public_ip" {
  description = "Public IP of the virtual machine"
  value       = azurerm_public_ip.example.ip_address
}

output "Private_ip" {
  sensitive   = true
  description = "Private IP of the virtual machine"
  value       = azurerm_network_interface.example.private_ip_address

}
```
Save the file

Once replaced, save the changes.
```
terraform init
```
```
terraform plan
```
```
terraform apply
```
Notice that `Private_ip` is shown as `<sensitive>` in the apply output, while `Public_ip` is printed.
```
terraform output Public_ip
```
```
terraform output Private_ip
```
Use the `terraform destroy` command to clean the infrastructure used in this lab.
```
terraform destroy
```
Once done, remove the directory using the `rm -rf` command below.
```
cd ~
rm -rf output-variable-lab
```
