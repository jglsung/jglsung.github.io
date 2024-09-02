---
title: 'Lab Environment: Infrastructure as Code with Vagrant, Ansible, and VirtualBox'
#date: YYYY-MM-DD HH:MM:SS -0400 #EDT Daylight Saving 
date: 2024-08-31 07:00:00 -0500 #EST Standard
description: A step-by-step guide on creating a consistent, reproducible lab environment using Vagrant, VirtualBox, and Ansible, focused on automating infrastructure setup through code. 
image:
    path: /assets/img/labenv.png
categories: [homelab,virtualization,automation]
tags: [homelab,virtualization,devops]                     # TAG names should always be lowercase
---

## Introduction

As I started exploring the world of DevOps, I quickly realized the importance of a consistent, reproducible environment. Whether it was developing software, testing configurations, or learning new technologies, the ability to spin up and tear down environments quickly and efficiently is a powerful skill. \
Enter Infrastructure as Code (IaC), the key to mastering environment consistency.

In this lab, I'll walk through setting up a development environment using Vagrant, Ansible, and VirtualBox. This setup will include a control node and multiple worker nodes, all configured through code. \
Let's dive in!

*   *   *

## Prerequisites 

Before I begin, I made sure to have the following installed on my local machine:
1. **Vagrant:** [Download and Install Vagrant](https://developer.hashicorp.com/vagrant/downloads), in my case:

```bash
sudo apt update && sudo apt install vagrant
```

2. **VirtualBox:** [Download and Install VirtualBox](https://www.virtualbox.org/wiki/Downloads), in my case:

```bash
sudo apt update && sudo apt install virtualbox
```

*   *   *

## Vagrant Setup

### Step 1: Setting Up My Project Directory

First, I created a project directory and navigated into it:

```bash
mkdir lab-env/vagrant && cd lab-env/vagrant
```

Inside this `vagrant` directory, I created a `Vagrantfile` to define the configuration of my virtual machines.

*   *   *

### Step 2: Writing My Vagrantfile

The `Vagrantfile` is the heart of this setup. It defines the configuration of my Vms, including the number of machines, their resources, and how they are provisioned. \
Here's an example `Vagrantfile` that I used to set up three VMs:

```ruby
Vagrant.configure("2") do |config|
    servers=[
        {
          :hostname => "control",
          :box => "bento/ubuntu-22.04",
          :ip => "172.16.1.50",
          :ssh_port => 2200
        },
        {
          :hostname => "node1",
          :box => "bento/ubuntu-22.04",
          :ip => "172.16.1.51",
          :ssh_port => 2201
        },
        {
          :hostname => "node2",
          :box => "bento/ubuntu-22.04",
          :ip => "172.16.1.52",
          :ssh_port => 2202
        }
      ]

    servers.each do |machine|
        config.vm.define machine[:hostname] do |node|
            node.vm.box = machine[:box]
            node.vm.hostname = machine[:hostname]
            node.vm.network :private_network, ip: machine[:ip]
            node.vm.network "forwarded_port", guest: 22, host: machine[:ssh_port], id: "ssh"
            node.vm.provider "virtualbox" do |vb|
                vb.gui = false #Disable GUI to save resources
                vb.linked_clone = true
                vb.memory = 1024
                vb.cpus = 2
            end
        end
    end
end
```

**Explanation:**
- `servers` array: I used this to define my control node and two worker nodes
- `box`: This specifies the base image for the VMs. I chose `bento/ubuntu-22.04`
- `private_network`: Assigns a static IP to each VM 
- `forwarded_port`: Forwards the SSH port from each VM to a specific port on my host machine
- `provider` block: Configures VirtualBox settings like memory and CPU allocation

#### Vagrant Architecture Diagram

*Image yet to be inserted*

*   *   *

### Step 3: Running the Environment
> Make sure to run command in the directory where the `Vagrantfile` is located.
{: .prompt-info }

After configuring the `Vagrantfile`, I brought up the environment using:

```bash
vagrant up
```

A long set of logs will show up, we can also see the VMs status using:

```bash
vagrant status
```

#### Result
![vagrant-up](/assets/img/labenv/vagrant-up.png)
_Screenshot of **vagrant status** command_

*   *   *

### Step 4: Accessing the VMs

Once the VMs were up and running, I could SSH into them using:

```bash
vagrant ssh control
vagrant ssh node1
vagrant ssh node2
```

*   *   *

### Step 5: Testing VM inter-communication

Using the `vagrant ssh control` command, I accessed the **control** machine. I decided to test communication between the machines by pinging them. \
To facilitate things, I copied the following `hosts` file to `/etc/hosts`

```
127.0.0.1 localhost
127.0.1.1 vagrant.vm vagrant

# The following lines are desirable for IPv6 capable hosts
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

172.16.1.50 control
172.16.1.51 node1
172.16.1.52 node2
```
By default, Vagrant will share the project directory (the directory with the `Vagrantfile` on host) to `/vagrant` (on guest machine). \
I have copied the above `hosts` file to the synced directory, therefore the *guests* in Vagrant should have it. Inside the **control** machine, I ran the following command:

``` bash
sudo cp /vagrant/hosts /etc/hosts
```
I pinged both **node1** and **node2** using the following:

```bash
ping -c 3 node1 ; ping -c 3 node2
```

The `-c` (count) flag allows me to stop pinging after 3 replies

#### Result
![vagrant-ping](/assets/img/labenv/vagrant-ping.png)
_Screenshot of **ping** command from **control** to **nodes**_

### Step 6: Enabling SSH connectivity

From the **control** machine, we'll ssh into **node1** using the following:


```bash
ssh vagrant@node1
```

#### Result
![vagrant-ssh1](/assets/img/labenv/vagrant-ssh1.png)
_Screenshot of **ssh** command from **control** to **node1**_

The result is the same when connecting to **node2** via **ssh** \
To avoid being prompted for a password and facilitate the use of Ansible, I'll create an ssh key pair.

#### Creating SSH Key Pair
> When prompted for a *passphrase*, press enter to set none.
{: .prompt-info }

From the **control** machine, we'll run:

```bash
ssh-keygen
```

The ssh keys (id_rsa & id_rsa.pub) should be saved at `/home/vagrant/.ssh/`

Now, I can copy the SSH Keys to the nodes by running:

```bash
ssh-copy-id node1 && ssh-copy-id node2
```

I'm prompted for a password, by default it sould be `vagrant` \
Lastly, I'll test the passwordless SSH login

```bash
ssh vagrant@node1
```

#### Result
![vagrant-ssh2](/assets/img/labenv/vagrant-ssh2.png)
_Screenshot of **ssh** command after copying keys to nodes_

Bam! Passwordless login :) \
It's time for Ansible !

*   *   *

## Ansible Setup

### Step 1: Installing Ansible on Control Machine
> Make sure to **ssh** into control machine first.
{: .prompt-info }

Before configuring Ansible, I'll install it on the control machine by running:

```bash
sudo apt install ansible -y
```

I verified the ansible version by running:

```bash
ansible --version
```

Now, I can setup the inventory file and playbook file.

### Step 2: Setting Up Inventory File

On my host machine, inside the `/lab-env/vagrant/ansible/` directory, I'll create a directory for ansible and navigate into it:

```bash
mkdir ansible && cd ansible
```

I then created the following inventory file `labhosts`:

```yaml
control:
  hosts:
    control:
nodes:
  hosts:
    node1:
    node2:
```

This inventory file defines the machines in my lab environment, organized into groups for easy management. The **control** group contains the control machine, while the **nodes** group includes **node1** and **node2**. \
This setup allows me to target specific machines or groups for automation tasks, streamlining the management of my lab environment 

### Step 3: Setting Up Playbook 

On my host machine, inside `/lab-env/vagrant/ansible/`, I'll create the following `playbook,yml`:


