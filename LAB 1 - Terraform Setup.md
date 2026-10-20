## Creating an Azure Virtual Machine and Installing Terraform

To begin, log in to the [Azure Portal](https://portal.azure.com).

### Task-1: Installing Terraform on Ubuntu operating system

* Manually create a Virtual Machine (**Virtual machines** > **Create** > **Azure virtual machine**) with the following settings:
    - **Resource group:** Create new > `Terraform-RG`
    - **Virtual machine name:** `TerraformServer`
    - **Region:** region of your choice
    - **Image:** `Ubuntu Server 24.04 LTS - x64 Gen2` (latest LTS version)
    - **Size:** `Standard_B1s` (if it is not available in your region, pick `Standard_B2s`)
    - **Authentication type:** `SSH public key`
    - **Username:** `azureuser`
    - **SSH public key source:** `Generate new key pair`
    - **Key pair name:** `Terraform-Keypair-YourName`
    - **Public inbound ports:** `Allow selected ports` and select `SSH (22)`, `HTTP (80)` and `HTTPS (443)`
    - **Disks:** OS disk type `Standard SSD`. Keep the default size (`30 GiB`), because Ubuntu images on Azure cannot use a smaller disk.
    - **Tags:** `Name : TerraformServer`
* Click on `Review + create` > `Create`.
* In the pop-up, choose `Download private key and create resource`. Save the `.pem` file safely.
* Once the deployment is complete, open the VM and copy its `Public IP address`.
* Connect to the VM using `MobaXterm`, `Putty` or a terminal with username "`azureuser`".

For a terminal, restrict the key permissions and connect:
```
chmod 400 Terraform-Keypair-YourName.pem
```
```
ssh -i Terraform-Keypair-YourName.pem azureuser@<public-ip-address>
```

Once the VM is ready, follow the below commands to perform the lab:
```
sudo hostnamectl set-hostname TerraformServer
bash
```
```
sudo apt update
```
```
sudo apt install wget unzip -y
```
```
wget https://releases.hashicorp.com/terraform/1.10.3/terraform_1.10.3_linux_amd64.zip

```
To know the latest Terraform version - [Install Terraform](https://developer.hashicorp.com/terraform/downloads)
```
unzip terraform_1.10.3_linux_amd64.zip
```
```
ls
sudo mv terraform /usr/local/bin
```
```
rm terraform_1.10.3_linux_amd64.zip
```
```
ls
terraform
```
```
terraform -v
```

### Task-2: Install Azure CLI and login to Azure using Credentials.
Install the Azure CLI:
```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```
```
az version
```
Login to Azure using the device code flow:
```
az login --use-device-code
```
* The command prints a code. Open `https://microsoft.com/devicelogin` in your browser, enter the code and sign in with your Azure account.

Once logged in, check the account access
```
az account show
```
`Or` Use below command to list all subscriptions available to your account.
```
az account list --output table
```
If you have more than one subscription, select the one to use for the labs.
```
az account set --subscription "<subscription-name-or-id>"
```
Terraform's `azurerm` provider (version 4.x) needs the Subscription ID. Export it and save it in `.bashrc` so every new terminal has it.
```
export ARM_SUBSCRIPTION_ID=$(az account show --query id --output tsv)
echo "export ARM_SUBSCRIPTION_ID=$ARM_SUBSCRIPTION_ID" >> ~/.bashrc
```
```
echo $ARM_SUBSCRIPTION_ID
```
Check whether you are authenticated and can read resources.
```
az group list --output table
```

##### Note: Terraform automatically uses your `az login` session, so no extra credentials are needed for the labs.

#### Optional: Login using a Service Principal
If you cannot use `az login` (for example in automation), create a Service Principal and give Terraform its values through environment variables.
```
az ad sp create-for-rbac --name terraform-sp-YourName --role Contributor --scopes /subscriptions/$ARM_SUBSCRIPTION_ID
```
* The command prints values like the example shown below. Keep them private and never commit them to GitHub.

| `appId` | `password` | `tenant` |
| ------------------ | ------------------------- | ------------------ |
| 00000000-0000-0000-0000-000000000000 | xxxxxxxx~xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx | 11111111-1111-1111-1111-111111111111 |

```
export ARM_CLIENT_ID="<appId>"
export ARM_CLIENT_SECRET="<password>"
export ARM_TENANT_ID="<tenant>"
```

### Task-3: Now we are ready to perform the labs
Create directory as Lab1 & create local file
```
mkdir Lab1 && cd Lab1
```
```
vi local.tf
```
```
resource "local_file" "myfile" {

  filename = "/home/azureuser/test.txt"
  content  = "wel come to terraform"
}
```
```
terraform init
```
```
terraform plan
```
```
terraform apply
```
To check whether the file is created follow the command

```
cd /home/azureuser
ls
```
```
cat test.txt
```
Once Verified, you can destroy the File by changing it to the Lab1 Directory
```
cd Lab1
```
```
terraform destroy
```
```
cd ~
rm -rf Lab1
```
