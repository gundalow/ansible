Getting Started: First Command and Playbook
======================================================

Put the concepts you learned to work with this quick tutorial. Install Ansible, execute a network configuration command manually, then execute the same command with Ansible. 

Prerequisites
```````````````````````````````````````````````````````````````

Before you work through these examples you need:
- Ansible 2.5 (or higher) installed
- One or more network devices that are compatible with Ansible
- Basic Linux command line knowledge
- Basic knowledge of network switch & router configuration

Install Ansible
```````````````````````````````````````````````````````````````

Install Ansible:
   `pip install ansible`

Confirm the version of Ansible (must be >= 2.5):
   `ansible --version`

For other ways to install Ansible, see :doc:`intro_installation`.

Establish a Manual Connection to a Managed Node
```````````````````````````````````````````````````````````````

To confirm your credentials, connect to a network device manually and retrieve its configuration. Replace the sample user and device name with your real credentials. For example, for a (? what is an-veos-02.ansible.eng.rdu2.redhat.com?):

```
   ssh veos_user@my-veos-device.com
   enable
   show running-config
```

For a (? what is an-vyos-02.ansible.eng.rdu2.redhat.com?):

```
   ssh vyos_user@my-vyos-device.com
   show config
```

Run Your First Network Ansible Command
```````````````````````````````````````````````````````````````

Now, repeat that same procedure in Ansible with a single command:

   `ansible 
See how your network device is currently configured
(examples for a number of different devices on a number of different platforms)
(demonstrate how to find and use the module documentation)

Create and Run Your First Network Ansible Playbook
```````````````````````````````````````````````````````````````

(start by saving one of the commands used above as a playbook and re-run it with ansible-playbook)
