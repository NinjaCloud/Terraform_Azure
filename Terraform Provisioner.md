## Software Provisioning

Provisioners run scripts or copy files after a resource is created. In this lab, Terraform creates an Azure VM, installs Nginx on it with `remote-exec`, copies a web page with `file`, and records the public IP on the local machine with `local-exec`.

`Note:` HashiCorp recommends provisioners only as a last resort. Prefer cloud-init (see `custom_data` in LAB 10) when it is possible.

#### Prerequisite
Complete LAB 1 (Azure CLI logged in and `ARM_SUBSCRIPTION_ID` exported).

Create the terraform directory and set up the config files in it
```
cd ~ && mkdir provisioning-lab && cd provisioning-lab
```
As a first step, create a SSH key pair using `ssh-keygen` command. If the key pair from LAB 2 already exists in `~/.ssh`, you can skip this step.
```
ssh-keygen -t rsa -b 4096
```
Create the web page that will be copied to the VM.
```
vi index.html
```
```
<h1>Welcome to Terraform Provisioning on Azure</h1>
```
```
vi main.tf
```
Copy and paste the below code into `main.tf`
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

resource "azurerm_resource_group" "provisioning" {
  name     = var.resource_group_name
  location = var.region
}

resource "azurerm_virtual_network" "provisioning" {
  name                = "provisioning-vnet"
  address_space       = ["10.30.0.0/16"]
  location            = azurerm_resource_group.provisioning.location
  resource_group_name = azurerm_resource_group.provisioning.name
}

resource "azurerm_subnet" "provisioning" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.provisioning.name
  virtual_network_name = azurerm_virtual_network.provisioning.name
  address_prefixes     = ["10.30.1.0/24"]
}

resource "azurerm_public_ip" "provisioning" {
  name                = "provisioning-pip"
  location            = azurerm_resource_group.provisioning.location
  resource_group_name = azurerm_resource_group.provisioning.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

# Open the ports 22 (SSH) and 80 (HTTP)
resource "azurerm_network_security_group" "provisioning" {
  name                = "provisioning-nsg"
  location            = azurerm_resource_group.provisioning.location
  resource_group_name = azurerm_resource_group.provisioning.name

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

resource "azurerm_network_interface" "provisioning" {
  name                = "provisioning-nic"
  location            = azurerm_resource_group.provisioning.location
  resource_group_name = azurerm_resource_group.provisioning.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.provisioning.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.provisioning.id
  }
}

resource "azurerm_network_interface_security_group_association" "provisioning" {
  network_interface_id      = azurerm_network_interface.provisioning.id
  network_security_group_id = azurerm_network_security_group.provisioning.id
}

# to create the Azure VM
resource "azurerm_linux_virtual_machine" "my_machine" {
  name                  = var.vm_name
  resource_group_name   = azurerm_resource_group.provisioning.name
  location              = azurerm_resource_group.provisioning.location
  size                  = var.vm_size
  admin_username        = var.admin_username
  network_interface_ids = [azurerm_network_interface.provisioning.id]
  depends_on            = [azurerm_network_interface_security_group_association.provisioning]

  admin_ssh_key {
    username   = var.admin_username
    public_key = file(var.public_key)
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
    Name = "Provisioning-VM"
  }

  # Wait for the first boot to finish, then install Nginx
  provisioner "remote-exec" {
    connection {
      type        = "ssh"
      user        = var.admin_username
      private_key = file(var.private_key) # Replace with the path to your private key
      host        = azurerm_public_ip.provisioning.ip_address
    }

    inline = [
      "cloud-init status --wait",
      "sudo apt-get update -y",
      "sudo apt-get install -y nginx"
    ]
  }

  # Copy the web page from the host machine to the VM
  provisioner "file" {
    source      = "${path.module}/index.html" # Source path on Host machine
    destination = "/tmp/index.html"           # Destination path on the VM
    connection {
      type        = "ssh"
      user        = var.admin_username
      private_key = file(var.private_key) # Replace with the path to your private key
      host        = azurerm_public_ip.provisioning.ip_address
    }
  }

  # After the file is copied, move it to the web root
  provisioner "remote-exec" {
    connection {
      type        = "ssh"
      user        = var.admin_username
      private_key = file(var.private_key) # Replace with the path to your private key
      host        = azurerm_public_ip.provisioning.ip_address
    }

    inline = [
      "sudo mv /tmp/index.html /var/www/html/index.html",
      "sudo chmod 644 /var/www/html/index.html"
    ]
  }

  # Runs on the machine where Terraform is executed
  provisioner "local-exec" {
    command = <<-EOT
      echo ${azurerm_public_ip.provisioning.ip_address} >> /home/azureuser/provisioning-lab/public-ip
    EOT
  }
}
```
Now, create the variables file with all variables to be used in the `main.tf` config file.
```
vi variables.tf
```
```
variable "region" {
  default = "eastus"
}

variable "resource_group_name" {
  default = "provisioning-lab-rg"
}

# Replace 'yourname' with your first name
variable "vm_name" {
  default = "yourname-provisioning-vm"
}

# We are only using a small size for this lab
variable "vm_size" {
  default = "Standard_B1s"
}

variable "admin_username" {
  default = "azureuser"
}

variable "public_key" {
  default = "/home/azureuser/.ssh/id_rsa.pub" #Ubuntu OS
}

variable "private_key" {
  default = "/home/azureuser/.ssh/id_rsa" #Ubuntu OS
}
```
Now, execute the terraform commands to launch the new server
```
terraform init
```
```
terraform plan
```
```
terraform apply -auto-approve
```
Once the Changes are applied, Go to the `Virtual machines` page in the Azure Portal and check that the `New VM` is launched. Also check the file `public-ip` to ensure the output.
```
cat /home/azureuser/provisioning-lab/public-ip
```
Check that Nginx is serving the copied page.
```
curl http://$(cat /home/azureuser/provisioning-lab/public-ip)
```
From the `TerraformServer` SSH into the new VM, check it is accessible.

```
ssh azureuser@<vm public ip address>
```
```
systemctl status nginx
```
```
exit
````
Now destroy the infra
```
terraform destroy -auto-approve
```
```
cd ~ && rm -rf provisioning-lab
```
