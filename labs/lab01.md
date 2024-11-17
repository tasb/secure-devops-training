# Lab 01 - Prepare your development environment

## Table of Contents

- [Goals](#goals)
- [:warning: Important Information](#warning-important-information)
- [Pre-requisites](#pre-requisites)
- [Guide](#guide)
  - [Step 01: Create a new repository](#step-01-create-a-new-repository)
  - [Step 02: Install Talisman](#step-02-install-talisman)
  - [Step 03: Test Talisman on your repository](#step-03-test-talisman-on-your-repository)
  - [Step 04: Configure exceptions for Talisman](#step-04-configure-exceptions-for-talisman)
  - [Step 05: Push your changes to the remote repository](#step-05-push-your-changes-to-the-remote-repository)

## Goals

- Install Talisman
- Test Talisman on your repository
- Configure exceptions for Talisman

## :warning: Important Information

**:warning: Please read this carefully before you proceed!**

For the labs of this course is recommended that you have a Linux based environment.

All labs instructions will be based on a Linux environment, so if you want to use another OS you need to adjust by yourself the commands and installation processes.

If you're using Windows, you can use the Windows Subsystem for Linux (WSL) to have a Linux environment on your machine.

If you cannot install WSL, you can create a virtual machine on your own machine or deploy on the Azure account provided to you.

These labs where tested on a Ubuntu 20.04 LTS environment on WSL and MacOS with Apple Silicon.

## Pre-requisites

- [ ] Have a GitHub Account
- [ ] Install [Visual Studio Code](https://code.visualstudio.com/)
  - [ ] [Connect Visual Studio Code to WSL](https://code.visualstudio.com/docs/remote/wsl)
- [ ] Install [Git](https://git-scm.com/)

## Guide

### Step 01: Create a new repository

On your GitHub account, create a new repository named `secure-devops-labs`.

Repository properties:

- Visibility: Private
- Initialize this repository with a README: Yes
- Add `.gitignore`: None

After creating the repository, clone it to your local machine.

### Step 02: Install Talisman

Talisman is a tool that scans your Git commits for secrets.

To install Talisman, follow the instructions on the [Talisman website](https://thoughtworks.github.io/talisman/docs/installation/global-hook/), depending on your operating system.

You must opt for Global Installation as a pre-commit hook.

For installation on Linux, you can use the following command:

```bash
curl --silent  https://raw.githubusercontent.com/thoughtworks/talisman/main/global_install_scripts/install.bash > /tmp/install_talisman.bash && /bin/bash /tmp/install_talisman.bash
```

During the installation process, you will need to answer some questions.

When you get this prompt:

```bash
PLEASE CHOOSE WHERE YOU WISH TO SET TALISMAN_HOME VARIABLE AND talisman binary PATH (Enter option number):
1) Set TALISMAN_HOME in ~/.bashrc
2) Set TALISMAN_HOME in ~/.bash_profile
3) Set TALISMAN_HOME in ~/.profile
4) I will set it later
```

You should select the option `1` to add the Talisman path to your `.bashrc` file and make it available for all terminal sessions.

Make sure that the lines added to your `.bashrc` file are similar to the following:

```bash
# >>> talisman >>>
# Below environment variables should not be modified unless you know what you are doing
export TALISMAN_HOME=/home/user/.talisman/bin
alias talisman=$TALISMAN_HOME/talisman_darwin_arm64
export TALISMAN_INTERACTIVE=true
# <<< talisman <<<
```

Please confirm you have this line: `export TALISMAN_INTERACTIVE=true` in your `.bashrc` file. This line is important to enable the interactive mode of Talisman.

If not, please added it manually.

Then you get a prompt asking if you want to have interactive mode enabled. You should select `Y` to have the interactive mode enabled.

When you get this prompt:

```bash
Please enter root directory to search for git repos (Default: /Users/tiago.bernardo):
```

You can set for the folder where you cloned your repo to only be assigned to that repo and not all repos on your machine.

There are other questions during the installation process, but you can leave the default values or change them according to your preferences.

To finalize the installation, you need to restart your terminal session.

### Step 03: Test Talisman on your repository

Add a file named `not-secret.txt` to your repository and add some random text without any sensitive information.

Then create a new file named `secret.txt` and add the following content:

```text
password=my-secret-password
```

Finally, add a new file named `appsettings.json` and add the following content:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "TodoDb": "Server=myServerAddress;Database=myDataBase;User Id=myUsername;Password=myPassword;"
  },
  "AllowedHosts": "*",
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://localhost:5400"
      },
      "Https": {
        "Url": "https://localhost:5401"
      }
    }
  }
}
```

Now commit these files and check the output of the Talisman scan.

### Step 04: Configure exceptions for Talisman

When you have done the commit, Talisman should start an interactive mode to add the files you want to ignore.

You should get a prompt like this:

```bash
==== Interactively adding to talismanrc ====

filename: appsettings.json
checksum: 28a51c0a9e3fec5f02ab740c420852827832607dad2e1d7ef81ac5a266675d5d

? Do you want to add labs/lab01.md with above checksum in talismanrc ?
```

You should reply `yes` to the `appsettings.json` file and `no` to the other files.

After finishing this process, you should see a new file on your repository named `.talismanrc`.

Please take a look at the content of this file and check if the `appsettings.json` file is there.

### Step 05: Push your changes to the remote repository

To proceed and to have talisman allowing you to proceed with the commit, you need can delete the `secret.txt` file or add it to the `.gitignore` file.

We recommend adding it to the `.gitignore` file because if you have it on your repository you may need it to your local development environment.

So create a `.gitignore` file and add the following content:

```text
secret.txt
```

Now, let's commit the changes and push them to the remote repository.

Congratulations! You have completed the first lab of this course and your local repo is much more secure now!
