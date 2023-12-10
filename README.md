# polaris-local-inventory

Local, development version of your Infrastructure.

The playbook in this repo is intended to be used with Ubuntu Multipass to spin
up whatever Inventory the developer needs while they work on
ansible-collection-polaris.


## Usage

In brief:

 - clone this repo
 - install dependencies
 - add/create your inventory
 - define polaris collection variables
 - run the "site.yaml" playbook.

### Clone this repo

Clone the repo and submodules in one go:
```
git clone --branch develop --recurse-submodules https://github.com/linkorb/polaris-local-inventory.git
```
or, as distinct steps:
```
git clone --branch develop https://github.com/linkorb/polaris-local-inventory.git
git submodule update --init --recursive
```

The polaris submodule tracks the `develop` branch of
linkorb/ansible-collection-polaris.  You can configure a different branch:
```
git config -f .gitmodules submodule.polaris.branch some-other-branch
```

### Install dependencies

```shell
ansible-galaxy install -r requirements.yaml

# required for schema validation in polaris collection
pip3 install jsonschema
```

### Add your Inventory

This repo provides a way to spin-up virtual machines (VMs) for use in development
and testing.  To do this, you create a minimal inventory file containing not much
more than the names or aliases of the hosts, organised into whatever groups you
need.  The playbook will begin by spinning-up the hosts and establishing
connections to them.

For example, you might stipulate a very simple inventory like (`my-development-inventory.yaml`):

```yaml
all:
  children:
    polaris_hosts:
      hosts:
        super-worker-1:
        super-worker-2:
        store-1:
        store-2:
```

For each of the hosts in such an inventory, Ansible will:

- create a Multipass VM with the name given in the inventory
- add a user account named "ansible" with SSH access Public Key
- learn the VM IP address and store it in the in-memory inventory

You can see how this is achieved by reviewing the make-multipass-inventory.yaml
play and templates/cloud-init.yaml.j2


### Define your group variables and secrets

For up to date reference of the variables required by the LinkORB Polaris collection review the
[README.md](https://github.com/linkorb/ansible-collection-polaris#readme) or
consult the [variables schema
definition](https://github.com/linkorb/ansible-collection-polaris/blob/main/variables.schema.yaml).


Certain mandatory variables contain secrets, and you'll want to encrypt those. For that you will need to have
[SOPS: Secrets OPeration](https://github.com/getsops/sops) installed.

Next you will need an encryption key. For development purpose you can generate a local PGP key using GPG ([per
the Ansible manual](https://docs.ansible.com/ansible/latest/collections/community/sops/docsite/guide.html#setting-up-sops)).

```shell
$ gpg --quick-generate-key your_email@example.com
... # most output omitted
pub   ...
      60458B91B38A65D7523AD43D9BCF76AF1BAD0BEE
uid                         your_email@example.com
sub   ...
```

At the root of the repository create a `.sops.yaml` file, that contains the previous public key fingerprint
within the following YAML structure:

```yaml
creation_rules:
  - pgp: '60458B91B38A65D7523AD43D9BCF76AF1BAD0BEE'
```

Once all the above is done you can edit encrypted group_vars files via SOPS.

```shell
$ sops group_vars/polaris_hosts.sops.yaml
```

```yaml
# Quickstart polaris variables definition. Review ansible-collection-polaris for an up to date list
polaris:
    admins_active: []
    admins_removed: []
    docker_users: []
    docker_registry_username: your_github_username
    docker_registry_pat: ghc_classic_personal_token_for_container_registry_access
```

> This file will be loaded and decrypted transparently via SOPS thanks to specific ansible.cfg configuration.



### Run the playbook

Run `ansible-playbook -i my-development-inventory.yaml site.yaml`.

Your inventory will be realised as multipass VMs during the first steps of the
"site" playbook.  The IP addresses of the VMs will be collected when the VMs are
up-and-running.  The remainder of the playbook will run when Ansible can connect
to those IP addresses.


## Development

### Developing a standalone Play

A standalone play should be written with the assumption that the Inventory is
up and running (the hosts are reachable).  During development, when the hosts
are Multipass VMs, you should import the tasks/update-multipass-inventory.yaml
into the beginning of a standalone playbook.  These tasks will learn the IP
address of the multipass VMs so that Ansible can connect to them during
execution of your play.  The site.yaml playbook already does this.


## Files in this repo

- `ansible.cfg`: Ansible config for the development environment.

- `collections/ansible_collections/linkorb/polaris`: The polaris collection as
  a git submodule.

- `example-site.yaml`: An example playbook.

- `site.yaml`: The main playbook.  You will run `ansible-playbook site.yaml`.

- `make-multipass-inventory.yaml`: A top-level playbook that realises the
  Inventory as Multipass VMs.

- `prepare-multipass-inventory.yaml`: A top-level playbook to prepare the fresh
  VMs

- `tasks/`: Tasks imported into top-level playbooks.

- `templates/`: Templates used by the top-level playbooks.
