# molecule-ansible-azure
[![Build Status](https://travis-ci.org/jonashackt/molecule-ansible-azure.svg?branch=master)](https://travis-ci.org/jonashackt/molecule-ansible-azure)
[![versionansible](https://img.shields.io/badge/ansible-2.7.9-brightgreen.svg)](https://docs.ansible.com/ansible/latest/index.html)
[![versionmolecule](https://img.shields.io/badge/molecule-2.20.0-brightgreen.svg)](https://molecule.readthedocs.io/en/latest/)
[![versiontestinfra](https://img.shields.io/badge/testinfra-1.19.0-brightgreen.svg)](https://testinfra.readthedocs.io/en/latest/)
[![versionazurecli](https://img.shields.io/badge/azurecli-2.0.60-brightgreen.svg)](https://aws.amazon.com/cli/)

Example projects showing how to do test-driven development of Ansible roles and running those tests on multiple Cloud providers at the same time

This project build on top of [molecule-ansible-docker-vagrant](https://github.com/jonashackt/molecule-ansible-docker-vagrant), where all the basics on how to do test-driven development of Ansible roles with Molecule is described. Have a look into the blog series so far:

* [Test-driven infrastructure development with Ansible & Molecule](https://blog.codecentric.de/en/2018/12/test-driven-infrastructure-ansible-molecule/)
* [Continuous Infrastructure with Ansible, Molecule & TravisCI](https://blog.codecentric.de/en/2018/12/test-driven-infrastructure-ansible-molecule/)
* [Continuous cloud infrastructure with Ansible, Molecule & TravisCI on AWS](https://blog.codecentric.de/en/2019/01/ansible-molecule-travisci-aws/)

## What about Multicloud?

Developing infrastructure code according to prinicples like test-driven development and continuous integration is really great! But what about pushing this to the next level? As [Molecule](https://molecule.readthedocs.io/en/latest/) is able to handle everything Ansible is albe to access, why not run our test automatically on all major cloud platforms at the same time?

With this, we would not only have a security net for our infrastructure code, but would also be safe regarding a switch of our current cloud or data center provider. Lot's of people talk about the unclear costs of this switch. **** If our infrastructure code would be able to run on every cloud platform possible, we would simply be able to switch to whatever platform we want - and all with just the virtually no expenses.Why not just reduce these to zero?!


## Add Azure to the game

Let's start with AWS by just forking [molecule-ansible-google-cloud](https://github.com/jonashackt/molecule-ansible-google-cloud), since there should be mostly everything needed to use Molecule with a Cloud provider.


First, you'll need a valid [Azure account](https://azure.microsoft.com), which happens to be a Microsoft account (you could also use that to access Office 365 and other things). If everything is fine with your account, you should be able to access the Azure Portal at [https://portal.azure.com/#home](https://portal.azure.com/#home):

![azure-portal](screenshots/azure-portal.png) 


According to the Molecule docs about [the Azure driver](https://molecule.readthedocs.io/en/latest/configuration.html#azure), we then we need to install Azure support for Molecule:

```
pip3 install ansible[azure]
```

Compared to AWS and GCE this time we install pip's Ansible package with Azure support, not Molecule itself.


Now let's initialize a new Molecule scenario calles `azure-ubuntu` inside our Ansible role:

```
cd molecule-ansible-azure/docker

molecule init scenario --driver-name azure --role-name docker --scenario-name azure-ubuntu
```

That should create a new directory `azure-ubuntu` inside the `docker/molecule` folder.  We'll integrate the results into our multi scenario project in a second.

Now let's dig into the generated [molecule.yml](docker/molecule/azure-ubuntu/molecule.yml):

```yaml
scenario:
  name: azure-ubuntu

driver:
  name: azure
platforms:
  - name: azure-ubuntu

provisioner:
  name: ansible
  lint:
    name: ansible-lint
    enabled: false
  playbooks:
    converge: ../playbook.yml

lint:
  name: yamllint
  enabled: false

verifier:
  name: testinfra
  directory: ../tests/
  env:
    # get rid of the DeprecationWarning messages of third-party libs,
    # see https://docs.pytest.org/en/latest/warnings.html#deprecationwarning-and-pendingdeprecationwarning
    PYTHONWARNINGS: "ignore:.*U.*mode is deprecated:DeprecationWarning"
  lint:
    name: flake8
  options:
    # show which tests where executed in test output
    v: 1

```

As we already tuned the `molecule.yml` files for our other scenarios like `aws-ec2-ubuntu`, we know what to change here. `provisioner.playbook.converge` needs to be configured, so the one `playbook.yml` could be found.

Also the `verifier` section has to be enhanced to gain all the described advantages like supressed deprecation warnings and the better test result overview.

As you may noticed, the driver now uses `azure` and the platform is pre-configured with only a concrete `name`. Here we just tune the instance name to `azure-ubuntu`.



### Configure Azure Ressource Manager (RM) image

Compared to the AWS and GCE Molecule drivers, these are only one configuration parameters. But there have to be more, just think about region or image configuration! Therefore we need to dive into the [create.yml](docker/molecule/azure-ubuntu/create.yml):

```yaml

- name: Create
  hosts: localhost
  connection: local
  gather_facts: false
  no_log: "{{ not (lookup('env', 'MOLECULE_DEBUG') | bool or molecule_yml.provisioner.log|default(false) | bool) }}"
  vars:
    resource_group_name: molecule
    location: westus
    ssh_user: molecule
    ssh_port: 22
    virtual_network_name: molecule_vnet
    subnet_name: molecule_subnet
    keypair_path: "{{ lookup('env', 'MOLECULE_EPHEMERAL_DIRECTORY') }}/ssh_key"
  tasks:
  ...
  - name: Create molecule instance(s)
        azure_rm_virtualmachine:
          resource_group: "{{ resource_group_name }}"
          name: "{{ item.name }}"
          vm_size: Standard_A0
          admin_username: "{{ ssh_user }}"
          public_ip_allocation_method: Dynamic
          ssh_password_enabled: false
          ssh_public_keys:
            - path: "/home/{{ ssh_user }}/.ssh/authorized_keys"
              key_data: "{{ keypair.ssh_public_key }}"
          image:
            offer: CentOS
            publisher: OpenLogic
            sku: '7.4'
            version: latest
        register: server
        with_items: "{{ molecule_yml.platforms }}"
        async: 7200
        poll: 0
  ...
```

And there we are! Molecule makes heavy usage of Ansible's [azure_rm_virtualmachine](https://docs.ansible.com/ansible/latest/modules/azure_rm_virtualmachine_module.html) module.

Now as we chose to implement the use case of a [standard Ubuntu Docker installation](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1), we need to switch the `image.offer` to another fitting one. So let's adhere to the docs' standard way and use Azure CLI to list available image configurations for our location. 

Therefore Azure CLI needs to be available on your machine. On my Mac I use `brew install azure-cli` to install it. Now we can do:

```
$ az vm image list --output table
You are viewing an offline list of images, use --all to retrieve an up-to-date list
Offer          Publisher               Sku                 Urn                                                             UrnAlias             Version
-------------  ----------------------  ------------------  --------------------------------------------------------------  -------------------  ---------
CentOS         OpenLogic               7.5                 OpenLogic:CentOS:7.5:latest                                     CentOS               latest
CoreOS         CoreOS                  Stable              CoreOS:CoreOS:Stable:latest                                     CoreOS               latest
Debian         credativ                9                   credativ:Debian:9:latest                                        Debian               latest
openSUSE-Leap  SUSE                    42.3                SUSE:openSUSE-Leap:42.3:latest                                  openSUSE-Leap        latest
RHEL           RedHat                  7-RAW               RedHat:RHEL:7-RAW:latest                                        RHEL                 latest
SLES           SUSE                    15                  SUSE:SLES:15:latest                                             SLES                 latest
UbuntuServer   Canonical               18.04-LTS           Canonical:UbuntuServer:18.04-LTS:latest                         UbuntuLTS            latest
WindowsServer  MicrosoftWindowsServer  2019-Datacenter     MicrosoftWindowsServer:WindowsServer:2019-Datacenter:latest     Win2019Datacenter    latest
WindowsServer  MicrosoftWindowsServer  2016-Datacenter     MicrosoftWindowsServer:WindowsServer:2016-Datacenter:latest     Win2016Datacenter    latest
WindowsServer  MicrosoftWindowsServer  2012-R2-Datacenter  MicrosoftWindowsServer:WindowsServer:2012-R2-Datacenter:latest  Win2012R2Datacenter  latest
WindowsServer  MicrosoftWindowsServer  2012-Datacenter     MicrosoftWindowsServer:WindowsServer:2012-Datacenter:latest     Win2012Datacenter    latest
WindowsServer  MicrosoftWindowsServer  2008-R2-SP1         MicrosoftWindowsServer:WindowsServer:2008-R2-SP1:latest         Win2008R2SP1         latest
```

As there we can spot a fitting Ubuntu 18.04 image inside the list, we should be able to configure Molecule. As we already saw inside the generated [create.yml](docker/molecule/azure-ubuntu/create.yml), the `azure_rm_virtualmachine` module uses a `with_items: "{{ molecule_yml.platforms }}"` configuration, so we only need to change the `create.yml` sligthly:

```yaml
  ...
  - name: Create molecule instance(s)
        azure_rm_virtualmachine:
          ...
          image: "{{ item.image }}"
        register: server
        with_items: "{{ molecule_yml.platforms }}"
        async: 7200
        poll: 0
  ...
```

With this, we can now move to our [molecule.yml](docker/molecule/azure-ubuntu/molecule.yml) and configure the Azure image:

```yaml
scenario:
  name: gcp-gce-ubuntu

driver:
  name: azure
platforms:
  - name: azure-ubuntu
    image:
      offer: UbuntuServer
      publisher: Canonical
      sku: 18.04-LTS
      version: latest
  ...
```


### Configure Azure location

We shouldn't forget the configuration of our Azure location (compare with GCP's zone and AWS' region). To find out the correct location, we can leverage the Azure CLI again. As the docs state, there is `az account list-locations` to list the configured location for your account. Running the command will maybe result in the following error:

```
$ az account list-locations
Please run 'az login' to setup account.
```

Now to interact with the `az account` commands, you'll need a valid Azure subscription. You can start with [the free 12 month subscription](https://azure.microsoft.com/en-us/free/) for example. If you entered everything, the subscription should be available inside the Azure Portal and the `azure login` command should work like this:

```
$ az login
Note, we have launched a browser for you to login. For old experience with device code, use "az login --use-device-code"
You have logged in. Now let us find all the subscriptions to which you have access...
[
  {
    "cloudName": "AzureCloud",
    "id": "1f0021b3-xxx-xxxx-xxxx-xxxxxxxxxxx",
    "isDefault": true,
    "name": "Free Trial",
    "state": "Enabled",
    "tenantId": "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx",
    "user": {
      "name": "jonas.hecht@codecentric.de",
      "type": "user"
    }
  }
]
```

Having the Azure CLI successfully logged in, we should try to run `az account list-locations` again. Just pick a location, which suits you best. I took `westeurope` for example and change the [create.yml](docker/molecule/azure-ubuntu/create.yml):

```yaml
- name: Create
  ...
  vars:
    resource_group_name: molecule
    location: westeurope
    ssh_user: molecule
    ssh_port: 22
```



### Creating a Azure RM VM instance with Molecule


Now we should have everything prepared. Let's try to run our first Molecule test on Azure (including `--debug` so that we see what's going on):

```
molecule --debug create --scenario-name azure-ubuntu
```

Open your Google Cloud Compute Engine dashboard and you should see the instance beeing created by Molecule:

![google-cloud-first-running-instance](screenshots/google-cloud-first-running-instance.png)




### Configure Travis CI to run our Molecule test automatically on Azure




