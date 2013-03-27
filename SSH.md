## SSH Configuration

In order to be able to log in as this new user via `ssh` instead of `ubuntu`, you'll need to do some SSH magic.  When you're logged in as `ubuntu`, do the following:

```bash
$ sudo su admin
$ cd ~
$ mkdir .ssh
$ sudo cp /home/ubuntu/.ssh/authorized_keys /home/admin/.ssh
$ sudo chown admin:admin /home/admin/.ssh/authorized_keys
```

Once the `authorized_keys` file exists, you will need to get your public key out of the file `~/.ssh/id_rsa.pub` on your local machine and paste it into the `authorized_keys` file on the Ubuntu machine.  **There must be a newline at the end of the line in the file or else it won't work.**

After this, you should be able to log directly in to the Ubuntu box as user `admin` without needing a password.
