
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

# Initial Setup

Use `ssh-copy-id` to copy a key you own to the server - this is the first & last time we use the VPS-provided password.

```bash
ssh-copy-id -i /path/to/my/id_ed25519 ubuntu@ozan.jmcateer.com
```

Remove password auth from the server.

```bash
sudo vim /etc/ssh/sshd_config
# Set "PasswordAuthentication no"
sudo systemctl restart sshd
```

# Installing & Using k3s

```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg lsb-release
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 6443
curl -sfL https://get.k3s.io | sh -

```



