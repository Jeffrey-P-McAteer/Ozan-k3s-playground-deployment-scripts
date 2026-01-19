
# k3s Playground for Ozan

OS: Ubuntu 25.04

Wildcard DNS: `*.ozan.jmcateer.com` and `ozan.jmcateer.com` will route to the machine's public ipv4 and ipv6 address.

Username: `ubuntu`

Access

```bash
# Least amount of config
ssh -i /path/to/my/id_ed25519 ubuntu@ozan.jmcateer.com

# If you're lazy like me, place the following in $HOME/.ssh/config (works on windows as well)
host ozan.jmcateer.com
  User ubuntu
  IdentityFile /path/to/my/id_ed25519

# Then you can do
ssh ozan.jmcateer.com
```


# Installing & Using k3s





