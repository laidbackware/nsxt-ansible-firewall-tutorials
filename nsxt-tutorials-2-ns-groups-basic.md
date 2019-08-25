---
title: NSX-T Security with Ansible - Pt2. NS Groups
tags: ['nsx-t']
---
## How to create dynamic groups of Objects in NSX-T with Ansible

## Disclaimer
This article is my opinion and not that of my employer.

# Intro
The first part of this series focused on building out a simple firewall rule based on a set of IP addresses. Now we'll move on to use object tags to create dynamic firewall rules. If you haven't already done so, please check out the first part to get all the context and the how to setup the environment.
[Part 1](https://medium.com/swlh/nsx-t-security-with-ansible-pt1-basic-firewall-rules-6aa08c25e226)


# NSX-T Concepts
## Namespace Groups
A Namespace (NS) Group can either be defined to include other objects such as IP Sets, Logical Switches or other NS Groups, but their real power is their ability to define dynamic membership criteria based on tags. For example you can have a group membership criteria to match logical ports with a tag scope of 'dmz', this will allow you to build firewall rules for all ports with the DMZ tag, such as denying access to the internal networks. 

# Example Ruleset Answers
In this example we'll put in a default rule for all Ubuntu VMs to connect to an external Apt repo on the alternate HTTPS port. The answers should be saved in a file called ###.yml in your configs folder.

Continuing the pattern for [Part 1](https://medium.com/swlh/nsx-t-security-with-ansible-pt1-basic-firewall-rules-6aa08c25e226), files will be created in the configs folder which shared the same parent folder as the ansible-for-nsxt modules.

## Service
NSX-T contains a number of default service definitions, which are used to map TCP or UDP ports to names. In this example we want to map port 8443 to a service called HTTPS-Alt. The 3 dashes at the top signify this is a YAML file, are not mandatory for Ansible, but are good practice.

```
---
nsxt_services:
- display_name: HTTPS-Alt
  nsservice_element:
    destination_ports:
    - 8443
    l4_protocol: TCP
    resource_type: L4PortSetNSService
  state: present
```

## IP Sets
In this example let's assume that we have 2 different targets which all Ubuntu Servers will need to connect to.

```
nsxt_ip_sets:
- display_name: dummy_canonical_repo
  ip_addresses:
  - '10.1.2.3'
  state: present
- display_name: dummy_ansible_repo
  ip_addresses:
  - '10.1.2.4'
  state: present
```

## NS Group
The first group will apply to all Ubuntu VMs by giving the NS Group have dynamic membership criteria selecting all VMs with Ubuntu in the OS Name. If you have another OS running in your cluster, just substitute the OS name for whatever you have. Note NSX-T is not case sensitive when selecting the OS type. 

The second group is simply defining that both IP Sets are it's members.

```
nsxt_ns_groups:
- display_name: repos_for_ubuntu
  members:
  - resource_type: NSGroupSimpleExpression
    target_property: id
    op: EQUALS
    target_type: IPSet
    value: "dummy_canonical_repo"
  - resource_type: NSGroupSimpleExpression
    target_property: id
    op: EQUALS
    target_type: IPSet
    value: "dummy_ansible_repo"  
  state: present
- display_name: ubuntu_vms
  membership_criteria:
    - resource_type: NSGroupSimpleExpression
      target_type: VirtualMachine
      target_property: os_name
      op: CONTAINS
      value: ubuntu
  state: present
```

## Distributed Firewall Rules
Finally the firewall rule defintion has Ubuntu VMs NSGroups as source and the repos groups as the destination, on the custom alternative https port.

```
nsxt_firewall_section:
- display_name: Ubuntu default rules
  description: Testing edge1
  stateful: True
  state: present
  rules:
  - display_name: Repo Access
    action: ALLOW
    destinations: 
    - target_display_name: repos_for_ubuntu
      target_type: NSGroup
    ip_protocol: IPV4
    resource_type: FirewallRule
    services: 
    - target_display_name: HTTPS-Alt
      target_type: NSService
    sources: 
    - target_display_name: ubuntu_vms
      target_type: NSGroup
```

## Bringing it all together 
Combining the yaml in the section will result in the following answer file, which should be saved as a file called part2_answers.yml in your config folder.

https://gist.github.com/laidbackware/7ad9bc51b5f6540235dc3425eec01c33

# Playbook
Now we just need a simple playbook to create the firewall section, which involves executing 4 tasks against the answers created in the last section. The playbook should be saved as part2_playbook.yml

https://gist.github.com/laidbackware/3b773b263d746d7c597ebae8012a0982

# Running the playbook
To import the modules from the ansible-for-nsxt folder, 2 environmental variables will needed to point to the library and module_utils folders.
'''
export ANSIBLE_LIBRARY="../ansible-for-nsxt";
export ANSIBLE_MODULE_UTILS="../ansible-for-nsxt/module_utils";
'''
When running the playbook, extra-vars is used to pass in the answers yaml files, with the @ telling Ansible to open a file. Note that manager.yml is needed form part 1.
'''
ansible-playbook part2_playbook.yml --extra-vars=@manager.yml --extra-vars=@part2_answers.yml -vvv;
'''
All goes well an output similar to below should be displayed.

# Checking the output
Now if you browse the GUI, under Inventory > Groups and select the ubuntu_vms groups, you should see that your groups has some members. It's worth noting that the OS type will only get set if the VM is booted with VM Tools installed.

![cat](https://catpics.com/some_cat.png)

Now click the members tab, change the drop down to 'Virtual Machines' and select the 'Effective Members' radio button. 

Any VM which starts on any cluster managed by NSX-T will automatically become a member of this group and inherit any firewall rules in which the ubuntu_vms NS Group is a member.

# Next Time
The next article in this series will focus on how to insert router firewall rules, which are necessary if you need to firewall a load balancer from anything that is not an NSX-T logical port, such as from any source outside of the overlay network.
