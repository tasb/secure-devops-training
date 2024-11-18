# Use of Talisman as a pre-commit hook

## Show Talisman website

- Go to [Talisman website](https://thoughtworks.github.io/talisman/)
- Describe the concept of Talisman
- Show the installation steps and the different ways to install

## Look at secure-devops-labs

- Check there are some files already added to the repo
- Look into the `.git/hooks` folder and show the symlink to the `pre-commit` script
- Show the `pre-commit` script

## Disable the pre-commit hook

- Disable hook with command `mv .git/hooks/pre-commit .git/hooks/pre-commit.bak`
- Add two files:
  - `git add secret-files/my-cert.pem`
  - `git add secret-files/secret`
- Commit files: `git commit -m "Add secret files"`
- Push the commit to the remote repository: `git push`

## Enable the pre-commit hook

- Enable hook with command `mv .git/hooks/pre-commit.bak .git/hooks/pre-commit`
- Add all files: `git add -A`
- Commit files: `git commit -m "Add secret files"`
- Show the talisman error message
- Show talisman interactive mode to add files to ignore mode

## Run talisman on the command line

- Run `talisman -s` on the command line
- Show the produced report
- Add the folder to the .gitignore file
