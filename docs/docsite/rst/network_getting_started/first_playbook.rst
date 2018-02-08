Network Getting Started: First Command and Playbook
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

   ssh my_vyos_user@vyos.example.net
   show config
   exit

This manual connection also establishes the authenticity of the network device, adding its RSA key fingerprint to your list of known hosts. (If you have connected to the device before, you have already established its authenticity.)


Run Your First Network Ansible Command
```````````````````````````````````````````````````````````````

Instead of manually connecting and running a command on the network device, you can retrieve its configuration with a single, stripped-down Ansible command:

.. code-block:: bash

   ansible all -i vyos.example.net, -c local -u my_vyos_user -k -m vyos_facts

The flags in this command set:
  - the inventory (-i, the device or devices to target - as demonstrated later, this flag can point to an inventory file)
  - the connection method (-c, the method for connecting and executing ansible)
  - the user (-u, the username for the SSH connection)
  - the SSH password method (-k, please prompt for the password)
  - the module (-m, the ansible module to run). 


Create and Run Your First Network Ansible Playbook
```````````````````````````````````````````````````````````````

If you want to run this command every day, you can save it in a playbook and run it with ansible-playbook instead of ansible. The playbook can store a lot of the parameters you provided with flags at the command line, leaving less to type at the command line. You need two files for this - a playbook and an inventory file.

1. Save this content in a file called first_playbook.yml:

.. code-block:: yaml

  - name: First Playbook
    connection: local
    hosts: vyos_routers
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

2. Save this content in a file called hosts:

.. code-block:: yaml

   [vyos_routers]
   vyos.example.net


3. Run the playbook with the command:

.. code-block:: bash

   ansible-playbook -i hosts first_playbook.yml

The playbook contains one play with two tasks, and should generate output like this:

.. code-block:: bash

   $ ansible-playbook -i hosts first_playbook.yml 
   
   PLAY [First Playbook]
   ***************************************************************************************************************************
   
   TASK [Gathering Facts]
   ***************************************************************************************************************************
   ok: [vyos.example.net]

   TASK [Get config for VyOS devices]
   ***************************************************************************************************************************
   ok: [vyos.example.net]
   
   TASK [Display some facts]
   ***************************************************************************************************************************
   ok: [vyos.example.net] => {
       "failed": false, 
       "msg": "The hostname is vyos and the OS is VyOS"
   }

4. Now that you can retrieve the device config, try updating it with Ansible. Update your playbook like this:

.. code-block:: yaml

  - name: First Playbook
    connection: local
    hosts: vyos_routers
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
      - name: Update the hostname
	    vyos_config:
	      backup: yes
	      lines:
	        - set system host-name vyos-changed
	      provider:
	        username: ansible
	        password: ansible
      - name: Get changed config for VyOS devices 
        vyos_facts:
          gather_subset: all
          provider:
            username: ansible
            password: ansible
      - name: Display the changed config
        debug:
          msg: "The hostname is {{ ansible_net_hostname }} and the OS is {{ ansible_net_version }}"

This playbook now has four tasks in a single play. The output shows you the change you made to the config:

.. code-block:: bash

   $ ansible-playbook -i hosts first_playbook.yml 

   PLAY [First Playbook]
   ************************************************************************************************************************************
   
   TASK [Gathering Facts]
   ***********************************************************************************************************************************
   ok: [vyos.example.net]
   
   TASK [Get config for VyOS devices]
   **********************************************************************************************************************************
   ok: [vyos.example.net]

   TASK [Display the config]
   *************************************************************************************************************************************
   ok: [vyos.example.net] => {
       "failed": false, 
       "msg": "The hostname is vyos and the OS is VyOS"
   }
   
   TASK [Update the hostname]
   *************************************************************************************************************************************
   changed: [vyos.example.net]

   TASK [Get changed config for VyOS devices]
   *************************************************************************************************************************************
   ok: [vyos.example.net]
   
   TASK [Display the changed config]
   *************************************************************************************************************************************
   ok: [vyos.example.net] => {
       "failed": false, 
       "msg": "The hostname is vyos-changed and the OS is VyOS"
   }
   
   PLAY RECAP
   ************************************************************************************************************************************
   vyos.example.net           : ok=6    changed=1    unreachable=0    failed=0   


Although this playbook is handy, it has a lot of repetitive information. And running a playbook against a single device is not a huge efficiency gain over making the same change manually. The next step to harnessing the full power of Ansible is to expand your inventory, so you can run playbooks against multiple devices.
