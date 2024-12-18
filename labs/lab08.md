# Lab 08 - Run DAST on your application

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Prerequisites](#prerequisites)
- [Guide](#guide)
  - [Step 01: Define workload identity with Azure](#step-01-define-workload-identity-with-azure)
  - [Step 02: Add Secrets to your repo](#step-02-add-secrets-to-your-repo)
  - [Step 03: Create a new branch](#step-03-create-a-new-branch)
  - [Step 04: Update build phase on your workflow](#step-04-update-build-phase-on-your-workflow)
  - [Step 05: Add DAST scan to Todo API](#step-05-add-dast-scan-to-todo-api)
  - [Step 06: Check ZAP results](#step-06-check-zap-results)
- [Conclusion](#conclusion)

## Learning Objectives

- Run DAST on your application using OWASP ZAP
- Use GitHub Actions to run DAST on your application
- Understand the vulnerabilities found by the DAST scan

## Prerequisites

- Have Azure CLI installed. If you don't have it installed, you can follow the instructions [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

## Guide

### Step 01: Define workload identity with Azure

To run DAST on your application, you need to create a workload identity on Azure to allow GitHub Actions to interact with Azure.

That is needed to run your Terraform code to create the resources needed to run the DAST scan and to do the application deployment.

On your machine, run the following command to login on Azure:

```bash
az login
```

And follow the instructions to login on Azure.

Then, run the following command to create a Service Principal:

```bash
az ad sp create-for-rbac --name "GitHubActions_<your_name>" --role contributor --scopes /subscriptions/<your-subscription-id>
```

Replace `<your_name>` with your name and `<your-subscription-id>` with your subscription ID.

This command will return a JSON with the Service Principal information. Save this information, you'll need it later.

Now create a JSON file named `policy_main.json` and add the following content on it:

```json
{
  "name": "gh-repo-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<GH-USERNAME>/<GH-REPO>:ref:refs/heads/main",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
```

Then, create a JSON file name `policy_pr.json` and add the following content on it:

```json
{
  "name": "gh-repo-pr",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<GH-USERNAME>/<GH-REPO>:pull_request",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
```

On the file you should replace the following placeholders:

- GH-USERNAME, with your GitHub username
- GH-REPO, with your GitHub repository name

The first policy is to allow the Service Principal to access the repository on the `main` branch and the second policy is to allow the Service Principal to access the repository on the Pull Request.

Run the following command to create the federated credential for the Service Principal:

```bash
az ad app federated-credential create --id <OBJECT_ID> --parameters @policy_main.json
az ad app federated-credential create --id <OBJECT_ID> --parameters @policy_pr.json
```

The `<OBJECT_ID>` is the `appId` from the Service Principal JSON.

### Step 02: Add Secrets to your repo

Now, you need to add the Service Principal information as secrets on your repo.

Navigate to your repo on GitHub and click on `Settings` on the toolbar.

Then, click on `Secrets and variables` on the left side menu and then click on `Actions`.

Now you click on `New repository secret` and add the following secrets:

- `AZURE_SUBSCRIPTION_ID`: The `subscriptionId` from the Service Principal JSON
- `AZURE_TENANT_ID`: The `tenant` from the Service Principal JSON
- `AZURE_CLIENT_ID`: The `appId` from the Service Principal JSON

Finally, create a new secret to store database password:

- `DB_PASSWORD`: The password you want to use for the database

### Step 03: Create a new branch

You need to create a new branch to start to develop any additional code since you enable the need to use Pull Requests to update `main` branch.

To be sure you have last version, do a clean up on your local repo. First, move your repo to `main` branch.

```bash
git checkout main
```

Then, get all update from this remote repo.

```bash
git pull
```

Now you're ready to create a new branch named `add-dast`.

```bash
git checkout -b add-dast
```

### Step 04: Update build phase on your workflow

Open the file `.github/workflows/todo-api-pr.yml` and add the following `env` block after the `on` block:

```yaml
env:
  ARTIFACT_NAME: todo-api
```

Then, on the `build` job, add the following steps after the `Test Report` step:

```yaml
- name: Publish
  run: |
    dotnet publish --no-build src/TodoAPI/TodoAPI.csproj -o src/TodoAPI/publish

- uses: actions/upload-artifact@v4
  with:
    name: ${{ env.ARTIFACT_NAME }}
    path: src/TodoAPI/publish
```

### Step 05: Add DAST scan to Todo API

Edit the file `.github/workflows/todo-api-pr.yml` and add the following job after the `run-container-scan` job:

```yaml
dast:
    runs-on: ubuntu-latest
    needs: build
    permissions:
      id-token: write
      contents: read
      checks: write
      issues: write
    
    env:
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      ARM_USE_OIDC: true

    defaults:
      run:
        working-directory: ./deploy/terraform/todo-api

    steps:
    - uses: actions/checkout@v3

    - uses: actions/download-artifact@v4
      with:
        name: ${{ env.ARTIFACT_NAME }}
        path: ./${{ env.ARTIFACT_NAME }}
    
    - uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    
    - uses: hashicorp/setup-terraform@v2
      with:
        terraform_wrapper: false

    - name: terraform init
      run: terraform init
    
    - name: terraform plan
      id: tfplan
      run: terraform plan -var="dbPassword=${{ secrets.DB_PASSWORD }}" -var="env=dast" -out=tfplan

    - name: Terraform Plan Status
      if: steps.tfplan.outcome == 'failure'
      run: exit 1

    - name: terraform apply
      run: |
        terraform apply tfplan
        echo "WEBAPP_NAME=$(terraform output -raw webappName)" >> $GITHUB_ENV
        echo "DB_ADDRESS=$(terraform output -raw dbAddress)" >> $GITHUB_ENV
        echo "WEBAPP_URL=$(terraform output -raw webappUrl)" >> $GITHUB_ENV

    - name: 'Azure webapp deploy - Staging'
      id: stg-deploy
      uses: azure/webapps-deploy@v2
      with: 
        app-name: ${{ env.WEBAPP_NAME }}
        package: ./${{ env.ARTIFACT_NAME }}
  
    - name: 'Configure azure webapp - Staging'
      uses: azure/appservice-settings@v1
      with:
        app-name: ${{ env.WEBAPP_NAME }}
        mask-inputs: false
        app-settings-json: '[{"name": "ConnectionStrings__TodosDb","value": "Server=${{ env.DB_ADDRESS }};Database=TodoDB;Port=5432;User Id=dbadmin;Password=${{ secrets.DB_PASSWORD }};Ssl Mode=VerifyFull;","slotSetting": true}, {"name": "ASPNETCORE_ENVIRONMENT", "value": "Development"}]'
  
    - name: logout
      run: |
        az logout

    - name: ZAP Scan
      uses: zaproxy/action-api-scan@v0.9.0
      with:
        target: 'https://${{ env.WEBAPP_URL }}/swagger/v1/swagger.json'
        format: 'openapi'

    - name: terraform destroy
      run: |
        terraform destroy tfplan
```

Let's review some parts of this job.

First, the permissions:

```yaml
permissions:
  id-token: write
  contents: read
  checks: write
  issues: write
```

The `id-token` permission is needed to get an ID token to authenticate on Azure. The `issues` permission is needed to write the results of the scan as an issue.

Then, the environment variables:

```yaml
env:
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  ARM_USE_OIDC: true
```

This block will set the environment variables needed to authenticate on Azure. Will be used to run Terraform and deploy the application.

Then, the steps related with Terraform:

```yaml
- uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
- uses: hashicorp/setup-terraform@v2
  with:
    terraform_wrapper: false

- name: terraform init
  run: terraform init

- name: terraform plan
  id: tfplan
  run: terraform plan -var="dbPassword=${{ secrets.DB_PASSWORD }}" -var="env=dast" -out=tfplan

- name: Terraform Plan Status
  if: steps.tfplan.outcome == 'failure'
  run: exit 1

- name: terraform apply
  run: |
    terraform apply tfplan
    echo "WEBAPP_NAME=$(terraform output -raw webappName)" >> $GITHUB_ENV
    echo "DB_ADDRESS=$(terraform output -raw dbAddress)" >> $GITHUB_ENV
    echo "WEBAPP_URL=$(terraform output -raw webappUrl)" >> $GITHUB_ENV
```

This block will authenticate on Azure, run Terraform to create the resources needed to run the DAST scan.

If the Terraform plan fails, the job will exit with an error.

And on last step, you get information for the output of the Terraform apply to use on the next steps.

Then, the deployment of the application:

```yaml
- name: 'Azure webapp deploy - Staging'
  id: stg-deploy
  uses: azure/webapps-deploy@v2
  with: 
    app-name: ${{ env.WEBAPP_NAME }}
    package: ./${{ env.ARTIFACT_NAME }}

- name: 'Configure azure webapp - Staging'
  uses: azure/appservice-settings@v1
  with:
    app-name: ${{ env.WEBAPP_NAME }}
    mask-inputs: false
    app-settings-json: '[{"name": "ConnectionStrings__TodosDb","value": "Server=${{ env.DB_ADDRESS }};Database=TodoDB;Port=5432;User Id=dbadmin;Password=${{ secrets.DB_PASSWORD }};Ssl Mode=VerifyFull;","slotSetting": true}, {"name": "ASPNETCORE_ENVIRONMENT", "value": "Development"}]'

- name: logout
  run: |
    az logout
```

This block will deploy the application on Azure and configure the connection string to the database.

This configuration is done using the properties that you got from Terraform code.

Finally, the DAST scan:

```yaml
- name: ZAP Scan
  uses: zaproxy/action-api-scan@v0.9.0
  with:
    target: 'https://tbernardo-todoapp-api-dast.azurewebsites.net/swagger/v1/swagger.json'
    format: 'openapi'
```

After everything is deployed, the job will run a `terraform destroy` to remove all resources created.

With this approach, you use the concept of a short-lived (disposable) environment to run the DAST scan.

Let's commit the changes and push to the remote repository:

```bash
git add .github/workflows/todo-api-pr.yml
git commit -m "Add DAST scan to Todo API"
git push origin add-dast
```

Now you can create a Pull Request to merge your changes to the `main` branch.

As soon as the PR is created, you can check the progress of the workflow on the Actions tab of your repo.

### Step 06: Check ZAP results

After the pipeline runs, you can navigate to the issues tab on your repo and check the issue created by the ZAP scan.

Please take some time to navigate through the issue content and understand the vulnerabilities found by the scan.

At the end of the issue content, you can see a link to download a report.

This link will send you to the workflow details page and you can download the report from the artifacts list.

Download the report and check the content to understand the vulnerabilities found by the scan. You have access to a HTML report that give you a good overview of the vulnerabilities found.

## Conclusion

Congratulations! You've successfully run a DAST scan on your application using GitHub Actions and OWASP ZAP. With this tool you can check your API regarding OWASP Top 10 vulnerabilities and improve the security of your application.
