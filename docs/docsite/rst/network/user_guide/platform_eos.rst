.. _eos_platform_options:

***************************************
EOS Platform Options
***************************************

Arista EOS supports multiple connections. This page offers details on how each connection works in Ansible 2.5 and how to use it. 

.. contents:: Topics

Connections Available
================================================================================

.. csv-table::
   :header: "", "CLI", "eAPI"
   :widths: 10, 10, 10

   "**Protocol:**", "SSH", "HTTP(S)"
   "**Credentials:**", "uses SSH keys / SSH-agent if present", "uses HTTPS certificates if present"
   "**Indirect Access:**", "via a bastion (jump host)", "via a web proxy"
   "**Connection Settings:**", "``ansible_network_connection: network_cli``", "``ansible_network_connection: local`` with ``transport: eapi`` in the ``provider`` dictionary"
   "**Enable Mode (Privilege Escalation):**", "supported - use ``become``", "supported - use ``authorize: yes`` in the ``provider`` dictionary"
   "**Returned Data Format:**", "``stdout[0].``", "``stdout[0].messages[0].``"

Using CLI in Ansible 2.5
================================================================================

Example CLI ``group_vars/eos.yml``

.. code-block:: yaml

   ansible_connection: network_cli
   ansible_network_os: eos
   ansible_user: myuser
   ansible_ssh_pass: !vault...
   ansible_become: yes
   ansible_become_method: enable
   ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -q bastion01"'

If you are using SSH keys (including an ssh-agent) you can remove the ansible_ssh_pass line.

Example CLI Task

.. code-block:: yaml

   - name: Backup switch (eos)
     eos_config:
       backup: yes
     register: backup_eos_location
     when: ansible_network_os == 'eos'



Using eAPI in Ansible 2.5
================================================================================

Example eAPI ``group_vars/eos.yml``

.. code-block:: yaml

   ansible_connection: local
   ansible_network_os: eos
   ansible_user: myuser
   ansible_pass: !vault | 
   eapi:
     host: "{{ inventory_hostname }}"
     transport: eapi

**QUESTION:** How do I set a web proxy for indirect access via an eAPI connection?

Example eAPI Task

.. code-block:: yaml

   - name: Backup switch (eos)
     eos_config:
       backup: yes
       provider: "{{ eapi }}"
     register: backup_eos_location
     when: ansible_network_os == 'eos'

In this example the ``eapi`` variable defined in ``group_vars`` is passed to the ``provider`` option of the module.

.. warning:: 
   Never store passwords in plain text. We recommend using :ref:`Ansible Vault <playbooks_vault>` to encrypt all sensitive variables.


Notes for Ansible < 2.5
================================================================================

Do we need any content here?
