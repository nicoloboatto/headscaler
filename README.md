# headscaler
An ansible script to facilitate the deployment of a dockerized headscale stack.

The final product should be a server running the following:

- Docker + Compose
- Headscale
- Headplane GUI
- Traefik Reverse Proxy
- Tailscale Node

# Step-by-Step First Setup

Create a sudoer user that ansible will use to log in and deploy the stack. **SSH root access will be disabled!**
```
useradd -m -s /bin/bash headscaler
visudo
```

Edit the file and add the following line:
```
# See sudoers(5) for more information on "@include" directives:
@includedir /etc/sudoers.d
headscaler ALL=(ALL) NOPASSWD: ALL
```

Add your public key to the /home/user/.ssh/authorized_keys file.

Clone repo on your control machine.

Edit inventory (inventory.ini)
```
[headscale_stack]
vpn-01 ansible_host=<domain>
```

Edit variables (domains, passwords) in config.yml.

Install Ansible collections:
```
ansible-galaxy collection install devsec.hardening community.docker
ansible-galaxy role install geerlingguy.docker
```

Dry-run
```
ansible-playbook -i inventory.ini deploy_headscale_stack.yml --check
```

Deploy
```
ansible-playbook -i inventory.ini deploy_headscale_stack.yml
```

Verify

```
https://hs.<domain> → Tailscale clients can log in.
https://plane.<domain> → Basic-Auth (admin / password)
```

# Update

Update Ansible collections:
```
ansible-galaxy collection install -U devsec.hardening community.docker
ansible-galaxy role install -U geerlingguy.docker
```

Pull latest version of the playbook or change versions manually and rerun the playbook.

# DNS

Records can be set from the headplane UI or from the `headscale/extra-records.json` file.