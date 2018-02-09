Network Getting Started: Working with Inventory
===============================================

A single playbook can maintain multiple network devices with a fully-featured inventory file.

Group your devices by OS and/or by function:

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


In addition to grouping your devices, you can add variables like IP, OS, user and more:

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

You can reduce duplication by consolidating common variables into group variables:

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
