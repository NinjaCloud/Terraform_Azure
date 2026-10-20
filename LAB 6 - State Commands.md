## State Commands

#### Prerequisite
Complete LAB 1 (Azure CLI logged in and `ARM_SUBSCRIPTION_ID` exported).

### Setup: Create some resources to practice on
```
cd ~
mkdir Lab6 && cd Lab6
```
```
vi main.tf
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
  name     = "state-lab-rg"
  location = "eastus"
}

resource "azurerm_virtual_network" "example" {
  name                = "state-lab-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
}
```
Save the file using "ESCAPE + :wq!"
```
terraform init
```
```
terraform apply -auto-approve
```

### terraform state list
This command lists all resources that are being tracked in the Terraform state.
```
terraform state list
```

### terraform state show
Displays detailed information about a specific resource in the Terraform state, including its attributes and metadata.
```
terraform state show azurerm_virtual_network.example
```

### terraform state pull
Manually pulls the state and outputs it as JSON. This is useful for debugging or inspecting the current state of your infrastructure.
```
terraform state pull > terraform_state.json
```
```
cat terraform_state.json
```

### Refreshing the state (terraform apply -refresh-only)
Updates the state file with the real-world infrastructure. This is useful when Terraform's state file becomes out of sync with the actual infrastructure, typically due to manual changes made outside of Terraform.

`Note:` The old `terraform refresh` command is deprecated. Use the `-refresh-only` mode instead, which shows the drift first and asks for your approval before the state is updated.

Make a manual change outside Terraform by adding a tag to the Virtual Network:
```
az network vnet update --name state-lab-vnet --resource-group state-lab-rg --set tags.env=manual
```
See the difference between the real infrastructure and the state:
```
terraform plan -refresh-only
```
Update the state file with the real-world values:
```
terraform apply -refresh-only
```
```
terraform state show azurerm_virtual_network.example
```
The `env = "manual"` tag is now stored in the state. Because the configuration does not have this tag, a normal `terraform plan` will now show that Terraform wants to remove it.

### terraform state mv
Moves an item in the Terraform state from one address to another. This is useful when you want to rename a resource or change its location in the state file.
```
terraform state mv azurerm_virtual_network.example azurerm_virtual_network.new_example
```
After moving the item in the state, rename it in the configuration also, otherwise Terraform will plan to destroy the old one and create a new one.
```
vi main.tf
```
Change `resource "azurerm_virtual_network" "example"` to `resource "azurerm_virtual_network" "new_example"`, then save the file.
```
terraform plan
```
The plan should show that only the tag will be removed, and nothing will be destroyed or created.

### terraform state rm
Removes one or more items from the Terraform state. This is typically used when you want to stop managing a resource with Terraform. **The real resource in Azure is not deleted.**
```
terraform state rm azurerm_virtual_network.new_example
```
```
terraform state list
```
The Virtual Network is still present in Azure, but Terraform no longer tracks it.
```
az network vnet list --resource-group state-lab-rg --output table
```

### terraform state push
Manually pushes the local state to the configured backend. This command is typically used in scenarios where you want to sync a local state file with the remote state backend (for example, Azure Blob Storage from LAB 5).

`Caution:` It overwrites the state in the backend, so use it only to recover a state file and never as a routine step. You do not need to run it in this lab.
```
terraform state push terraform_state.json
```

### Cleanup
Because the Virtual Network is no longer tracked, `terraform destroy` would try to delete only the Resource Group, and Azure blocks deleting a Resource Group that still holds resources. Delete the whole Resource Group with the Azure CLI instead.
```
az group delete --name state-lab-rg --yes --no-wait
```
```
cd ~
rm -rf Lab6
```
