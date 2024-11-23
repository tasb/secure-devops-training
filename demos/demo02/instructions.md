# Demo 02 - Secret Scanning

## Push files with secrets

- Show Azure SP and GH Token files
- Commit and push files
- Show the files on the repo
- Show the PAT

## Change repo visibility

- Change repo visibility to public
- Check Code Security tab
- Enable secret scanning
- Check Security tab
- Check what happened with GH PAT Token

## Try to push the files again

- Generate new PAT
- Add to a file
- Run az command to create new sp: `az ad sp create-for-rbac -n tbernardo_sp4 > azure_sp2.json`
- Try to push the files again
- Check error on push
- Force the GH PAT to be added
- Check the repo and what happened to the PAT
