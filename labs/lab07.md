# Lab 07 - Secure your Containers

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Guide](#guide)
  - [Step 01: Create a new branch](#step-01-create-a-new-branch)
  - [Step 02: Update GitHub PR workflows](#step-02-update-github-pr-workflows)
  - [Step 03: Review scanning results](#step-03-review-scanning-results)
  - [Step 04: Shift-left Trivy](#step-04-shift-left-trivy)
- [Conclusion](#conclusion)

## Learning Objectives

- Run Trivy on your image using GitHub Actions
- Shift-left Trivy and run it on your Dockerfile

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

Now you're ready to create a new branch named `add-trivy`.

```bash
git checkout -b add-trivy
```

### Step 02: Update GitHub PR workflows

Now you need to update your GitHub Actions workflows to perform a scan on your Terraform code.

Edit `.github/workflows/todo-api-pr.yml` file and add a new stage.

```yaml
  run-container-scan: 
    permissions:
      contents: read
      security-events: write
      actions: read

    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: build local container
        uses: docker/build-push-action@v2
        with:
          context: src/TodoAPI
          tags: todo-api:trivy
          push: false
          load: true
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: todo-api:trivy
          format: 'sarif'
          output: 'trivy-results.sarif'
          vuln-type: 'os,library'

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        if: success() || failure()
        with:
          sarif_file: trivy-results.sarif
      
      - name: Publish security report to artifact
        uses: actions/upload-artifact@v4
        if: success() || failure()
        with:
          name: trivy-report
          path: trivy-results.sarif
```

Please pay attention to the indentation and the order of the steps. This new stage should be at same indentation level of the other stages and after the `build` stage.

The `Run Trivy vulnerability scanner` will scan your Terraform code and generate a SARIF file that will be uploaded to your repo as an artifact.

Now, edit `.github/workflows/todo-webapp-pr.yml` file and add a new stage.

```yaml
  run-container-scan: 
    permissions:
      contents: read
      security-events: write
      actions: read

    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: build local container
        uses: docker/build-push-action@v2
        with:
          context: src/TodoWebapp
          tags: todo-webapp:trivy
          push: false
          load: true
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: todo-webapp:trivy
          format: 'sarif'
          output: 'trivy-results.sarif'
          vuln-type: 'os,library'

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        if: success() || failure()
        with:
          sarif_file: trivy-results.sarif
      
      - name: Publish security report to artifact
        uses: actions/upload-artifact@v4
        if: success() || failure()
        with:
          name: trivy-report
          path: trivy-results.sarif
```

Finally, you can commit and push your changes to your repo.

```bash
git add -A
git commit -m "Add Terraform scripts and update GitHub Actions workflows"
git push origin add-trivy
```

Navigate to your repo on GitHub and create a new Pull Request to merge your changes to `main` branch.

### Step 03: Review scanning results

After the PR is created, you need to wait for the pipeline to run and scan your Terraform code.

When you see that all checks run successfully, you can navigate to the Actions menu and check the logs of the `run-container-scan` stage.

Additionally, you have access to the results on two other ways.

First, you navigate to the `Security` tab on your repo and then select the `Code scanning` option. You will see the results of the scan.

When you get access to that list of alerts, you can try to do some filters to help you understand the issues found.

For instance, you can filter by `Tool` to see only the results from Checkov. On the search bar, you can type `tool` and you'll get a list of tools used on your repo. Select `trivy` and you'll see only the results from Checkov.

The second way is to analyse the SARIF file generated by the scan. You can download this file from the `Artifacts` section on the details of your workflow run.

You can access it through the `Actions` tab on your repo, then select the workflow run you want to check and navigate to the `Artifacts` section.

Download the file and then use a recommended extension for VS Code to view the content of this file. The extension is called [`SARIF Viewer`](https://marketplace.visualstudio.com/items?itemName=MS-SarifVSCode.sarif-viewer).

On these workflows you're checking and scanning all your code and getting feedback about the quality of your code. Although you're not enforcing the workflow to fail because we'll not fix the code but breaking the workflow can be a good practice to ensure that the code is being reviewed.

### Step 04: Shift-left Trivy

You can have an integration at your IDE to run Trivy on your Dockerfile. This way you can have a faster feedback about the vulnerabilities on your code.

First, you need to install the Trivy CLI on your machine. You can follow the instructions on the [official documentation](https://trivy.dev/latest/getting-started/installation/).

Make sure that trivy is on your PATH.

Then install the `Trivy` extension on your VS Code. You can find it on the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=AquaSecurityOfficial.trivy-vulnerability-scanner).

On that page, click on install and you have the extension installed on your VS Code.

On the left side of your VS Code, you'll see a new icon with the Trivy logo. Click on it and you'll see a new panel on your VS Code.

You need to click on `Run Trivy against workspace` to scan your Dockerfiles on your workspace.

Than you can have a list of the findings and an explanation for each one.

## Conclusion

You have learned how to create Terraform scripts to deploy your infrastructure and how to scan your Terraform code using Checkov.
