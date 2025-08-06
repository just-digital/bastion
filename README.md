# Bastion Hardening
First line of defense for a bastion

```bash
$ sudo su -
$ wget https://raw.githubusercontent.com/just-digital/bastion/refs/heads/main/configure.sh
$ sh configure.sh <admin-user-name> <comma seperated IP whitelist>
```
