# headscaler
An ansible script to facilitate the deployment of a dockerized [headscale](https://github.com/juanfont/headscale) + [headplane](https://github.com/tale/headplane) stack.

The final product should be a server running the following:

- Docker + Compose
- Headscale
- Headplane GUI
- Traefik Reverse Proxy
- Tailscale Node

The Headplane GUI will be protected with HTTP Basic Auth. Implementing SSO, if required, is left as an excercise to the reader.

# Step-by-Step First Setup

## Server Preparation

Spin up a server with a public IP and two DNS A records both pointing to it. In this repo I use the following:
- A -> hs.example.com
- A -> plane.example.com

SSH into the server, create a sudoer user that ansible will use to log in and deploy the stack. **SSH root access will be disabled!**
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

Add your local machine's public key to the /home/headscaler/.ssh/authorized_keys file.

## Local Machine

Clone this repo on your local machine.
```
git clone https://github.com/nicoloboatto/headscaler.git
```

**Edit variables (URLs, passwords) in config.yml.** and edit inventory to contain your headscale server URL (inventory.ini)
```
[headscale_stack]
vpn-01 ansible_host=<Headscale Server URL>
```

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

At the end of the deployment, the following information will be printed. Note it down for later.
```
┌──────────────────────────────────────────────────────────────────
│  Headscale URL    : https://<YOUR_HEADSCALE_URL>
│  Headplane URL    : https://<YOUR_HEADPLANE_URL>
│  Plane login      : <HEADPLANE_USER> / <HEADPLANE_PASSWORD>
│  HP cookie secret : <HP_COOKIE_SECRET>
│  HS API Key       : <HS_API_KEY>                                               
└──────────────────────────────────────────────────────────────────
```

Verify

You should now be able to use your tailscale client to log in through your headscale server.
```
tailscale login --login-server https://<YOUR_HEADSCALE_URL>
```

The headplane web interface should be available at the URL you configured. You will be prompted for you HTTP Basic Auth credentials and for the headscale API key. Enter the ones shown at the end of the deployment process.
```
https://<YOUR_HEADPLANE_URL>
```

# Update

Update Ansible collections:
```
ansible-galaxy collection install -U devsec.hardening community.docker
ansible-galaxy role install -U geerlingguy.docker
```

Pull the latest version of the playbook, or update the version variables manually, then re-run the playbook.
Your database data and any custom DNS records you configured will be preserved.

# DNS

Records can be set from the headplane UI or from the `headscale/extra-records.json` file located on your server after deployment.
