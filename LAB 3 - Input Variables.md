## Input Variables

#### Prerequisite
Complete LAB 1 and LAB 2 (Azure CLI logged in, `ARM_SUBSCRIPTION_ID` exported and the SSH key at `~/.ssh/id_rsa.pub` created).

### Task-1: Create Azure VM using Input variables with default values
```
cd ~
```
```
mkdir variables-lab && cd variables-lab/
```
Now Create Files ie. `provider.tf,` `vars.tf,` `network.tf,` `instance.tf.`
```
vi vars.tf
```
Add the given lines, by pressing "INSERT" also replace your `Region`
```
variable "AZURE_REGION" {
  description = "Azure region for all the resources"
  default     = "eastus"
}

variable "Vm_size" {
  description = "VM size for the Azure Virtual Machine"
  default     = "Standard_B1s"
}

variable "Environment" {
  description = "Environment tag for the Azure Virtual Machine"
  default     = "Training"
}
```
Save the file using "ESCAPE + :wq!"

```
vi provider.tf
```
Add the given lines, by pressing "INSERT" 
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
vi network.tf
```
Add the given lines, by pressing "INSERT" Also replace `YourName`
```
resource "azurerm_resource_group" "terraform_example" {
  name     = "Variables-Lab3-YourName-RG"
  location = var.AZURE_REGION
}

resource "azurerm_virtual_network" "terraform_example" {
  name                = "Variables-Lab3-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.terraform_example.location
  resource_group_name = azurerm_resource_group.terraform_example.name
}

resource "azurerm_subnet" "terraform_example" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.terraform_example.name
  virtual_network_name = azurerm_virtual_network.terraform_example.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_interface" "terraform_example" {
  name                = "Variables-Lab3-nic"
  location            = azurerm_resource_group.terraform_example.location
  resource_group_name = azurerm_resource_group.terraform_example.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.terraform_example.id
    private_ip_address_allocation = "Dynamic"
  }
}
```
Save the file using "ESCAPE + :wq!"
```
vi instance.tf
```
Add the given lines, by pressing "INSERT" Also replace `YourName`
```
resource "azurerm_linux_virtual_machine" "terraform_example" {
  name                  = "Variables-Lab3-YourName"
  resource_group_name   = azurerm_resource_group.terraform_example.name
  location              = azurerm_resource_group.terraform_example.location
  size                  = var.Vm_size
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.terraform_example.id]

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
    Name        = "Variables-Lab3-YourName"
    Environment = var.Environment
  }
}
```
Save the file using "ESCAPE + :wq!"
```
terraform init
```
```
terraform plan
```
```
terraform apply
```
Once applied, go to the Portal and check that the new VM is created using Variables. Check its size and the `Environment` tag.


### Task-2: Create Azure VM using Input variables with -var flag

We will be using same configuration file created above but when applying will pass the variable values with **-var** flag to see the precedence

```
terraform apply -var="Vm_size=Standard_B2s" -var="Environment=Dev"
```
The plan shows the VM size and tag updated in place (`~`). Azure restarts the VM while resizing it.

### Task-3: Create Azure VM using Input variables with Environment Variable
Now we will be using same configuration file created above but when applying will pass the variable values from **ENV variables** to see the precedence

```
export TF_VAR_Vm_size="Standard_B2ms"
```
```
printenv
```
```
terraform apply 
```
The `-var` values from Task-2 are not remembered, so the size comes from the environment variable and the tag goes back to its default value.

`Note:` Terraform loads variable values in this order (a later source overrides an earlier one): default value in the file > `TF_VAR_` environment variables > `terraform.tfvars` > `*.auto.tfvars` > `-var` and `-var-file` flags.


### Task-4: Implementing map variables that dynamically fetch the image based on the Linux distro selected
Remove the environment variable used in Task-3 so that it does not override the file.
```
unset TF_VAR_Vm_size
```
```
vi instance.tf
```
Delete all the existing lines and Add the given code, by pressing "INSERT" and replacing `YourName`

`Note:` To delete all the lines at a time use the commands `Esc+gg+dG` and once cleared then add the new lines.
```
resource "azurerm_linux_virtual_machine" "terraform_example" {
  name                  = "Variables-Lab3-YourName"
  resource_group_name   = azurerm_resource_group.terraform_example.name
  location              = azurerm_resource_group.terraform_example.location
  size                  = var.Vm_size
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.terraform_example.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = var.IMAGES[var.Linux_distro].publisher
    offer     = var.IMAGES[var.Linux_distro].offer
    sku       = var.IMAGES[var.Linux_distro].sku
    version   = "latest"
  }

  tags = {
    Name        = "Variables-Lab3-YourName"
    Environment = var.Environment
  }
}
```
Save the file using "ESCAPE + :wq!"

Now, Also make changes in the `vars.tf` file.
```
vi vars.tf
```
Delete all existing lines and Add the given lines, by pressing "INSERT" Also ensure to replace your `Region` and use a VM size available in that region.

`Note:` To delete all the lines at a time use `Esc+gg+dG`
```
variable "Vm_size" {
  description = "VM size for the Azure Virtual Machine"
  default     = "Standard_B1s"
}

variable "AZURE_REGION" {
  default = "eastus"
}

variable "Environment" {
  default = "Training"
}

variable "Linux_distro" {
  #description = "Please Enter the Linux distro (redhat, ubuntu, debian)"
  default = "debian"
}

variable "IMAGES" {
  type = map(object({
    publisher = string
    offer     = string
    sku       = string
  }))
  default = {
    redhat = {
      publisher = "RedHat"
      offer     = "RHEL"
      sku       = "9-lvm-gen2"
    }
    ubuntu = {
      publisher = "Canonical"
      offer     = "0001-com-ubuntu-server-jammy"
      sku       = "22_04-lts-gen2"
    }
    debian = {
      publisher = "Debian"
      offer     = "debian-12"
      sku       = "12-gen2"
    }
  }
}
```
Save the file using "ESCAPE + :wq!"
```
terraform fmt
```
```
terraform validate
```
```
terraform plan 
```
```
terraform apply 
```
The VM is replaced with a Debian 12 VM, because the image changed. Now remove the default line in the `Linux_distro` variable of the `vars.tf` file along with # ( comment )
```
vi vars.tf
```
The variable should look like this:
```
variable "Linux_distro" {
  description = "Please Enter the Linux distro (redhat, ubuntu, debian)"
}
```

Save the file using "ESCAPE + :wq!"

```
terraform apply
```
Terraform now asks you to enter a value for `var.Linux_distro`. Enter `redhat` and confirm with `yes`. Check in the Portal that the VM now uses the Red Hat image.

Use the `terraform destroy` command to clean the infrastructure used in this lab
```
terraform destroy
```
When prompted for `var.Linux_distro`, enter any of the three values (for example `redhat`).

Once Done remove the `variables-lab` Directory.
```
cd ~
rm -rf variables-lab

```
