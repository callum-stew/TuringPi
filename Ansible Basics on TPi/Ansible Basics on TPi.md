# Ansible

This covers how to setup ansible and make a simple update playbook. I will using this on the nodes in my Turing Pi 2 but it can be setup to run on virtual machines. The Ansible website has great [documentation](https://docs.ansible.com/ansible/latest/getting_started/index.html) and the book Ansible for DevOps by Jeff Geerling is a very useful resource here: [GitHub - geerlingguy/ansible-for-devops: Ansible for DevOps examples.](https://github.com/geerlingguy/ansible-for-devops)

## Installing

Ansible [intsallation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) documentation can be found on their website.

I used `pipx` to install Ansible as it uses isolated environments to run applications. `pipx` can be intalled using `pip`.

```shell
pip install --user pipx
pipx ensurepath
```

Ansible can now be installed.

```shell
pipx install --include-deps ansible
ansible --version
```

## Inventory

The inventory file is used to specify the IPs or hostnames of the nodes that will be used. It should be created in the folder that you are planning on storing your playbooks in and is usualy named `hosts.ini` but can be created as a YAML file. They are collected into groups with a name that can be used to select the group in a playbook.

---

__In Turing Pi 2 BMC firmware version 1.1.0 python3 was added. This means Ansible can operate on it.__

---

Multiple groups can be combined with the `:children` suffix.

```ini
[cluster:children]
control_node
worker_nodes
```

Variables can be set with the `:vars` suffix. Here I have set the user that should be used to SSH into the nodes.

```ini
[cluster:vars]
ansible_user=root
```

Further documentation on [inventories](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html) can be found on Ansibles website.

## Ansible Config

The`ansible.cfg` file can be used to configuration for use when you run a playbook. It should be in the same location as `hosts.ini` In my case I am just specifying which file should be used as the inventory.

```ini
[defaults]
inventory = hosts.ini
```

Futher documentation on the [Configuration File](https://docs.ansible.com/ansible/latest/reference_appendices/config.html) can be found on Ansibles website.

## Ad-Hoc commands

Ad-Hoc command allow you to run individual commands on your nodes. Documentation for [Ad-Hoc commands](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html) can be found on ansibles website.

An example is to ping them to check everything is working. `-i` specifies the inventory file and `-m` specifies the module to use.

```shell
ansible -i hosts.ini cluster -m ping
```

It should return something like this. If not the `-vv` or `-vvvv` arguments can be used to get a more verbose error output.

```json
192.168.0.102 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
192.168.0.101 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

This can be useful for sending a command to all nodes for example `reboot`. This uses the shell module and the `-a` argument to pass a shell command to the module. `reboot` causes the device to shutdown leading to Ansible raising an error when the connection is lost. To prevent this `-B 1 -P 0` is used to set it as a background task with no polls. `-b` specifies that Ansible should become to get privilege escalation.

```shell
ansible -i hosts.ini cluster -b -B 1 -P 0 -m shell -a "reboot"
```

## Creating an update playbook

Playbooks run a sequence of commands on the nodes. To show their use I will be making a playbook that updates and upgrades APT packages, reboots if necessary and then removes redundent packages.

There is good documentation on the ansible website for [playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/index.html) and the built in [modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html) that I am using.

Playbooks start with a header that contains the name used in the output, the hosts to use form inventory file and options to __gather_facts__ and __become__ a privaledged user. Facts are not needed in this playbook and it takes time to gather them from nodes so it is disabled. As we want the package upgrade to run as a priviledged user __become__ is set to true.

```yaml
---
- name: Upgrade apt packages on cluster
  hosts: cluster
  gather_facts: false
  become: true
```

You can then start listing tasks under the __tasks:__ key. A task has a name and module. This uses the [APT module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html#ansible-collections-ansible-builtin-apt-module) to `update` the cahce then run a `full-upgrade`.

```yaml
  tasks:
    - name: Update and upgrade apt packages
      ansible.builtin.apt:
        update_cache: true
        upgrade: full
```

The file `/var/run/reboot-required` can be used to signify if a reboot is required. The status of this fill can be retrived with the [Stat module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/stat_module.html#ansible-collections-ansible-builtin-stat-module) and the output registered under the name __reboot_required_file__ so that it can be used by later tasks.

```yaml
    - name: Check if reboot is required
      ansible.builtin.stat:
        path: /var/run/reboot-required
        get_checksum: false
      register: reboot_required_file
```

This uses the [Reboot module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/reboot_module.html#ansible-collections-ansible-builtin-reboot-module) to reboot the node. The __when:__ key specifies a condition that must be met for this task to run. In this case it checks the output of the previous task to see if the file exists.

```yaml
    - name: Reboot if required
      ansible.builtin.reboot:
      when: reboot_required_file.stat.exists == true
```

Finaly `autoremove` can be run with the apt module.

```yaml
    - name: Removed dependencies that are no longer required
      ansible.builtin.apt:
        autoremove: true
```

The finished file `upgrade.yaml` can be stored in the same folder as the config and invetory files and then run with:

```shell
ansible-playbook upgrade.yaml
```

It should return an output similar to this:

```
PLAY [Upgrade apt packages on cluster] *****************************************

TASK [Update and upgrade apt packages] *****************************************
ok: [192.168.0.102]
ok: [192.168.0.101]

TASK [Check if reboot is required] *********************************************
ok: [192.168.0.101]
ok: [192.168.0.102]

TASK [Reboot if required] ******************************************************
skipping: [192.168.0.101]
skipping: [192.168.0.102]

TASK [Removed dependencies that are no longer required] ************************
ok: [192.168.0.101]
ok: [192.168.0.102]

PLAY RECAP *********************************************************************
192.168.0.101              : ok=3    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
192.168.0.102              : ok=3    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0  
```