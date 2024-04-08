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

The `polaris` submodule tracks the `main` branch of linkorb/ansible-collection-polaris and is mounted in the
***collections/ansible_collections/linkorb/polaris/*** folder. To configure a different branch, run:

```shell
git config -f .gitmodules submodule.polaris.branch some-other-branch
```

The `shipyard.charts` submodule tracks the `main` branch of linkorb/shipyard-charts and is mounted in the ***shipyard/charts/*** folder.

### Install dependencies

```shell
ansible-galaxy install -r requirements.yaml \
&& pip3 install jsonschema
```

### Define your group variables and secrets

1. Create a ***group_vars/polaris_hosts.sops.yaml*** file and fill out the required variables:
   
   ```yaml
   # Quickstart polaris variables definition. Review ansible-collection-polaris for an up to date list
   polaris:
     admins_active: []
     admins_removed: []
     docker_users: []
     docker_registry_username: your_github_username
     docker_registry_pat: ghc_classic_personal_token_for_container_registry_access
     traefik_tls_files:
       - certFile: {{ inventory_dir }}/cert.pem
         keyFile: {{ inventory_dir }}/key.pem
         domain: whoami.local
     docker_secrets_for_shipyard_stacks:
       - name: proxi_jwt_secret_key
         data: someSecretKey
       - name: proxi_jwt_secret_key_passphrase
         data: someSecretKey
       - name: proxi_provider_client_secret
         data: someSecretKey
   ```
   
   > [!NOTE]
   > For an up-to-date reference of the variables required by the LinkORB's Polaris collection, 
   > review the [README.md](https://github.com/linkorb/ansible-collection-polaris#readme) or 
   > consult the [variables schema definition](https://github.com/linkorb/ansible-collection-polaris/blob/main/variables.schema.yaml).

1. Generate a local PGP key for development purposes.

   ```shell
   gpg --quick-generate-key your_email@example.com
   ```

2. Create a ***.sops.yaml*** file at the root of the repository and fill it out, replacing the `pgp` field with your previously generated PGP key's ID as shown below:

   ```yaml
   creation_rules:
     - pgp: 'YOUR_PGP_KEY_ID'
   ```

3. Encrypt the previously created variables and secrets:
   
   ```yaml
   sops --encrypt --in-place group_vars/polaris_hosts.sops.yaml
   ```

   > [!NOTE]
   > You may decrypt the encrypted variables with the below command if you need to edit it:
   >
   > `sops group_vars/polaris_hosts.sops.yaml`

## Generate a TLS key and certificate

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

### Spin up an app

The default configuration of each Polaris-hosted app is stored in the ***shipyard/charts/`app-name`/values.yaml*** file.

To spin up an app using its default configuration, start by adding the app's settings to the `stacks` array of the ***shipyard/shipyard.yaml*** file:
   
```yaml
stacks:
... Other stuff ...
- name: #The app's short name. E.g. cyberchef
  chart: # The name of the chart (shipyard/chart/NAME) containing the app's default values. E.g. cyberchef
  host: swarm-manager
  tag: app
```

To override the default values:

1. Create a ***shipyard/stacks/`the-app's-name`/values.yaml*** file. For example, to spin up [CyberChef](https://github.com/gchq/CyberChef), create a ***shipyard/stacks/cyberchef/values.yaml*** file.
2. Using the app's default configuration (i.e., ***shipyard/charts/\*/values.yaml***) as your reference, populate ***shipyard/stacks/`the-app's-name`/values.yaml*** your preferred settings. 
3. Run the playbook.
   
   ```shell
   ansible-playbook -i inventory.yaml site.yaml
   ```
4. Get the `swarm-manager`'s ip address:
   
   ```shell
   multipass info swarm-manager
   ```
5. In the Multipass host's ***/etc/hosts*** file, map the app's hostname to the `swarm-manager's` IP address. You should be able to access the app by visiting `the-hostname` or `the-hostname:port` if you overrode the default port.

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
  VMs

- [shipyard/charts/](shipyard/charts/): The linkorb/shipyard-charts repo as a submodule.

- [shipyard/shipyard.yaml](shipyard/shipyard.yaml): The helmfile-like manifest of shipyard stacks.

- [shipyard/stacks/](shipyard/stacks/): Instances of shipyard charts, as per the shipyard.yaml manifest.

- [tasks/](tasks/): Tasks imported into top-level playbooks.

- [templates/](templates/): Templates used by the top-level playbooks.
