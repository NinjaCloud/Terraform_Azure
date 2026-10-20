## Understanding Terraform Import

Terraform import brings a resource that was created outside Terraform (for example, in the Azure Portal) under Terraform's management. In Azure, every resource is imported using its **Resource ID**.

#### Prerequisite
Complete LAB 1 (Azure CLI logged in and `ARM_SUBSCRIPTION_ID` exported).

### Task-1: Import existing resources using the `terraform import` command
First, create the resources manually. They stand for the existing infrastructure to be imported.
```
az group create --name import-lab-rg --location eastus
```
```
az network vnet create --name import-lab-vnet --resource-group import-lab-rg --location eastus --address-prefixes 10.10.0.0/16
```
Get the Resource IDs, which are needed for the import.
```
az group show --name import-lab-rg --query id --output tsv
```
```
az network vnet show --name import-lab-vnet --resource-group import-lab-rg --query id --output tsv
```
The IDs look like this:

| Resource | Resource ID format |
| ---------- | -------------------- |
| Resource Group | `/subscriptions/<subscription-id>/resourceGroups/import-lab-rg` |
| Virtual Network | `/subscriptions/<subscription-id>/resourceGroups/import-lab-rg/providers/Microsoft.Network/virtualNetworks/import-lab-vnet` |

```
mkdir import_lab && cd import_lab
```
```
vi import.tf
```
Add the given lines, by pressing "INSERT" and make the configuration match the existing resources.
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

#Execute the below commands
#terraform import azurerm_resource_group.imported <***RESOURCE-GROUP-ID***>
#terraform import azurerm_virtual_network.imported <***VIRTUAL-NETWORK-ID***>

resource "azurerm_resource_group" "imported" {
  name     = "<***NAME OF THE RESOURCE GROUP TO BE IMPORTED***>"
  location = "<***LOCATION OF THE RESOURCE GROUP***>"
}

resource "azurerm_virtual_network" "imported" {
  name                = "<***NAME OF THE VIRTUAL NETWORK***>"
  location            = azurerm_resource_group.imported.location
  resource_group_name = azurerm_resource_group.imported.name
  address_space       = ["<***ADDRESS SPACE OF THE VIRTUAL NETWORK***>"]
  tags = {
    name = "<TAGS if ANY / New Tags can be added as well>"
  }
}
```
For this lab, the values are `import-lab-rg`, `eastus`, `import-lab-vnet` and `10.10.0.0/16`.

Save the file using "ESCAPE + :wq!"
```
terraform init
```
Import the Resource Group first, as the Virtual Network lives inside it.
```
terraform import azurerm_resource_group.imported <***RESOURCE-GROUP-ID***>
```
```
terraform import azurerm_virtual_network.imported <***VIRTUAL-NETWORK-ID***>
```
```
terraform state list
```
```
terraform state show azurerm_virtual_network.imported
```
Run a plan to compare the configuration with the imported resource. The goal is `No changes`. If the plan shows differences, update the configuration (for example, the tags or the address space) until it matches.
```
terraform plan
```

### Task-2: Import a resource using an `import` block (Terraform 1.5 and above)
An `import` block lets you import as part of the normal `plan` and `apply`, and Terraform can even write the resource configuration for you.

Create one more Virtual Network manually.
```
az network vnet create --name import-lab-vnet2 --resource-group import-lab-rg --location eastus --address-prefixes 10.20.0.0/16
```
```
echo $ARM_SUBSCRIPTION_ID
```
```
vi import-block.tf
```
Add the given lines, by pressing "INSERT" and replace `<SUBSCRIPTION-ID>` with the value printed above.
```
import {
  to = azurerm_virtual_network.generated
  id = "/subscriptions/<SUBSCRIPTION-ID>/resourceGroups/import-lab-rg/providers/Microsoft.Network/virtualNetworks/import-lab-vnet2"
}
```
Save the file using "ESCAPE + :wq!"

Ask Terraform to generate the configuration for the resource into a new file.
```
terraform plan -generate-config-out=generated.tf
```
```
cat generated.tf
```
Review the generated file. Remove any attribute that Terraform reports as an error or as a conflict, then apply.
```
terraform apply
```
```
terraform state list
```
`Note:` After the import is done, the `import` block is no longer needed and can be deleted.

### Cleanup
```
terraform destroy
```
`Note:` The generated Virtual Network has no reference to the Resource Group in the configuration, so Terraform may try to delete the Resource Group while the Virtual Network is still being deleted and show an error. If this happens, run `terraform destroy` again.

Finally, confirm in the Portal or with the CLI that the Resource Group is gone, and delete it if it is not.
```
az group exists --name import-lab-rg
```
```
cd ..
```
```
rm -rf import_lab
```
