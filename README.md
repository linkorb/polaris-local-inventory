# polaris-local-inventory

Local, development version of Polaris Infrastructure-as-Code.

The playbook in this repo is intended to be used with Ubuntu Multipass to spin
up a pre-defined Inventory.  For each of the hosts in such an inventory, Ansible
will:

- create a Multipass VM with the name given in the inventory
- add an "ansible" user that has SSH access to the VM
- learn the VMs IP address and store it in the in-memory inventory so that
  Ansible can run tasks on it

You can see how this is achieved by reviewing the below-listed files:

- [make-multipass-inventory.yaml](make-multipass-inventory.yaml)
- [templates/cloud-init.yaml.j2](templates/cloud-init.yaml.j2)

## Prerequisites

- [Multipass](https://multipass.run)
- [Git](https://git-scm.com/)
- python3-pip (apt package)
- [Ansible >= 2.16.3](https://www.ansible.com/)
- [jsonschema (PIP package)](https://pypi.org/project/jsonschema/)
- [SOPS: Secrets OPeration](https://github.com/getsops/sops)
- GitHub [classic Personal Access Token (PAT)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic) with `read:packages` scope.

## Usage

In brief:

 1. Clone this repo.
 2. Install dependencies.
 3. Define polaris collection variables.
 4. Run the "site.yaml" playbook.

### Clone this repo

Clone the repo and submodules in one go:

```shell
git clone --recurse-submodules https://github.com/linkorb/polaris-local-inventory.git
```

or, as distinct steps:

```shell
git clone https://github.com/linkorb/polaris-local-inventory.git
git submodule update --init --recursive
```

The `polaris` submodule tracks the `chartless` branch of linkorb/ansible-collection-polaris and is mounted in the
***collections/ansible_collections/linkorb/polaris/*** folder. To configure a different branch, run:

```shell
git config -f .gitmodules submodule.polaris.branch some-other-branch
```

The `shipyard.charts` submodule has been removed because it is no longer needed.

### Install dependencies

```shell
ansible-galaxy install -r requirements.yaml \
&& pip3 install jsonschema
```

### Define your group variables and secrets

Edit ***group_vars/polaris_hosts/restricted-vars.yaml*** fill out the required variables:

```yaml
# Quickstart polaris variables definition. Review ansible-collection-polaris for an up to date list
polaris_docker_registry_username: your_github_username
polaris_docker_registry_pat: ghc_classic_personal_token_for_container_registry_access
```

> [!NOTE]
> For an up-to-date reference of the variables required by the LinkORB's Polaris collection,
> review the [README.md](https://github.com/linkorb/ansible-collection-polaris#readme) or
> consult the [variables schema definition](https://github.com/linkorb/ansible-collection-polaris/blob/main/variables.schema.yaml).


### Generate a TLS key and certificate

1. For testing purposes, generate a self-signed certificate for the `whoami.local` domain:

   ```shell
   openssl req -x509 -newkey rsa:4096 \
   -keyout key.pem \
   -out cert.pem \
   -sha256 -days 3650 -nodes \
   -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=whoami.local"
   ```

2. To encrypt the TLS certificate and key files in-place using SOPS, run:

   ```shell
   sops --encrypt --in-place cert.pem \
   && sops --encrypt --in-place key.pem
   ```

### Run the playbook

Run `ansible-playbook -i inventory.yaml site.yaml`.

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
execution of your play.  The ***[site.yaml](site.yaml)*** playbook already does this.

## Files in this repo

- [ansible.cfg](ansible.cfg): Ansible configuration for the development environment.

- [inventory.yaml](inventory.yaml): The Inventory of hosts managed by Ansible.

- [collections/ansible_collections/linkorb/polaris](collections/ansible_collections/linkorb/polaris): The polaris collection as
  a git submodule.

- [example-site.yaml](example-site.yaml): An example playbook.

- [site.yaml](site.yaml): The main playbook. You have to run `ansible-playbook site.yaml`.

- [make-multipass-inventory.yaml](make-multipass-inventory.yaml): A top-level playbook that realises the
  Inventory as Multipass VMs.

- [prepare-multipass-inventory.yaml](prepare-multipass-inventory.yaml): A top-level playbook to prepare the fresh
  VMs.

- [stacks/manifest.yaml](stacks/manifest.yaml): The manifest of stacks.

- [stacks/](stacks/): The stacks.

- [tasks/](tasks/): Tasks imported into top-level playbooks.

- [templates/](templates/): Templates used by the top-level playbooks.
