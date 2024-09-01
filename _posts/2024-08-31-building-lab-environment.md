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

### Introduction

As I started exploring the world of DevOps, I quickly realized the importance of a consistent, reproducible environment. Whether it was developing software, testing configurations, or learning new technologies, the ability to spin up and tear down environments quickly and efficiently is a powerful skill.
Enter Infrastructure as Code (IaC), the key to mastering environment consistency.

In this lab, I'll walk through setting up a development environment using Vagrant, Ansible, and VirtualBox. This setup will include a control node and multiple worker nodes, all configured through code. Let's dive in!

*   *   *

### Prerequisites 

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

### Step 1: Setting Up My Project Directory

First, I created a project directory and navigated into it:
```bash
mkdir lab-env/src
cd lab-env/src
```

Inside this `src` directory, I created a `Vagrantfile` to define the configuration of my virtual machines.

*   *   *

### Step 2: Writing My Vagrantfile

The `Vagrantfile` is the heart of this setup. It defines the configuration of my Vms, including the number of machines, their resources, and how they are provisioned. Here's an example `Vagrantfile` that I used to set up three VMs:

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

**Vagrant Architecture Diagram:**

![]()

*   *   *

### Step 3: Running the Environment

After configuring the `Vagrantfile`, I brought up the environment using:
*Note: Make sure to be in the directory where the `Vagrantfile` is located*

```bash
vagrant up
```

**Screenshot:**

![]()

*   *   *

### Step 4: Accessing the VMs

Once the VMs were up and running, I could SSH into them using:

```bash
vagrant ssh control
vagrant ssh node1
vagrant ssh node2
```

From here, I verified that each VM was configured correctly and ready for further automation with Ansible.

*   *   *


