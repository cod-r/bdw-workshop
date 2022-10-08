# How to add your ssh key to your GitHub account
[Official Documentation](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account#adding-a-new-ssh-key-to-your-account)

1. Check if you have a ssh key
```sh
ls -al ~/.ssh
```

If you don't have a ssh key generate one:
```sh
ssh-keygen
```

2. Copy the SSH public key to your clipboard.
```sh
cat ~/.ssh/id_rsa.pub
```
Select and copy the contents of the output displayed in the terminal to your clipboard.

3. In the upper-right corner of any page, click your profile photo, then click `Settings`.

4. In the "Access" section of the sidebar, click `SSH and GPG keys`.

5. Click `New SSH key`.

6. Paste your key into the "Key" field.

7. Click `Add SSH key`. 

8. Test connection.
```
ssh -T git@github.com
```
Example output:
```
Hi cod-r! You've successfully authenticated, but GitHub does not provide shell access.
```