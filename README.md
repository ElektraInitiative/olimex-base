# Configuring the Olimex Base Image using Ansible

This repository contains an [Ansible](https://www.ansible.com/) playbook to showcase how to setup the [base image](http://images.olimex.com/release/a20/) for your [Olimex A20-OLinuXino-LIME2](https://www.olimex.com/Products/OLinuXino/A20/A20-OLinuXino-LIME2/open-source-hardware) .
We also provide a quick introduction to Ansible.

## What is Ansible?

Ansible is an open-source IT automation and configuration management tool that simplifies the management and deployment of software applications and infrastructure.
It follows a declarative approach, allowing you to describe the desired state of your systems rather than writing procedural code. 
We will present an overview of the basic concepts of Ansible.
For a more thorough introduction, take a look at [Ansible's excellent documentation](https://docs.ansible.com/ansible/latest/getting_started/index.html). 

### Inventory

Ansible uses an [inventory](https://docs.ansible.com/ansible/latest/inventory_guide/index.html) file to define the target hosts or groups on which actions will be performed.
It can be a simple text file or a [dynamic inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_dynamic_inventory.html) script that retrieves host information from external sources.

The most basic inventory is actually just a file with a list of IP addresses and [FQDNs](https://en.wikipedia.org/wiki/Fully_qualified_domain_name).
Hosts in inventories can be grouped together, and it is possible to define variables for each host that can then be used within playbooks.

```ini
192.168.0.1 ansible_user=olimex target_static_ip=10.0.2.1

[webservers]
www.example.com ansible_user=web
www2.example.com ansible_user=max
```

### Playbooks

[Playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/index.html) are written in YAML format and contain a set of instructions, known as *tasks*, that Ansible executes on the target hosts.
Playbooks define the desired state of the system and can include variables, conditionals, and loops.

### Tasks

A task is a single unit of work in Ansible.
It represents an action to be performed on the target hosts, such as installing packages, starting services, or configuring files.
Tasks are idempotent, meaning they can be safely run multiple times without causing unintended side effects.

The following example task adds the user `olimex` to the group `i2c`.

```yaml
- name: Add user olimex to i2c group
  user:
    name: olimex
    groups: i2c
    append: true
```

As can bee seen, tasks can be given a human-readable name.
During execution of the playbook, Ansible will print out which tasks it executed, and what the status of the task is, e.g. whether it was successful, it changed something on the host or maybe it failed.

### Handlers

Handlers are similar to tasks, but they are only triggered when notified by other tasks.
They are typically used to restart services or perform specific actions after a change has been made on the target hosts.
Handlers ensure that actions are performed only when necessary.
By default, the notified handlers run after all the tasks in a particular play have been completed. 

### Variables

Ansible allows you to define [variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#playbooks-variables) to customize the behavior of your playbooks and tasks.
Variables can be set globally in inventory files, defined in playbooks, or sourced from external files.
They provide flexibility and allow you to reuse playbooks across different environments.

One very common use-case for variables is in [templates for configuration files](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html).

The following example tasks adds the user specified by the (built-in) variable `ansible_user` to the group `i2c`.

```yaml
- name: Add the Ansible connection user to the i2c group.
  user:
    name: '{{ ansible_user }}'
    groups: i2c
    append: true
```

### Modules

Ansible comes with a wide range of [built-in modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html) that perform specific actions on target hosts.
Modules are used within tasks to carry out operations like managing files, executing commands, manipulating data, or interacting with cloud providers.
Modules can be executed remotely over SSH or leverage connection plugins for different protocols.

[Ansible Galaxy](https://galaxy.ansible.com/) is a hub for sharing and discovering collections of moudles created by the Ansible community.
It provides a centralized repository where you can find pre-written solutions for common tasks, saving you time and effort.

## How to use this Repository

This repository contains playbooks for the most basic tasks when setting up your OLinuXino within the [playbooks](playbooks/) directory.

- [playbook.yaml](playbooks/playbook.yaml):
  The container playbook that includes all the other playbooks.
  
- [bootstrap.yaml](playbooks/bootstrap.yaml): 
  For most tasks, Ansible requires Python to be installed.
  This playbook uses [raw commands](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/raw_module.html) to check for the presence of Python, and if it isn't found it will install a minimal Python environment on the host.
  
- [base-settings.yaml](playbooks/base-settings.yaml):
  In this playbook, the actual setting up of the system happens.
  We set the timme zone, install software (i.e. `chrony`), configure the keyboard, configure users and even let you set a static IP for the host!
  
  If you want to make use of setting a static IP, you need to define the variable `target_static_ip` with the IP you want in the inventory for the host.
  We will also automatically restart the host if we change the IP, with the use of a handler.
  Make sure you change the IP address for the host in the inventory file once the static IP has been applied, otherwise Ansible won't be able to locate it next time 😉.

- [inventory.ini](playbooks/inventory.ini): 
  A sample inventory file.
  Please replace the sample contents with your inventory.

```
$ cd playbooks
$ ansible-playbook -i inventory.ini playbook.yaml -kK
```

The `-k` parameter tells Ansible that it should ask you for the SSH login password for the user specified within the inventory file.
The upper-case `K` tells it to also ask for the root user password.
On the default image, both are `olimex`.

> Note: Ansible will complain that it can not login to the remote host, because by default it does check for the fingerprint of the hosts.
>       Please make sure to connect to each host first via a manual `ssh` invocation first, and when asked by ssh let it save the finger print in the `known_hosts` file.