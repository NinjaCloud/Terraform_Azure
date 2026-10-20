## Remote State using Azure Blob Storage

### Task-1: Create a Storage Account and Container for the state file
Terraform stores its state in a blob inside an Azure Storage Account. The storage account must exist before Terraform uses it, so create it manually.

#### Option A: Using the Azure Portal
* Create a new Resource group named `tfstate-rg` in your Allocated Region.
* Create a new Storage account in `tfstate-rg` with the name: `youryametfstate` (3 to 24 characters, lowercase letters and numbers only, and the name must be unique across Azure).
* While creating,
    - **Performance:** `Standard`
    - **Redundancy:** `Locally-redundant storage (LRS)`
    - In the `Advanced` tab, keep `Allow enabling anonymous access on individual containers` **unchecked** (the state file must stay private).
    - In the `Data protection` tab, check `Enable versioning for blobs`
    - Then, click on `Create`
* Once the storage account is ready, open it > `Data storage` > `Containers` > `+ Container` > name it `tfstate` and keep the access level `Private`.

#### Option B: Using the Azure CLI
Replace `<storage-account-name>` and the region with your own values.
```
az group create --name tfstate-rg --location eastus
```
```
az storage account create --name <storage-account-name> --resource-group tfstate-rg --location eastus --sku Standard_LRS --allow-blob-public-access false
```
```
az storage account blob-service-properties update --account-name <storage-account-name> --resource-group tfstate-rg --enable-versioning true
```
```
az storage container create --name tfstate --account-name <storage-account-name> --auth-mode key
```
Cross-check whether the Storage account and the container are created or not, by running the below commands in CLI.
```
az storage account list --resource-group tfstate-rg --output table
```
```
az storage container list --account-name <storage-account-name> --auth-mode key --output table
```

### Task-2: Configure Remote State
```
cd ~
mkdir Blob-Lab && cd Blob-Lab
```
```
mkdir remote-state-lab && cd remote-state-lab
```
```
ls
```
Create a New Configuration File 
```
vi main.tf
```
Add the below given lines, by pressing "INSERT"  

`Note:` This lab creates a Resource Group and a Virtual Network. They are free and quick to create, because the goal here is to see where the state file is stored.
```
resource "azurerm_resource_group" "terraform_remote_state" {
  name     = "remote-state-lab-rg"
  location = var.AZURE_REGION
}

resource "azurerm_virtual_network" "terraform_remote_state" {
  name                = "remote-state-lab-vnet"
  address_space       = var.ADDRESS_SPACE[var.ENVIRONMENT]
  location            = azurerm_resource_group.terraform_remote_state.location
  resource_group_name = azurerm_resource_group.terraform_remote_state.name
}

```
Save the file using "ESCAPE + :wq!"
```
vi vars.tf
```
In `vars.tf` file ensure to replace your `region`.
```
variable "AZURE_REGION" {
  default = "eastus"
}

variable "ENVIRONMENT" {
  default = "dev"
}

variable "ADDRESS_SPACE" {
  type = map(list(string))
  default = {
    dev  = ["10.10.0.0/16"]
    test = ["10.20.0.0/16"]
    prod = ["10.30.0.0/16"]
  }
}
```
Once done, save the file and follow further steps.
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
Now, Create a New Configuration File for storing "`terraform.tfstate`" file in the backend. (ie. `Azure Blob Storage.`)

```
vi backend.tf
```
Add the given lines, by pressing "INSERT" and add your `Resource Group Name` and `Storage Account Name`
```
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "<Replace your storage account name>"
    container_name       = "tfstate"
    key                  = "terraform/remotestate.tfstate"
  }
}
```
`Note:` Backend blocks cannot use variables, so the values must be written directly in the file.

Save the file using "ESCAPE + :wq!"
```
cat backend.tf
```
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
```
terraform apply
```
Check that no state file is created in the local directory, because the state is now stored in Azure.
```
ls
```
* Go to the Azure Portal > your Storage account > `Data storage` > `Containers` > `tfstate`. You will see the blob `terraform/remotestate.tfstate`.
* Click on the blob > `Edit` tab to view the content of the state file. (The container is private, so it cannot be opened from a public URL. This is the safer approach, as a state file may contain secrets.)
* It shows the attributes of the Resource Group and Virtual Network in the Terraform state (`azurerm_resource_group.terraform_remote_state` and `azurerm_virtual_network.terraform_remote_state`).
* You can also list the blob from the CLI:
```
az storage blob list --account-name <storage-account-name> --container-name tfstate --auth-mode key --output table
```
You can also read the remote state from Terraform:
```
terraform state list
```
```
terraform state pull
```
`Note:` Azure Blob Storage supports state locking automatically using a blob lease. While `terraform apply` is running, the blob shows `Lease status: Locked` in the portal, so two people cannot change the state at the same time.

To see versioning working, go to the blob > `Versions` tab. Each `terraform apply` that changes the state creates a new version.

Use the `terraform destroy` command to clean the infrastructure used in this lab, 
```
terraform destroy
```
Once done, Remove the directory using "`rm -rf`"
```
cd ~
rm -rf Blob-Lab
```
**Note:** Also Ensure to delete the `tfstate-rg` Resource Group, which removes the Storage account and the state file with it.
```
az group delete --name tfstate-rg --yes --no-wait
```
