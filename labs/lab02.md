# Lab 02: Enable Secret Scanning

In this lab you'll learn how to use GitHub Secret Scanning to protect your secrets on the repository.

## Table of Contents

- [Goals](#goals)
- [Pre-requisites](#pre-requisites)
- [Guide](#guide)
  - [Step 01: Create a new GitHub PAT](#step-01-create-a-new-github-pat)
  - [Step 02: Create a new Azure SP](#step-02-create-a-new-azure-sp)
  - [Step 03: Push files with secrets](#step-03-push-files-with-secrets)
  - [Step 04: Change repo visibility](#step-04-change-repo-visibility)
  - [Step 05: Review Security alerts](#step-05-review-security-alerts)
  - [Step 06: Try to push the files again](#step-06-try-to-push-the-files-again)
  - [Step 07: Review Security alerts](#step-07-review-security-alerts)
  - [Step 08: Clean up the repo](#step-08-clean-up-the-repo)
  - [Step 09: Clean git history](#step-09-clean-git-history)
  - [Step 10: Check the repo](#step-10-check-the-repo)
- [Conclusion](#conclusion)

## Goals

- Push files with secrets to a repository
- Change the repository visibility to public
- Enable secret scanning on the repository
- Check the security alerts
- Check how GitHub automatically revokes the PAT
- Check ou GitHub  Secret Scanning control your push
- Clean up the repo

## Pre-requisites

- GitHub account
- Azure account
- Azure CLI installed
- Finished [Lab 01](lab01.md) and navigate to the repo used

## Guide

### Step 01: Create a new GitHub PAT

Go to your GitHub account and click on your profile picture.

Click on `Settings`.

On the left side, click on `Developer settings`.

Click on `Personal access tokens > Tokens (classic)`.

Click on `Generate new token > Generate New Token (classic)`.

Add a name for your token, scroll down and click on `Generate token`.

You'll get a token and copy it.

On the repo used on last lab, create a file called `gh-token` and paste the token.

### Step 02: Create a new Azure SP

Open your terminal and run the following command:

```bash
az ad sp create-for-rbac -n <your_name>_sp > azure_sp.json
```

Replace `<your_name>` with your name using first letter of your first name and full last name.

Check that you may have to login to your Azure account and use the credentials provided by training team. You should not use your company credentials.

Check the content of the file `azure_sp.json`. You should get a JSON with the following content:

```json
{
  "appId": "your_app_id",
  "displayName": "your_name_sp",
  "password": "your_password",
  "tenant": "your_tenant"
}
```

### Step 03: Push files with secrets

Add the files `gh-token` and `azure_sp.json` to the repo.

No, let's commit the files using the following commands:

```bash
git add gh-token azure_sp.json
git commit -m "Add secrets"
```

You should get Talisman warnings, please ignore them for now and add both files to the `.talismanrc` file.

Now push the files to the remote repository:

```bash
git push
```

### Step 04: Change repo visibility

Go to your repo on GitHub and click on `Settings`.

Scroll down on that page and find a button named `Change visibility`.

Follow the steps to change the repo visibility to public.

Then, select the option `Code scanning` on left menu of `Settings`.

Scroll down until find `Secret scanning` block and enable it and `Push Protection` too.

### Step 05: Review Security alerts

Go to the `Security` tab on the repo (on the menu where you find the `Code` tab).

You should see two security alerts for the files you've pushed.

Navigate on them and check the details.

Now return to the place where you create you GitHub PAT token and check it was automatically revoked.

### Step 06: Try to push the files again

Now repeat the steps [Step 01](#step-01-create-a-new-github-pat) and [Step 02](#step-02-create-a-new-azure-sp) to get new tokens.

When creating the SP, please pay attention on the SP name that must be different from the first one.

On both steps you should create new files to add the data: `gh-token2` and `azure_sp2.json`.

Now let's commit the files using the following commands:

```bash
git add gh-token2 azure_sp2.json
git commit -m "Add secrets"
```

You should get Talisman warnings, please ignore them for now and add both files to the `.talismanrc` file.

Now push the files to the remote repository:

```bash
git push
```

You should get an error on the push. Check the error message and follow the instructions to force the add of both files.

### Step 07: Review Security alerts

Go to the `Security` tab on the repo (on the menu where you find the `Code` tab).

You should see two new security alerts for the files you've pushed.

Navigate on them and check the details.

Check again the place where you create you GitHub PAT token and check it was automatically revoked.

### Step 08: Clean up the repo

Delete the files `gh-token`, `azure_sp.json`, `gh-token2` and `azure_sp2.json` from the local repo.

Commit the changes and push them to the remote repository, using the following commands:

```bash
git add -A
git commit -m "Clean up secrets"
git push
```

### Step 09: Clean git history

Even though you've deleted the files from the repo, they are still on the git history.

If you clone the repo again, you'll get the files and you can navigate back on the history to get them.

So you need to be sure that the files are not on the git history.

In this case, knowing that repo don't have too many commits, you could use a `git rebase` to remove the files from the history.

But when you have a lot of commits, you should use a tool like `git filter-branch` to remove the files.

Let's check how that tool works.

On your local repo, run the following command:

```bash
git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch gh-token azure_sp.json gh-token2 azure_sp2.json' --prune-empty --tag-name-filter cat -- --all
```

Now, push the changes to the remote repository:

```bash
git push origin --force --all
```

### Step 10: Check the repo

Go to the repo on GitHub and check that the files are not there anymore.

Navigate on the history and check that the files are not there anymore.

Clone the repo again on a new folder and check that the files are not there anymore.

## Conclusion

You've learned how to use GitHub Secret Scanning to protect your secrets on the repository.

You've also learned how to clean up the git history to remove files that were added by mistake.
