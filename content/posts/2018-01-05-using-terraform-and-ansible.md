---
type: post
title: using terraform with ansible
date: '2018-01-05T15:24:58-0600'
author: arges
tags:
- devops
- ansible
- terraform
modified_time: '2018-01-10T09:23:06-0600'

---

A lot of my work lately has been transforming one-off machines into systems
that can be re-generated and re-deployed easily. For much of these we use
Debian packaging and ansible. In many instances the output needs to be cloud
images as well as things that can be imaged on baremetal. For this we use
Packer which can drive ansible. While using ansible to describe how a system is
composed is nice, we also need to describe how those systems will be deployed.
This is where something like terraform comes in.

We wanted to figure out tools that would help accomplish a wide range of goals
without having to lock us into a particular provider. ansible is nice because it
is a client only architecture and works on a wide range of Linux distributions
and platforms. To manage deployment we could also just use ansible, but decided
we liked the ability to better describe services that may change in topology
and numbers without having to manually manage instances. In addition we want to
deploy to many different kinds of platforms like AWS, Azure, and VMWare.

One approach to doing all this would be build images in Packer with ansible,
then deploy with terraform. However we also wanted the ability to iterate a bit
more quickly with instances. This is where using terraform to drive ansible is
nice.

First install terraform from its binary (or build from source):
```
wget https://releases.hashicorp.com/terraform/0.11.1/terraform_0.11.1_linux_amd64.zip
unzip terraform_0.11.1_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

Then create a terraform file:
```
provider "aws" {
  region = "us-east-2"
}

resource "aws_instance" "example" {
  count = 2
  ami = "ami-82f4dae7"
  key_name = "default"
  instance_type = "t2.micro"
  tags {
    Name = "terraform-example-${format("%02d", count.index+1)}"
    Role = "terraform-example"
  }
}

```

This is pretty much a stock example of using terraform, but one thing that did
cause issues was using a static value for `tags.Name`. Without this change
using the terraform-inventory plugin did not work correctly.

To check what will be deployed use:
```
terraform plan
```

If this looks ok use the following to apply it:
```
terraform apply
```

Next we could just manually get the IPs from AWS and put them into an inventory
file, or we could use terraform output to get IPs. But since we're using
ansible it would be better to use dynamic inventory to get the machines we want
to deploy to. A dynamic inventory can be passed to ansible provided it
implements certain flags. Luckily there is already a terraform-ansible dynamic
inventory plugin.

To use it we need to install and set it up:
```
git clone https://github.com/mantl/terraform.py
pip install ./terraform.py
cp terraform.py/scripts/terraform_inventory.sh .
```

We will use `terraform_inventory.sh` in the ansible command.

Install ansible:
```
sudo apt-get install ansible

```

Finally create an ansible file we want to deploy as `provision.yml`:
```
---
- hosts: all
  remote_user: ubuntu
  gather_facts: no
  become: yes
  tasks:
  - name: echo hello
    shell: echo "hello"

```

Now we can run the playbook against the terraform hosts:
```
ansible-playbook --extra-vars="ansible_user=ubuntu" -i terraform_inventory.sh provision.yml
```

Now you'll see it run against all your hosts.

Some refinements here would be we could tag our hosts and use the tag instead
of `all`. In addition we could also dynamically create a key using terraform
and then pass that into ansible.
