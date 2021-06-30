# HashiStack: Consul + Vault + Nomad Quick Setup Guide by Footloose & Ansible

This is a lightweight, semi-automated setup guide for [Consul](https://www.consul.io/), [Vault](https://www.vaultproject.io/), and [Nomad](https://www.nomadproject.io/) on [Footloose](https://github.com/weaveworks/footloose) powered Docker "container VMs", with [Ansible](https://github.com/ansible/ansible).

It's not meant for production, instead, it's more for building up a sandbox on one VM, or our laptop.

> Note: All these have been built and tested on my MBP, but your miles may vary.

## Prerequisites

These are the prerequisites and versions -- other versions may work too.

### Ansible

```sh
$ ansible --version
ansible [core 2.11.2]
  config file = None
  ...
  python version = 3.9.5 (default, May  4 2021, 03:36:27) [Clang 12.0.0 (clang-1200.0.32.29)]
  jinja version = 3.0.1
  libyaml = True
```

### Footloose

```sh
$ footloose version
version: 0.6.3
```

### Docker

```sh
$ docker version
Client:
 Version:           20.10.7
 ...

Server: Docker Engine - Community
 Engine:
  Version:          20.10.7
  ...
```

## Spin Up "VMs" by Footloose

```sh
# Create a dedicated Docker network
$ docker network create footloose-cluster

# Create the "VMs"
$ footloose create --config footloose.yaml
INFO[0000] Creating SSH key: cluster-key ...
INFO[0002] Docker Image: quay.io/footloose/ubuntu18.04 present locally
INFO[0002] Creating machine: sandbox-ubuntu-0 ...
INFO[0002] Connecting sandbox-ubuntu-0 to the footloose-cluster network...
INFO[0009] Creating machine: sandbox-ubuntu-1 ...
INFO[0009] Connecting sandbox-ubuntu-1 to the footloose-cluster network...
INFO[0015] Creating machine: sandbox-ubuntu-2 ...
INFO[0016] Connecting sandbox-ubuntu-2 to the footloose-cluster network...

# Have a check
$ docker ps
CONTAINER ID   IMAGE                           COMMAND        CREATED          STATUS          PORTS                                                                                                         NAMES
30b3b4509f60   quay.io/footloose/ubuntu18.04   "/sbin/init"   11 seconds ago   Up 5 seconds    0.0.0.0:49247->22/tcp, 0.0.0.0:4648->4646/tcp, :::4648->4646/tcp, 0.0.0.0:8502->8500/tcp, :::8502->8500/tcp   sandbox-ubuntu-2
aff78a966bea   quay.io/footloose/ubuntu18.04   "/sbin/init"   18 seconds ago   Up 12 seconds   0.0.0.0:49206->22/tcp, 0.0.0.0:4647->4646/tcp, :::4647->4646/tcp, 0.0.0.0:8501->8500/tcp, :::8501->8500/tcp   sandbox-ubuntu-1
3738eb18bdb7   quay.io/footloose/ubuntu18.04   "/sbin/init"   25 seconds ago   Up 19 seconds   0.0.0.0:4646->4646/tcp, :::4646->4646/tcp, 0.0.0.0:8500->8500/tcp, :::8500->8500/tcp, 0.0.0.0:49164->22/tcp   sandbox-ubuntu-0
```

> Note: other than the default port `22`, here we also expose extra ports: Consul UI (`8500`), Vault UI (`8200`) and Nomad UI (`4646`)


## Set Up HashiStack Components

Before we do that, let's install the required Ansible roles first:

```sh
# Install Ansible roles
$ ansible-galaxy install --roles-path ~/.ansible/roles -r requirements.yaml

# List to double check
$ ansible-galaxy list
- brianshumate.vault, v2.5.2
- brianshumate.nomad, v1.9.5
- griggheo.consul-template, 1.2.5
- geerlingguy.docker, 3.1.2
- brianshumate.consul, v2.5.4
```

### Consul

```sh
$ ansible-playbook playbooks/hashistack-consul.yaml
```

> Note: Access Consul UI from the Docker host, which in my case is my laptop: http://localhost:8500/


### Vault

```sh
$ ansible-playbook playbooks/hashistack-vault.yaml
```

> Note: Access Vault UI from the Docker host, which in my case is my laptop: http://localhost:8200/

By default the Vault is sealed, do remember to unseal Vault before proceeding to the next.

To unseal Vault, typically there are 3 simple steps:

- Access: http://localhost:8200/
- Key in `Key Shares` as `1` and `Key Threshold` as `1` in our case, click "Initialize"
- Click "Download Keys" to download the keys to local safe place, then click "Continue to unseal"
- Unseal Vault by using the `keys` token
- Now it's ready to sign in, try using `root_token` to sign in

> Note: Do remember that you will have to unseal Vault every time you restart it.


### Nomad

Nomad will be integrated with Vault.

To achieve that, Nomad servers must be provided a Vault token. This token can either be a root token or a periodic token with permissions to create from a token role.

For simplicity purpose only, we're going to use the root token, which is NOT recommended in production.

```sh
$ ansible-vault create playbooks/group_vars/nomad_instances
```

Type a password (and remember it) and then in the promt key in this:

```yaml
---
nomad_vault_token: <Your Vault Root Token Goes to Here>
```

Now let's spin up Nomad and we will be prompted for Vault password this time.
The playbook will install Docker as one of the Nomad drivers and Nomad cluster.

```sh
$ ansible-playbook playbooks/hashistack-nomad.yaml
```

> Note: Access Nomad UI from the Docker host, which in my case is my laptop: http://localhost:4646/


## Test Case



## Clean Up

To clean up, we can simply delete the Footloose container VMs:

```sh
$ footloose delete --config footloose.yaml
```
