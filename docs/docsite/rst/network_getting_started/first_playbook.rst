Getting Started: First Command and Playbook
======================================================

Put the concepts you learned to work with this quick tutorial. Install Ansible, execute a network configuration command manually, execute the same command with Ansible, then create a playbook so you can execute the command any time on multiple network devices. 

Prerequisites
```````````````````````````````````````````````````````````````

Before you work through this tutorial you need:

- Ansible 2.5 (or higher) installed
- One or more network devices that are compatible with Ansible
- Basic Linux command line knowledge
- Basic knowledge of network switch & router configuration

Install Ansible
```````````````````````````````````````````````````````````````

Install Ansible:

.. code-block:: bash

   pip install ansible


Confirm the version of Ansible (must be >= 2.5):

.. code-block:: bash

   ansible --version


For other ways to install Ansible, see :doc:`../intro_installation`.

Establish a Manual Connection to a Managed Node
```````````````````````````````````````````````````````````````

To confirm your credentials, connect to a network device manually and retrieve its configuration. Replace the sample user and device name with your real credentials. For example, for a VyOS router:

.. code-block:: bash

   ssh my_vyos_user@my-vyos-device.com
   show config
   exit

This manual connection also establishes the authenticity of the network device, adding its RSA key fingerprint to your list of known hosts.


Run Your First Network Ansible Command
```````````````````````````````````````````````````````````````

Instead of manually connecting and running a command on the network device, you can retrieve its configuration with a single, stripped-down Ansible command:

.. code-block:: bash

   ansible all -i my-vyos-device.com, -c local -u my_vyos_user -k -m vyos_facts

The flags in this command set:
  - the inventory (-i, the device or devices to target - as demonstrated later, this flag can point to a file)
  - the connection (-c, the method for connecting and executing ansible)
  - the user (-u, the username for the SSH connection)
  - the SSH password (-k, please prompt for the password)
  - the module (-m, the ansible module to run). 


Create and Run Your First Network Ansible Playbook
```````````````````````````````````````````````````````````````

If you want to run this command every day, you can save it in a playbook and run it with ansible-playbook instead of ansible. The playbook can provide a lot of the parameters as well, so you can type less at the command line. 

1. Save this content in a file called first_playbook.yml:

.. code-block:: yaml

  - name: First Playbook
    connection: local
    user: ansible
    hosts: an-vyos-02.ansible.eng.rdu2.redhat.com
    tasks:
      - name: Get config for VyOS devices 
        vyos_facts:
          gather_subset: all
          provider:
            username: ansible
            password: ansible
      - name: Display the config
        debug:
          msg: "The hostname is {{ ansible_net_hostname }} and the OS is {{ ansible_net_version }}"
