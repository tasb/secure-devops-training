# Lab 06 - Secure your Infrastructure as Code

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Guide](#guide)
  - [Step 01: Create a new branch](#step-01-create-a-new-branch)
  - [Step 02: Create Terraform scripts](#step-02-create-terraform-scripts)
  - [Step 03: Update GitHub PR workflows](#step-03-update-github-pr-workflows)
  - [Step 04: Review scanning results](#step-04-review-scanning-results)
- [Conclusion](#conclusion)

## Learning Objectives

- Create Terraform scripts to deploy your infrastructure
- Scan your Terraform code using Checkov
- Analyze the results of the scan

## Guide

### Step 01: Create a new branch

You need to create a new branch to start to develop any additional code since you enable the need to use Pull Requests to update `main` branch.

To be sure you have last version, do a clean up on your local repo. First, move your repo to `main` branch.

```bash
git checkout main
```

Then, get all update from this remote repo.

```bash
git pull
```

Now you're ready to create a new branch named `add-iac`.

```bash
git checkout -b add-iac
```

### Step 02: Create Terraform scripts

Create two new folders:

1) `deploy` > `terraform` > `todo-api`
2) `deploy` > `terraform` > `todo-webapp`

Inside each folder you will create the Terraform files needed to deploy (and then destroy) your infrastructure:

- `main.tf`: main file where you define resources to be created
- `variables.tf`: each block of this file defines a variable/parameter to be used on your IaC process to make it more dynamic
- `terraform.tfvars`: where you define default values for variables/parameters defined on `variables.tf` file
- `output.tf`: where you define outputs of your IaC scripts that can be used during your CD process to set some variables
- `versions.tf`: where you define the version of Terraform to be used on this folder

On `todo-api` folder, create a file named `variables.tf` with following content.

```terraform
variable "env" {
    type = string
    description = "Environment name to deploy"
    nullable = false
}

variable "location" {
    type = string
    description = "The Azure Region in which all resources in this example should be created."
    default = "westeurope"
}

variable "appName" {
    type = string
    description = "Application Name"
    nullable = false
}

variable "appServiceName" {
    type = string
    description = "App Service Name"
    nullable = false
}

variable "dbName" {
    type = string
    description = "Db Name"
    nullable = false
}

variable "dbAdmin" {
    type = string
    description = "Db Username"
    nullable = false
}

variable "dbPassword" {
    type = string
    description = "Db User Password"
    nullable = false
    sensitive = true
}
```

Then create a `terraform.tfvars` file with the following content. Pay attention where you need to replace `<your-prefix>` with your unique prefix.

```terraform
appName = "<your-prefix>-todoapp"
appServiceName = "api"
dbName = "<your-prefix>-todoapp-db"
dbAdmin = "dbadmin"
```

After that, let's define your outputs that will be used during CD. Create a file named `output.tf` with following content.

```terraform
output "webappName" {
    value = "${azurerm_linux_web_app.webapp.name}"
}

output "webappUrl" {
    value = "${azurerm_linux_web_app.webapp.name}.azurewebsites.net"
}

output "dbAddress" {
    value = "${azurerm_postgresql_flexible_server.db.fqdn}"
}
```

Now, let's create `versions.tf` file where you define the version of Terraform to be used on this folder.

```terraform
terraform {
}

provider "azurerm" {
  features {}
}
```

Finally, let's create `main.tf` where you define the resources to be created, uses variables defined previously and set outputs at the end of the execution.

The file must have the following content and you need to pay attention where you need to replace `<your-prefix>` with your unique prefix.

```terraform
# Creates a Resource Group to group the following resources
resource "azurerm_resource_group" "rg" {
  name     = "${var.appName}-${var.appServiceName}-${var.env}-rg"
  location = var.location
}

# Create the Linux App Service Plan
resource "azurerm_service_plan" "appserviceplan" {
  name                = "asp-${var.appName}-${var.appServiceName}-${var.env}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "B2"
}

#Create the web app, pass in the App Service Plan ID
resource "azurerm_linux_web_app" "webapp" {
  name                = "${var.appName}-${var.appServiceName}-${var.env}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.appserviceplan.id

  site_config {
    always_on = true
    application_stack {
      dotnet_version = "6.0"
    }
  }
}

# Create Azure Database for PostgreSQL server. Database will be automatically created by Todo API code
resource "azurerm_postgresql_flexible_server" "db" {
  name                   = "${var.dbName}-${var.env}-psql"
  resource_group_name    = azurerm_resource_group.rg.name
  location               = azurerm_resource_group.rg.location
  version                = "14"
  administrator_login    = "${var.dbAdmin}"
  administrator_password = "${var.dbPassword}"
  zone                   = "1"

  storage_mb = 32768

  sku_name   = "B_Standard_B1ms"
}

# Add firewall rule on your Azure Database for PostgreSQL server to allow other Azure services to reach it
resource "azurerm_postgresql_flexible_server_firewall_rule" "example" {
  name             = "AllowAllAzureServicesAndResourcesWithinAzureIps"
  server_id        = azurerm_postgresql_flexible_server.db.id
  start_ip_address = "0.0.0.0"
  end_ip_address   = "0.0.0.0"
}
```

If you take a look again on `variables.tf` file you may find the following variable.

```terraform
variable "env" {
    type = string
    description = "Environment name to deploy"
    nullable = false
}
```

With this variables used as a input parameter you may reuse the same Terraform code to create your infrastructure for Staging and Production environment, following best practices on IaC approach that states that the way you create your resources on several environments must be always the same.

Now let's add the files to create Todo Webapp infrastructure.

On `todo-webapp` folder, create a file named `variables.tf` with following content.

```terraform
variable "env" {
    type = string
    description = "Environment name to deploy"
    nullable = false
}

variable "appName" {
    type = string
    description = "Application Name"
    nullable = false
}

variable "appServiceName" {
    type = string
    description = "Application Name"
    nullable = false
}

variable "apiName" {
    type = string
    description = "Application Name"
    nullable = false
}

variable "location" {
    type = string
    description = "The Azure Region in which all resources in this example should be created."
    default = "westeurope"
}
```

Then create a `terraform.tfvars` file with the following content. Pay attention where you need to replace `<your-prefix>` with your unique prefix.

```terraform
appName = "<your-prefix>-todoapp"
appServiceName = "webapp"
apiName = "api"
```

After that, let's define your outputs that will be used during CD. Create a file named `output.tf` with following content.

```terraform
output "webappName" {
    value = "${azurerm_linux_web_app.webapp.name}"
}

output "webappUrl" {
    value = "${azurerm_linux_web_app.webapp.name}.azurewebsites.net"
}

output "webapiUrl" {
    value = "${var.appName}-${var.apiName}-${var.env}.azurewebsites.net"
}
```

Finally, let's create `main.tf` where you define the resources to be created, uses variables defined previously and set outputs at the end of the execution.

The file must have the following content and you need to pay attention where you need to replace `<your-prefix>` with your unique prefix.

```terraform
terraform {
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "${var.appName}-${var.appServiceName}-${var.env}-rg"
  location = var.location
}

# Create the Linux App Service Plan
resource "azurerm_service_plan" "appserviceplan" {
  name                = "asp-${var.appName}-${var.appServiceName}-${var.env}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "B2"
}

#Create the web app, pass in the App Service Plan ID, and deploy code from a public GitHub repo
resource "azurerm_linux_web_app" "webapp" {
  name                = "${var.appName}-${var.appServiceName}-${var.env}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.appserviceplan.id

  site_config {
    always_on = true
    application_stack {
      dotnet_version = "6.0"
    }
  }
}
```

You complete the step to create Terraform scripts. But now you only have code, you need to execute this scripts.

Let's proceed with preparing your GitHub Repo adding Environments to help handle deployments.

### Step 03: Update GitHub PR workflows

Now you need to update your GitHub Actions workflows to perform a scan on your Terraform code.

Edit `.github/workflows/todo-api-pr.yml` file and add a new stage.

```yaml
  scan-terraform:
    permissions:
      contents: read
      security-events: write
      actions: read

    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v3

      - name: Checkov GitHub Action
        uses: bridgecrewio/checkov-action@v12
        continue-on-error: true
        with:
          output_format: cli,sarif
          output_file_path: console,results.sarif
          directory: deploy/terraform/todo-api
        
      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        if: success() || failure()
        with:
          sarif_file: results.sarif

      - name: Publish security report to artifact
        uses: actions/upload-artifact@v4
        with:
          name: checkov-report
          path: results.sarif
```

Please pay attention to the indentation and the order of the steps. This new stage should be at same indentation level of the other stages and after the `build` stage.

The `Checkov GitHub Action` will scan your Terraform code and generate a SARIF file that will be uploaded to your repo as an artifact.

You need to update the triggers on this workflow to include the Terraform code folder.

```yaml
  pull_request:
    branches: [ main ]
    paths:
      - 'src/TodoAPI/**'
      - 'src/TodoAPI.Tests/**'
      - '.github/workflows/todo-api.yaml'
      - 'deploy/terraform/todo-api/**'
```

Now, edit `.github/workflows/todo-webapp-pr.yml` file and add a new stage.

```yaml
  scan-terraform:
    permissions:
      contents: read
      security-events: write
      actions: read

    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v3

      - name: Checkov GitHub Action
        uses: bridgecrewio/checkov-action@v12
        continue-on-error: true
        with:
          output_format: cli,sarif
          output_file_path: console,results.sarif
          directory: deploy/terraform/todo-webapp
        
      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        if: success() || failure()
        with:
          sarif_file: results.sarif

      - name: Publish security report to artifact
        uses: actions/upload-artifact@v4
        with:
          name: checkov-report
          path: results.sarif
```

On both workflows, you need to specify the directory to be scanned on `directory` property of `Checkov GitHub Action`. If you have your code on another folder, you need to change this property to point to the correct folder.

You need to update the triggers on this workflow to include the Terraform code folder.

```yaml
  pull_request:
    branches: [ main ]
    paths:
      - 'src/TodoWebapp/**'
      - '.github/workflows/todo-webapp.yaml'
      - 'deploy/terraform/todo-webapp/**'
```

Finally, you can commit and push your changes to your repo.

```bash
git add -A
git commit -m "Add Terraform scripts and update GitHub Actions workflows"
git push origin add-iac
```

Navigate to your repo on GitHub and create a new Pull Request to merge your changes to `main` branch.

### Step 04: Review scanning results

After the PR is created, you need to wait for the pipeline to run and scan your Terraform code.

When you see that all checks run successfully, you can navigate to the Actions menu and check the logs of the `scan-terraform` stage.

Additionally, you have access to the results on two other ways.

First, you navigate to the `Security` tab on your repo and then select the `Code scanning` option. You will see the results of the scan.

When you get access to that list of alerts, you can try to do some filters to help you understand the issues found.

For instance, you can filter by `Tool` to see only the results from Checkov. On the search bar, you can type `tool` and you'll get a list of tools used on your repo. Select `Checkov` and you'll see only the results from Checkov.

The second way is to analyse the SARIF file generated by the scan. You can download this file from the `Artifacts` section on the details of your workflow run.

You can access it through the `Actions` tab on your repo, then select the workflow run you want to check and navigate to the `Artifacts` section.

Download the file and then use a recommended extension for VS Code to view the content of this file. The extension is called [`SARIF Viewer`](https://marketplace.visualstudio.com/items?itemName=MS-SarifVSCode.sarif-viewer).

On these workflows you're checking and scanning all your code and getting feedback about the quality of your code. Although you're not enforcing the workflow to fail because we'll not fix the code but breaking the workflow can be a good practice to ensure that the code is being reviewed.

On these workflows this is ot happening because we're not enforcing the workflow to fail. But you can do this by removing the `continue-on-error: true` property from the `Checkov GitHub Action` step.

## Conclusion

You have learned how to create Terraform scripts to deploy your infrastructure and how to scan your Terraform code using Checkov.
