# NSX-T Security with Ansible - Pt2. NS Groups Basic

## Disclaimer
This article is my opinion and not that of my employer.

# Intro
The first part of this series focused on building out a simple firewall rule based on a set of IP addresses. Now we'll move on to use metadata to create dynamic firewall rules. If you haven't already done so, please check out the first part to get all the context and the how to setup the environment.


# NSX-T Concepts
## Namespace Groups
A Namespace (NS) Group can either be defined to include other static members, such as IP Sets, Logical Switches or other NS Groups, but their real power is their ability to have dynamic membership criteria based on tags. For example you can have a group membership criteria to match logical ports with a tag scope of 'dmz', this will allow you to build firewall rules for all ports with the DMZ tag, such as denying access to internal networks. 


##### NS Group criteria example.

# Example Ruleset Answers
In this example we'll put in some a default rule for all Ubuntu VMs to connect to an external Apt repo on the alternate HTTPS port. The answers should be saved in a file called ###.yml in your configs folder.

## Service
NSX-T contains a number of default service definitions, mapping ports for common services. In this example we want to map port 8443 to a service called HTTPS-Alt. The 3 dashes at the top signify this is a YAML file, are not mandatory for Ansible, but are good practice.

---
services:
- display_name: HTTPS-Alt
  nsservice_element:
    destination_ports:
    - 8443
    l4_protocol: TCP
    resource_type: L4PortSetNSService
    state: present`

## IP Set
In this example let's assume that we have 2 different targets which Ubuntu Servers both need to consume on port 8443. 

nsxt_ip_sets:
- display_name: dummy_canonical_repo
  ip_addresses:
  - '66.0.0.8'
  state: present`
- display_name: dummy_ansible_repo
  ip_addresses:
  - '99.0.0.8'
  state: present

## NS Group
First we need a group that will apply to all Ubuntu VMs, so the NS Group will have dynamic membership criteria selecting all VMs with Ubuntu in the OS Name. If you have another OS running in your cluster, just substitute the OS name for whatever you have. Note NSX-t is not case sensitive when selecting the OS type. 

The second group is simply defining that both IP Sets are it's members.

- display_name: ubuntu_vms
  membership_criteria:
    - resource_type: NSGroupSimpleExpression
      target_type: VirtualMachine
      target_property: os_name
      op: contains
      value: ubuntu
  state: present
- display_name: repos_for_ubuntu
  members:
    - resource_type: NSGroupSimpleExpression
      target_type: VirtualMachine
      target_property: os_name
      value: ubuntu
  state: present

## DFW Rules
The rule then has NSGroups as both the source and destination on the custom port we bound.

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

###
Playbook
    - name: Create service
      nsxt_services:
        display_name: "{{ item.display_name }}"
        nsservice_element: "{{ item.nsservice_element }}"
        state: "{{ item.tags | default('present') }}"
      with_items:
        - "{{ services }}"
    - name: Create a new IP Set
      nsxt_ip_sets:
        display_name: "{{ item.display_name }}"
        ip_addresses: "{{ item.ip_addresses }}"
        state: "{{ item.tags | default('present') }}"
      with_items:
        - "{{ nsxt_ip_sets }}"
    - name: Create a new NS Group
      nsxt_ns_groups:
        display_name: "{{  item.display_name  }}"
        membership_criteria: "{{ item.membership_criteria | default([]) }}"
        state: "{{ item.tags | default('present') }}"
      with_items:
        - "{{ nsxt_ns_groups }}"
    - name: Create DFW Section with Rules
      nsxt_firewall_section_with_rules:
        display_name: "{{ item.display_name }}"
        rules: "{{ item.rules | default([]) }}"
        state: "{{ item.tags | default('present') }}"
        stateful: "{{ item.stateful | default(True) }}"
      with_items:
        - "{{ nsxt_dfw_section_with_rules }}"

# Running the playbook

# Checking the output
Now if you browse the GUI, under inventor, Groups and select the ubuntu_vms groups, you should see that your groups has some members.

<insert pic>

Now click the members tab, change the drop down to 'Virtual Machines' and select the 'Effective Members' radio button. 

Now any VM which starts on any cluster managed by NSX-T will automatically inherit the firewall rules in which the ubuntu_vms NS Group is a member.

# Next Time
The next article in this series will look how Kubernetes passes in metadata to NSX-T via NCP and how you can use this with more advanced use cases of NS Groups to build out firewall policy based on Kubernetes labels, plus how to use NSX-T Context Profiles to enforce the protocols of specific flows.
