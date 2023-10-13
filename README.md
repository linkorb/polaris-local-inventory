# polaris-local-inventory

Local, development version of your Infrastructure.

The playbook in this repo is intended to be used with Ubuntu Multipass to spin
up whatever Inventory the developer needs.


## Usage

In brief: clone this repo, add your inventory and run the "site" playbook.

### Clone this repo

Clone the repo:
```
git clone --branch develop https://github.com/linkorb/polaris-local-inventory.git
```

### Add your Inventory

This repo provides a way to spin-up virtual machines (VMs) for use in development
and testing.  To do this, you create a minimal inventory file containing not much
more than the names or aliases of the hosts, organised into whatever groups you
need.  The playbook will begin by spinning-up the hosts and establishing
connections to them.

For example, you might stipulate a very simple inventory like so:

```
; my-development-inventory
[superduper_megalicious_workers]
super-worker-1
super-worker-2

[storage_nodes]
store-1
store-2
```

For each of the hosts in such an inventory, Ansible will:

- create a Multipass VM with the name given in the inventory
- add a user account named "ansible" with SSH access Public Key
- learn the VM IP address and store it in the in-memory inventory

You can see how this is achieved by reviewing the make-multipass-inventory.yaml
play and templates/cloud-init.yaml.j2

### Run the playbook

Run `ansible-playbook -i my-development-inventory site.yaml`.

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
- `example-site.yaml`: An example playbook.
- `site.yaml`: The main playbook.  You will run `ansible-playbook site.yaml`.
- `make-multipass-inventory.yaml`: A top-level playbook that realises the
  Inventory as Multipass VMs.
- `tasks/`: Tasks imported into top-level playbooks.
- `templates/`: Templates used by the top-level playbooks.
