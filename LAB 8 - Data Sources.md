## Data sources

A data source reads information that already exists (outside this configuration or in Azure itself) so that the configuration can use it, without creating or changing it.

#### Prerequisite
Complete LAB 1 and LAB 2 (Azure CLI logged in, `ARM_SUBSCRIPTION_ID` exported and the SSH key at `~/.ssh/id_rsa.pub` created).

First, create a Resource Group manually, which Terraform will read as an existing resource.
```
az group create --name data-lab-rg --location eastus
```
```
cd ~
```
```
mkdir lab8 && cd lab8
```
```
vi data.tf
```

Add the given lines, by pressing "INSERT" 

* `azurerm_client_config` reads the identity (subscription, tenant) you are logged in with.
* `azurerm_resource_group` reads the existing Resource Group.
* `azurerm_platform_image` finds the latest version of an Azure Marketplace image (like the AMI lookup in AWS).
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

data "azurerm_client_config" "current" {}

data "azurerm_resource_group" "existing" {
  name = "data-lab-rg"
}

data "azurerm_platform_image" "ubuntu" {
  location  = data.azurerm_resource_group.existing.location
  publisher = "Canonical"
  offer     = "0001-com-ubuntu-server-jammy"
  sku       = "22_04-lts-gen2"
}

resource "azurerm_virtual_network" "data_lab" {
  name                = "data-lab-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = data.azurerm_resource_group.existing.location
  resource_group_name = data.azurerm_resource_group.existing.name
}

resource "azurerm_subnet" "data_lab" {
  name                 = "internal"
  resource_group_name  = data.azurerm_resource_group.existing.name
  virtual_network_name = azurerm_virtual_network.data_lab.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_interface" "data_lab" {
  name                = "data-lab-nic"
  location            = data.azurerm_resource_group.existing.location
  resource_group_name = data.azurerm_resource_group.existing.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.data_lab.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_linux_virtual_machine" "ubuntu_server" {
  name                  = "Ubuntu-B1s"
  resource_group_name   = data.azurerm_resource_group.existing.name
  location              = data.azurerm_resource_group.existing.location
  size                  = "Standard_B1s"
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.data_lab.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = data.azurerm_platform_image.ubuntu.publisher
    offer     = data.azurerm_platform_image.ubuntu.offer
    sku       = data.azurerm_platform_image.ubuntu.sku
    version   = data.azurerm_platform_image.ubuntu.version
  }

  tags = {
    Name = "Ubuntu-B1s"
  }
}

output "subscription_id" {
  value = data.azurerm_client_config.current.subscription_id
}

output "ubuntu_image_version" {
  value = data.azurerm_platform_image.ubuntu.version
}
```

Save the file using "ESCAPE + :wq!"

```
terraform init
```
```
terraform plan
```
Notice in the plan that the Resource Group is **not** listed for creation, because it is only read, and that the image version is already filled in.
```
terraform apply
```
Check the outputs, and that the VM runs the exact image version that the data source found.
```
terraform output
```
```
terraform destroy
```
The Resource Group `data-lab-rg` was not created by Terraform, so it is not deleted by `terraform destroy`. Delete it manually.
```
az group delete --name data-lab-rg --yes --no-wait
```
```
cd ~
rm -rf lab8
```
