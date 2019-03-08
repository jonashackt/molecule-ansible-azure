# molecule-ansible-aws-gcp-azure
[![Build Status](https://travis-ci.org/jonashackt/molecule-ansible-aws-gcp-azure.svg?branch=master)](https://travis-ci.org/jonashackt/molecule-ansible-aws-gcp-azure)
[![versionansible](https://img.shields.io/badge/ansible-2.7.8-brightgreen.svg)](https://docs.ansible.com/ansible/latest/index.html)
[![versionmolecule](https://img.shields.io/badge/molecule-2.19.0-brightgreen.svg)](https://molecule.readthedocs.io/en/latest/)
[![versiontestinfra](https://img.shields.io/badge/testinfra-1.16.0-brightgreen.svg)](https://testinfra.readthedocs.io/en/latest/)
[![versionawscli](https://img.shields.io/badge/awscli-1.16.110-brightgreen.svg)](https://aws.amazon.com/cli/)
[![versionazurecli](https://img.shields.io/badge/azurecli-2.0.59-brightgreen.svg)](https://aws.amazon.com/cli/)

Example projects showing how to do test-driven development of Ansible roles and running those tests on multiple Cloud providers at the same time

This project build on top of [molecule-ansible-docker-vagrant](https://github.com/jonashackt/molecule-ansible-docker-vagrant), where all the basics on how to do test-driven development of Ansible roles with Molecule is described. Have a look into the blog series so far:

* [Test-driven infrastructure development with Ansible & Molecule](https://blog.codecentric.de/en/2018/12/test-driven-infrastructure-ansible-molecule/)
* [Continuous Infrastructure with Ansible, Molecule & TravisCI](https://blog.codecentric.de/en/2018/12/test-driven-infrastructure-ansible-molecule/)
* [Continuous cloud infrastructure with Ansible, Molecule & TravisCI on AWS](https://blog.codecentric.de/en/2019/01/ansible-molecule-travisci-aws/)

## What about Multicloud?

Developing infrastructure code according to prinicples like test-driven development and continuous integration is really great! But what about pushing this to the next level? As [Molecule](https://molecule.readthedocs.io/en/latest/) is able to handle everything Ansible is albe to access, why not run our test automatically on all major cloud platforms at the same time?

With this, we would not only have a security net for our infrastructure code, but would also be safe regarding a switch of our current cloud or data center provider. Lot's of people talk about the unclear costs of this switch. **** If our infrastructure code would be able to run on every cloud platform possible, we would simply be able to switch to whatever platform we want - and all with just the virtually no expenses.Why not just reduce these to zero?!


## A selection of cloud providers: Azure, Google, AWS

So let's pick some more providers so that we can safely speak about going 'Multicloud':

* [Molecule's Azure driver](https://molecule.readthedocs.io/en/latest/configuration.html#azure)
* [Molecule's Google Compute Engine (GCE) driver](https://molecule.readthedocs.io/en/latest/configuration.html#gce)
* We already know [how to use Molecule with AWS EC2](https://blog.codecentric.de/en/2019/01/ansible-molecule-travisci-aws/). 


Let's start with AWS by just forking [molecule-ansible-docker-vagrant](https://github.com/jonashackt/molecule-ansible-docker-vagrant), since there should be mostly everything needed to use Molecule with AWS.

This should run in no time :)


## Add Google Cloud Platform to the game

First you'll need a valid [Google Cloud Platform](https://cloud.google.com) account - you should have at least 300$ using a test account for free.

Then we need to install Google Compute Engine (GCE) support for Molecule:

```
pip3 install molecule[gce]
```

Now let's initialize a new Molecule scenario calles `gcp-gce-ubuntu` inside our Ansible role:

```
cd molecule-ansible-aws-gcp-azure/docker

molecule init scenario --driver-name gce --role-name docker --scenario-name gcp-gce-ubuntu
```

That should create a new directory `gcp-gce-ubuntu` inside the `docker/molecule` folder.  We'll integrate the results into our multi scenario project in a second.

Now let's dig into the generated [molecule.yml](docker/molecule/gcp-gce-ubuntu/molecule.yml):

```yaml
s---
 scenario:
   name: gcp-gce-ubuntu
 
 driver:
   name: gce
 platforms:
   - name: gcp-gce-ubuntu
     zone: europe-west3-a
     machine_type: f1-micro
     image: ubuntu-1804-bionic-v20190212a 
 
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

As you may noticed, the driver now uses `gce` and the platform is already pre-configured with a concrete `zone`, `machine_type` and a Google Compute Engine image. Here we just tune the instance name to `gcp-gce-ubuntu` and the `zone` according to our preferred region (see [available regions & zones here](https://cloud.google.com/compute/docs/regions-zones/regions-zones#available)).

Let's also configure a suitable image (see [the Image list here](https://console.cloud.google.com/compute/images)) - for us using our "Install Docker on Ubuntu use case", we should choose `ubuntu-1804-bionic-v20190212a`. The preconfigure [Machine Type](https://cloud.google.com/compute/docs/machine-types?hl=en) `f1-micro` should suffice for us.


### Install needed Python packages: gcloud, apache-libcloud & pycrypto

We need to have `gcloud cli` installed, which is packaged with the Google Cloud SDK. BUT don't install it this way, again use Python package manager pip instead:

```
pip3 install gcloud apache-libcloud pycrypto
```

We also need to install [Apache Libcloud](https://libcloud.apache.org/), so it's already attached to the pip install command. Libcloud is used to interact with Google Compute Engine by Molecule. Also [PyCrypto](https://pypi.org/project/pycrypto/) needs to be installed in order to let Molecule connect to GCP successfully.


### Create a Service Account inside GCE & configure Apache Libcloud

As [described in the docs](https://libcloud.readthedocs.io/en/latest/compute/drivers/gce.html#connecting-to-google-compute-engine) we need to [create a Service account](https://libcloud.readthedocs.io/en/latest/compute/drivers/gce.html#service-account) inside our Google Cloud Console:

> Select the existing or newly created project and go to IAM & Admin -> Service Accounts -> Create service account to create a new service account. 

Provide the service account with a speaking name like `molecule`, then click __NEXT__. Grant the service account the `Owner` role and again click __NEXT__.

Select the role `Owner` and at the tab `Grant users access to this service account (optional)` you should click on __create key__ to create and download new private key you will use to authenticate (I went with the `.json` format). Place the json file into a folder inside your profile:

```
cd ~
mkdir .googlecloud
mv ~/Downloads/yourprojectname-youridhere.json .googlecloud/yourprojectname-youridhere.json
```

Now the Google Cloud credentials json file should reside in `/Users/youruserhere/.googlecloud/yourprojectname-youridhere.json`.

At the end you're service account should be listed inside your projects settings:

![google-cloud-service-account](screenshots/google-cloud-service-account.png)


### Configure GCE credentials for Ansible gce Module

If we have a more detailed look into the [create.yml](docker/molecule/gcp-gce-ubuntu/create.yml) playbook we see, that Molecule use Ansible's [gce Module](https://docs.ansible.com/ansible/latest/modules/gce_module.html) to create Google Compute Engine instances.

And the `create.yml` uses 3 environment variables, that we need to set in order to execute Molecule successfully:`
 
* `GCE_SERVICE_ACCOUNT_EMAIL`: Copy the email address of the created service account.
* `GCE_CREDENTIALS_FILE`: We need to place the path to the credentials file here (like `/Users/youruserhere/.googlecloud/yourprojectname-youridhere.json`)
* `GCE_PROJECT_ID`: Copy the project Id from the project dashboard:

![google-cloud-project-dashboard](screenshots/google-cloud-project-dashboard.png)

Now set all those environment variables locally:

```
export GCE_SERVICE_ACCOUNT_EMAIL=libcloud@yourprojectname-youridhere.iam.gserviceaccount.com
export GCE_CREDENTIALS_FILE=~/.googlecloud/yourprojectname-youridhere.json
export GCE_PROJECT_ID=yourprojectname-youridhere
```


### Creating a Google Compute Engine instance with Molecule


Now we should have everything prepared. Let's try to run our first Molecule test on Google Compute Engine (including `--debug` so that we see what's going on):

```
molecule --debug create --scenario-name gcp-gce-ubuntu
```

Open your Google Cloud Compute Engine dashboard and you should see the instance beeing created by Molecule:

![google-cloud-first-running-instance](screenshots/google-cloud-first-running-instance.png)


### Prepare step fails with no such identity: /Users/yourUserHere/.ssh/google_compute_engine & user@yourIpHere: Permission denied (publickey).

Until here, we didn't need to have the [Google Cloud SDK](https://cloud.google.com/sdk/?hl=en) installed - although I was wondering all the time, when we will need it. And here we are, the [prepare.yml](docker/molecule/gcp-gce-ubuntu/prepare.yml) (and every other) playbook will need the file `/Users/yourUserHere/.ssh/google_compute_engine` to be present to be able to connect to your GCE instances. Otherwise the Molecule execution will fail with something like the following:

```
fatal: [gcp-gce-ubuntu]: UNREACHABLE! => {
        "changed": false,
        "msg": "Failed to connect to the host via ssh: Warning: Permanently added '35.198.116.39' (ECDSA) to the list of known hosts.\r\nno such identity: /Users/yourUserHere/.ssh/google_compute_engine: No such file or directory\r\nyourUserHere@35.198.116.39: Permission denied (publickey).",
        "unreachable": true
    }
```

So now we need to install the Google Cloud SDK:

```
brew cask install google-cloud-sdk
```

If the SDK was successfully installed, we need to give our Google Cloud SDK the needed rights:

```
gcloud auth login
```

This will open your Browser and you'll need to confirm all the occurring questions.

Now configure your project Id in gcloud CLI:

```
gcloud config set project testproject-233213
```

We're now also able to leverage the gcloud CLI for our needs. Let's have a look onto our running instances for example:

```
$ gcloud compute instances list
NAME            ZONE            MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
gcp-gce-ubuntu  europe-west3-a  f1-micro                   10.156.0.5   35.198.116.39  RUNNING
```

Now we're able to generate the necessary `/Users/yourUserHere/.ssh/google_compute_engine` along with `/Users/yourUserHere/.ssh/google_compute_known_hosts`:

```
gcloud compute ssh gcp-gce-ubuntu
```

Finally we should be able to run successfully locally:

```
molecule --debug create --scenario-name gcp-gce-ubuntu
```

[![asciicast](https://asciinema.org/a/231709.svg)](https://asciinema.org/a/231709)



### Configure Travis CI to run our Molecule test automatically on Google Cloud Platform

There are only a few sources on how to do that:

https://cloud.google.com/solutions/continuous-delivery-with-travis-ci

https://stackoverflow.com/questions/38762590/how-to-install-google-cloud-sdk-on-travis


#### Install needed Python packages: gcloud, apache-libcloud & pycrypto

Like as we're already used to locally, we need to have `gcloud cli` installed, which is packaged with the Google Cloud SDK. BUT don't install it this way, again use Python package manager pip instead:

```
pip3 install gcloud apache-libcloud pycrypto
```

We also need to install [Apache Libcloud](https://libcloud.apache.org/), so it's already attached to the pip install command. Libcloud is used to interact with Google Compute Engine by Molecule. Also [PyCrypto](https://pypi.org/project/pycrypto/) needs to be installed in order to let Molecule connect to GCP successfully.


#### Install Google SDK in TravisCI

Add the following to your [.travis.yml](.travis.yml):

```
cache:
  directories:
    - "$HOME/google-cloud-sdk/"

install:
...

# install Google Cloud related packages
- gcloud version || true
- if [ ! -d "$HOME/google-cloud-sdk/bin" ]; then rm -rf $HOME/google-cloud-sdk; export CLOUDSDK_CORE_DISABLE_PROMPTS=1; curl https://sdk.cloud.google.com | bash; fi
# Add gcloud to $PATH
- source /home/travis/google-cloud-sdk/path.bash.inc
- gcloud version
```

Now this should output the `gcloud version` on TravisCI like you're used to locally:

```
$ gcloud version
Google Cloud SDK 236.0.0
bq 2.0.41
core 2019.02.22
gsutil 4.36
```


#### Authenticate gcloud CLI against GCP

As we can't use the interactive mode of `gcloud auth login` on TravisCI, we need to find another option! Luckily the example project https://github.com/GoogleCloudPlatform/continuous-deployment-demo contains the needed steps:

We use encrypted service account credentials to authenticate gcloud CLI non-interactively like this: `gcloud auth activate-service-account --key-file client-secret.json`

So let's do it! First we need to encrypt our service account .json key file. [There's a documentation on how to encrypt files in TravisCI](https://docs.travis-ci.com/user/encrypting-files/) - and here's the fast way:

Install TravisCI CLI locally:

```
brew install travis
```

Then login the CLI to your Travis account:

```
travis login
```

Now copy the service account .json key file `.googlecloud/yourprojectname-youridhere.json` to the project's root (__BUT don't check this into source control!__) and create an ignore entry in the [.gitignore](.gitignore) file:

```
# Google Cloud service account key file for TravisCI
testproject-233213-45d56e1b7fc5.json
```

We can now encrypt our json key file with the `travis encrypt-file` command:

```
$ travis encrypt-file testproject-233213-45d56e1b7fc5.json 
Detected repository as jonashackt/molecule-ansible-aws-gcp-azure, is this correct? |yes| yes
encrypting testproject-233213-45d56e1b7fc5.json for jonashackt/molecule-ansible-aws-gcp-azure
storing result as testproject-233213-45d56e1b7fc5.json.enc
storing secure env variables for decryption

Please add the following to your build script (before_install stage in your .travis.yml, for instance):

    openssl aes-256-cbc -K $encrypted_c0be5bd8086d_key -iv $encrypted_c0be5bd8086d_iv -in testproject-233213-45d56e1b7fc5.json.enc -out testproject-233213-45d56e1b7fc5.json -d

Pro Tip: You can add it automatically by running with --add.

Make sure to add testproject-233213-45d56e1b7fc5.json.enc to the git repository.
Make sure not to add testproject-233213-45d56e1b7fc5.json to the git repository.
Commit all changes to your .travis.yml.
```

As the output already states, we need to add the `openssl aes-256-cbc -K ...` command to our [.travis.yml](.travis.yml) right after the gcloud CLI installation:

```
...
- gcloud version
# Decrypt Google Cloud Platform service account json key file
- openssl aes-256-cbc -K $encrypted_c0be5bd8086d_key -iv $encrypted_c0be5bd8086d_iv -in testproject-233213-45d56e1b7fc5.json.enc -out testproject-233213-45d56e1b7fc5.json -d
...
``` 

Don't forget to check in the __encrypted__ service account json key file `testproject-233213-45d56e1b7fc5.json.enc` into your Git repo.

Now finally add the `gcloud auth activate-service-account` command to your [.travis.yml](.travis.yml):

```
- gcloud auth activate-service-account --key-file testproject-233213-45d56e1b7fc5.json
```


#### Configure the GCP project-ID & ssh config on TravisCI

Now configure your GCE project id inside TravisCI settings as `GCP_GCE_PROJECT_ID` environment variable and configure the project to be used by `gcloud` CLI inside the [.travis.yml](.travis.yml):

```
- gcloud config set project $GCP_GCE_PROJECT_ID
```

Just as we are used to locally, we should now be able to generate the necessary `/Users/yourUserHere/.ssh/google_compute_engine` along with `/Users/yourUserHere/.ssh/google_compute_known_hosts` - but this time on TravisCI! Therefore we need to add the following into our [.travis.yml](.travis.yml):

```
- gcloud compute ssh gcp-gce-ubuntu
```

Now there should be everything in place to finally run our Molecule test on Travis! Therefore add the well known `molecule test --scenario-name gcp-gce-ubuntu` into your Travis config. The [.travis.yml](.travis.yml) should now look something like this:

```yaml
sudo: false
language: python

services:
- docker

cache:
  directories:
    - "$HOME/google-cloud-sdk/"

install:
- pip install molecule
- pip install docker-py

# install Google Cloud related packages
- gcloud version || true
- if [ ! -d "$HOME/google-cloud-sdk/bin" ]; then rm -rf $HOME/google-cloud-sdk; export CLOUDSDK_CORE_DISABLE_PROMPTS=1; curl https://sdk.cloud.google.com | bash; fi
# Add gcloud to $PATH
- source /home/travis/google-cloud-sdk/path.bash.inc
- gcloud version
# Decrypt Google Cloud Platform service account json key file
- openssl aes-256-cbc -K $encrypted_c0be5bd8086d_key -iv $encrypted_c0be5bd8086d_iv -in testproject-233213-45d56e1b7fc5.json.enc -out testproject-233213-45d56e1b7fc5.json -d
# Authenticate against GCP with decrypted key file
- gcloud auth activate-service-account --key-file testproject-233213-45d56e1b7fc5.json
  # Set GCP project id
- gcloud config set project $GCP_GCE_PROJECT_ID
  # Generate GCP ssh files
- gcloud compute ssh gcp-gce-ubuntu

script:
- cd docker
# Molecule Testing Travis-locally with Docker
- molecule test
# Run Molecule test on GCP
- molecule test --scenario-name gcp-gce-ubuntu
```



#### same ssh error --> create needed ssh files

To prevent `ERROR: (gcloud.compute.ssh) Underspecified resource [gcp-gce-ubuntu].`, we need to specify the `--zone` flag.

```
- gcloud compute ssh gcp-gce-ubuntu --zone europe-west3-a
```

We need a non-interactive version of `gcloud compute ssh gcp-gce-ubuntu --zone=europe-west3-a`. And the trick here is really to just use the `--command` parameter and directly exit the prompt in the same command (because we're on Travis :) :

```
gcloud compute ssh gcp-gce-ubuntu --zone=europe-west3-a --command "exit"
```

And as you need a running machine to ssh into it, we need to do this not inside our `.travis.yml`, but our prepare.yml - so it will be executed by Molecule after the machine is up and running.

Add the following to the [prepare.yml](docker/molecule/gcp-gce-ubuntu/prepare.yml):

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Execute gcloud compute ssh to obtain needed files for ssh - and exit in the same command
      shell: gcloud compute ssh gcp-gce-ubuntu --zone=europe-west3-a --command "exit"
      changed_when: false

...

```

#### ERROR: (gcloud.compute.ssh) Underspecified resource [gcp-gce-ubuntu]. Specify the [--zone] flag.

If the above error appears, add the zone to the `gcloud compute ssh` command inside the [.travis.yml](.travis.yml):

```
- gcloud compute ssh gcp-gce-ubuntu --zone europe-west3-a
```



## Add Azure to the party



