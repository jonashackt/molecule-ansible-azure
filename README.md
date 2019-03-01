# molecule-ansible-aws-gcp-azure
[![Build Status](https://travis-ci.org/jonashackt/molecule-ansible-aws-gcp-azure.svg?branch=master)](https://travis-ci.org/jonashackt/molecule-ansible-aws-gcp-azure)
[![versionansible](https://img.shields.io/badge/ansible-2.7.5-brightgreen.svg)](https://docs.ansible.com/ansible/latest/index.html)
[![versionmolecule](https://img.shields.io/badge/molecule-2.19.0-brightgreen.svg)](https://molecule.readthedocs.io/en/latest/)
[![versiontestinfra](https://img.shields.io/badge/testinfra-1.16.0-brightgreen.svg)](https://testinfra.readthedocs.io/en/latest/)
[![versionawscli](https://img.shields.io/badge/awscli-1.16.80-brightgreen.svg)](https://aws.amazon.com/cli/)

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


## Start with AWS

Let's just fork [molecule-ansible-docker-vagrant](https://github.com/jonashackt/molecule-ansible-docker-vagrant), since there should be mostly everything needed to use Molecule with AWS.

