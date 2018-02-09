Network Getting Started: Working with Inventory
===============================================

A fully-featured inventory file can serve as the source of truth for your network. Using an inventory file, a single playbook can maintain hundreds of network devices with a single command. This page shows you how to build an inventory file, step by step.

First, group your devices by OS and/or by function. In this tiny example data center, all leaf and spine devices are running VyOS:

.. code-block:: yaml

[leafs]
leaf01 
leaf02

[spines]
spine01
spine02

[network:children]
leafs
spines

[servers]
server01
server02

[datacenter:children]
leafs
spines
servers


Next, you can set many of the variables you needed in your first Ansible command in the inventory, so you can skip them in the ansible-playbook command. This includes each network device's IP, OS, and SSH user:

.. code-block:: yaml

[leafs]
leaf01 ansible_host=10.16.10.11 ansible_network_os=vyos ansible_user=my_vyos_user
leaf02 ansible_host=10.16.10.12 ansible_network_os=vyos ansible_user=my_vyos_user

[spines]
spine01 ansible_host=10.16.10.13 ansible_network_os=vyos ansible_user=my_vyos_user
spine02 ansible_host=10.16.10.14 ansible_network_os=vyos ansible_user=my_vyos_user

[network:children]
leafs
spines

[servers]
server01 ansible_host=10.16.10.15 ansible_user=my_server_user
server02 ansible_host=10.16.10.16 ansible_user=my_server_user

[datacenter:children]
leafs
spines
servers

When devices in a group share the same variable values, such as OS or SSH user, you can reduce duplication and simplify maintenance by consolidating these into group variables:

.. code-block:: yaml

[leafs]
leaf01 ansible_host=10.16.10.11
leaf02 ansible_host=10.16.10.12

[leafs_vars]
ansible_network_os=vyos
ansible_user=my_vyos_user

[spines]
spine01 ansible_host=10.16.10.13
spine02 ansible_host=10.16.10.14

[spines_vars]
ansible_network_os=vyos
ansible_user=my_vyos_user

[network:children]
leafs
spines

[servers]
server01 ansible_host=10.16.10.15
server02 ansible_host=10.16.10.16

[datacenter:children]
leafs
spines
servers
