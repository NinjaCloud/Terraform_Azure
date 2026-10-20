## Understanding Terraform functions 

#### Prerequisite
Complete LAB 1 and LAB 2 (Azure CLI logged in, `ARM_SUBSCRIPTION_ID` exported and the SSH key at `~/.ssh/id_rsa.pub` created).

### Functions
```
cd ~
```
```
mkdir Lab7 && cd Lab7
```
```
vi functions.tf
```
Add the given lines, by pressing "INSERT" 

`Note:` The `lookup()` function picks a value from a map by key, `element()` picks an item from a list by index, and `count` creates more than one copy of a resource. Make sure each VM size in `vm_size` is available in that region.
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

variable "region" {
  default = "centralindia"
}

variable "names" {
  type    = list(string)
  default = ["vm-1", "vm-2"]
}

variable "region_code" {
  type = map(string)
  default = {
    "eastus"       = "eus"
    "westus"       = "wus"
    "centralindia" = "cin"
  }
}

variable "vm_size" {
  type = map(string)
  default = {
    "eastus"       = "Standard_B1s"
    "westus"       = "Standard_B2s"
    "centralindia" = "Standard_B2s"
  }
}

resource "azurerm_resource_group" "functions" {
  name     = "functions-lab-rg"
  location = var.region
}

resource "azurerm_virtual_network" "functions" {
  name                = "functions-lab-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.functions.location
  resource_group_name = azurerm_resource_group.functions.name
}

resource "azurerm_subnet" "functions" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.functions.name
  virtual_network_name = azurerm_virtual_network.functions.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_interface" "application_servers" {
  count               = 2
  name                = format("%s-nic-%s", lookup(var.region_code, var.region), count.index)
  location            = azurerm_resource_group.functions.location
  resource_group_name = azurerm_resource_group.functions.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.functions.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_linux_virtual_machine" "application_servers" {
  count                 = 2
  name                  = format("%s-%s", lookup(var.region_code, var.region), element(var.names, count.index))
  resource_group_name   = azurerm_resource_group.functions.name
  location              = azurerm_resource_group.functions.location
  size                  = lookup(var.vm_size, var.region)
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.application_servers[count.index].id]

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
    Name = element(var.names, count.index)
  }
}
```

Save the file using "ESCAPE + :wq!"
```
terraform init
```
Before creating anything, try the functions in the Terraform console. Type `exit` to leave the console.
```
terraform console
```
```
lookup(var.region_code, "eastus")
element(var.names, 1)
element(var.names, 2)
length(var.names)
upper("azure")
format("%s-%s", "cin", "vm-1")
```
`Note:` `element(var.names, 2)` returns `vm-1` again, because `element()` wraps around when the index is greater than the size of the list.
```
terraform plan
```
```
terraform apply
```
Go to the Portal > `functions-lab-rg` and check that two VMs were created with the names `cin-vm-1` and `cin-vm-2`, and with the size picked from the `vm_size` map.
```
terraform destroy
```
```
cd ~
rm -rf Lab7
```
