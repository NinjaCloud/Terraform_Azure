## Terraform Modules

A module is a folder of Terraform files that can be reused. In this lab, the root configuration calls two child modules: `network` (Virtual Network, Subnet and Network Security Group) and `compute` (Public IP, Network Interface and Linux VM running Nginx).

#### Prerequisite
Complete LAB 1 (Azure CLI logged in and `ARM_SUBSCRIPTION_ID` exported).

### Task-1: Create the module folders
```
cd /home/azureuser/
```
```
sudo apt install tree -y
```
```
mkdir -p terraform-modules/modules/network terraform-modules/modules/compute
```
```
cd terraform-modules
```
Create a key pair. The same public key will be used in the new Azure VM.
```
ssh-keygen -t rsa -b 4096 -f mykey -N ""
```
By the end of the lab the folder will look like this:
```
terraform-modules
├── main.tf
├── outputs.tf
├── provider.tf
├── variables.tf
├── mykey
├── mykey.pub
└── modules
    ├── compute
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    └── network
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
```

### Task-2: Create the `network` module
```
vi modules/network/variables.tf
```
Add the given lines, by pressing "INSERT"
```
variable "location" {
  type = string
}

variable "resource_group_name" {
  type = string
}

variable "vnet_cidr" {
  type = string
}

variable "subnet_cidr" {
  type = string
}

variable "nsg_name" {
  type = string
}
```
Save the file using "ESCAPE + :wq!"
```
vi modules/network/main.tf
```
```
resource "azurerm_virtual_network" "this" {
  name                = "modules-lab-vnet"
  address_space       = [var.vnet_cidr]
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_subnet" "this" {
  name                 = "modules-lab-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.this.name
  address_prefixes     = [var.subnet_cidr]
}

resource "azurerm_network_security_group" "this" {
  name                = var.nsg_name
  location            = var.location
  resource_group_name = var.resource_group_name

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

  security_rule {
    name                       = "Allow-HTTP"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_subnet_network_security_group_association" "this" {
  subnet_id                 = azurerm_subnet.this.id
  network_security_group_id = azurerm_network_security_group.this.id
}
```
Save the file using "ESCAPE + :wq!"
```
vi modules/network/outputs.tf
```
```
output "subnet_id" {
  description = "ID of the subnet created by this module"
  value       = azurerm_subnet.this.id
}
```
Save the file using "ESCAPE + :wq!"

### Task-3: Create the `compute` module
```
vi modules/compute/variables.tf
```
```
variable "location" {
  type = string
}

variable "resource_group_name" {
  type = string
}

variable "subnet_id" {
  type = string
}

variable "vm_size" {
  type = string
}

variable "admin_username" {
  type = string
}

variable "public_key" {
  type = string
}
```
Save the file using "ESCAPE + :wq!"
```
vi modules/compute/main.tf
```
```
resource "azurerm_public_ip" "this" {
  name                = "modules-lab-pip"
  location            = var.location
  resource_group_name = var.resource_group_name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_interface" "this" {
  name                = "modules-lab-nic"
  location            = var.location
  resource_group_name = var.resource_group_name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = var.subnet_id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.this.id
  }
}

resource "azurerm_linux_virtual_machine" "this" {
  name                  = "modules-lab-vm"
  resource_group_name   = var.resource_group_name
  location              = var.location
  size                  = var.vm_size
  admin_username        = var.admin_username
  network_interface_ids = [azurerm_network_interface.this.id]

  admin_ssh_key {
    username   = var.admin_username
    public_key = var.public_key
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

  # Install Nginx at first boot using cloud-init
  custom_data = base64encode(<<-EOT
    #cloud-config
    package_update: true
    packages:
      - nginx
  EOT
  )
}
```
Save the file using "ESCAPE + :wq!"
```
vi modules/compute/outputs.tf
```
```
output "public_ip" {
  description = "Public IP address of the VM"
  value       = azurerm_public_ip.this.ip_address
}
```
Save the file using "ESCAPE + :wq!"

### Task-4: Create the root configuration
```
vi provider.tf
```
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
```
Save the file using "ESCAPE + :wq!"
```
vi variables.tf
```
**Note:** Replace the `location` default with your allocated `Region`.
* If you get an `Error` for the Network Security Group or the Resource Group (it already exists), change the `nsg_name` or `resource_group_name` here.
* If `Standard_B1s` is not available in your region, use another size (for example `Standard_B2s`).
```
variable "location" {
  default = "eastus"
}

variable "resource_group_name" {
  default = "modules-lab-rg"
}

variable "vnet_cidr" {
  default = "10.20.0.0/16"
}

variable "subnet_cidr" {
  default = "10.20.1.0/24"
}

variable "nsg_name" {
  default = "modules-lab-nsg"
}

variable "vm_size" {
  default = "Standard_B1s"
}

variable "admin_username" {
  default = "azureuser"
}

variable "public_key_path" {
  default = "./mykey.pub"
}
```
Save the file using "ESCAPE + :wq!"
```
vi main.tf
```
Notice how the output of the `network` module (`subnet_id`) is passed as an input to the `compute` module.
```
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

module "network" {
  source = "./modules/network"

  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  vnet_cidr           = var.vnet_cidr
  subnet_cidr         = var.subnet_cidr
  nsg_name            = var.nsg_name
}

module "compute" {
  source = "./modules/compute"

  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = module.network.subnet_id
  vm_size             = var.vm_size
  admin_username      = var.admin_username
  public_key          = file(var.public_key_path)
}
```
Save the file using "ESCAPE + :wq!"
```
vi outputs.tf
```
```
output "public_ip" {
  value = module.compute.public_ip
}

output "web_url" {
  value = "http://${module.compute.public_ip}"
}
```
Save the file using "ESCAPE + :wq!"
```
tree
```

### Task-5: Run the configuration
Notice that `terraform init` now also prints `Initializing modules...`
```
terraform init
```
```
terraform fmt -recursive
```
```
terraform validate
```
```
terraform plan
```
```
terraform apply -auto-approve
```
**Note:**
* If it is showing any `Error` for the `Network Security Group / Resource Group`, It means that they already exist. Rename them in the `variables.tf` file or delete them in the Portal.

Once the resources are created. Then, verify all the resources in the Portal under the Resource Group `modules-lab-rg`. Wait 2 to 3 minutes for Nginx to install, and open the URL from the output in a browser (or use `curl`).
```
terraform output
```
```
curl $(terraform output -raw public_ip)
```
```
ssh -i mykey azureuser@$(terraform output -raw public_ip)
```
```
exit
```
Then destroy them.
```
terraform destroy
```
Once Destroyed, remove the Directory.
```
cd ~
rm -rf terraform-modules
```
