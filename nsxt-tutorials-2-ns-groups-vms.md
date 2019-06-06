# NSX-T Security with Ansible - Pt2. NS Groups for VMs

## Disclaimer
This article is my opinion and not that of my employed.

# Intro
The first part of this series focused on building out a simple firewall rule based on a set of IP addresses. Now we'll move on to use metadata to create dynamic firewall rules. If you haven't already done so, please check out the first part to get all the context and the how to setup the environment.


# NSX-T Concepts
## Namespace Groups
A Namespace (NS) Group can either be defined to include other staticly defined members, such as IP Sets or other NS Groups, but their real power is their ability to have dynamic membership criteria based on tags. For example you can have a group membership criteria to match logical ports with a tag scope of 'dmz', this will allow you to build firewall rules for all ports with the DMZ tag, such as denying access to internal networks.


##### NS Group criteria example.

# Example Ruleset Answers
In this example we'll put in some a default rule for all Ubuntu VMs to connect to an external Apt repo on the alternate HTTPS port.

## Service
NSX-T contains a number of default service definitions, mapping ports for common services. In this example we want to map port 8443 to a service called HTTPS-Alt. The 3 dashes at the top signify this is a YAML file, are not mandatory for Ansible, but are good practice.

`---
services:
- display_name: HTTPS-Alt
  nsservice_element:
    destination_ports:
    - 8443
    l4_protocol: TCP
    resource_type: L4PortSetNSService
    state: present`

##NS Group
We want this rule to apply to all Ubuntu VMs, so the NS Group will have dynamic membership criteria selecting all VMs with Ubuntu in the OS Name. Not NSX-t is not case sensitive.

`- display_name: ubuntu_vms
  membership_criteria:
    - resource_type: NSGroupSimpleExpression
      target_type: VirtualMachine
      target_property: os_name
      op: contains
      value: ubuntu
  state: present`

##IP Set
In this example we're making up an IP Set to represent the destination in our firewall rule for an imaginary Apt repo.

`nsxt_ip_sets:
- display_name: dummy_repo
  ip_addresses:
  - '58.0.0.8'
  state: present`


###
Playbook
    - name: Create service
      nsxt_services:
        display_name: "{{ item.display_name }}"
        nsservice_element: "{{ item.nsservice_element }}"
        state: "{{ item.tags | default('present') }}"
      with_items:
        - "{{ services }}"
    - name: Create a new NS Group
      nsxt_ns_groups:
        display_name: "{{  item.display_name  }}"
        membership_criteria: "{{ item.membership_criteria | default([]) }}"
        state: "{{ item.tags | default('present') }}"
      with_items:
        - "{{ nsxt_ns_groups }}"