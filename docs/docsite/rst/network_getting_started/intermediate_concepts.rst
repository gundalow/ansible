Network Getting Started: Beyond the Basics
======================================================

This page introduces some best practices for managing your Ansible files, along with intermediate concepts like roles, global configuration, privilege escalation, filters and conditional comparisons.

Organizing Playbooks, Inventory and Other Ansible Files
```````````````````````````````````````````````````````````````

Ansible expects to find certain files in certain places. As you expand your inventory and create and run more playbooks, your working Ansible project directory looks like this:

.. code-block:: console

   .
   ├── backup
   │   ├── vyos.example.net_config.2018-02-08@11:10:15
   │   ├── vyos.example.net_config.2018-02-12@08:22:41
   ├── first_playbook.yml
   ├── inventory
   ├── group_vars
   │   ├── vyos.yml
   │   └── eos.yml
   ├── roles
   │   ├── ????
   │   └── ????
   ├── second_playbook.yml
   └── third_playbook.yml

The ``backup`` directory and the files in it get created when you run modules like ``vyos_config`` with the ``backup: yes`` parameter.


Tracking Changes to Inventory and Playbooks with Git
```````````````````````````````````````````````````````````````

start here in the morning . . .

Ansible Configuration: Setting global defaults

Privilege Escalation: `authorize` and `become`

Jinja2: Using Data with Filters and Tests

Conditional Comparison in Network Modules
```````````````````````````````````````````````````````````````

Conditional statements in Ansible evaluate the output from a managed node to determine what happens next in a playbook. Linux/Unix and Windows modules use mathematical symbols (for example, `==`, `<`, and `>`) for comparison. However, network modules use different conditional comparisons. The conditional tests for network modules are:

- eq - Equal
- neq - Not equal
- gt - Greater than
- ge - Greater than or equal
- lt - Less than
- le - Less than or equal
- contains - Object contains specified item

