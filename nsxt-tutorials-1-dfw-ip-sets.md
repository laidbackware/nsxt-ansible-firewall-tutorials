# NSX-T Security with Ansible - Pt1. Basic Firewall Rules

## Disclaimer
This article is my opinion and not that of my employed.

# Background
NSX-T is VMware's next gen software defined network stack. Whilst there is a GUI, if your using it to drive the firewall you're doing it all wrong. This article will present how to use Ansible to drive the NSX-T firewall using GitOps workflows.

This article assume that you have a basic understanding of Ansible. If not there are many videos and article available via your favorite search engine.

Using these Ansible modules it is possible to integrate NSX-T into Infrastructure as Code pipelines. Whether you want to use this to build VM rules, container based rules or firewall you NSX-T load balancers, you can use these modules within your pipelines to automagically drive the NSX-T Edge and Distributed firewalls based on minimal inputs. We'll be building firewall rules in YAML that can be stored in Git, then because the Ansible Modules are idempotent, we can re-apply the rules at any time and the modules will update the configuration in NSX-T if it detects the defined YAML doesn't match the configured state on NSX-T.

## Sidenotes
NSX-T 2.4 introduced a newer and more declarative API, which simplifies the APIs calls required to drive the firewall, whilst at the same time introduce dependencies which don't suit all use cases. The modules presented below focus on the original API spec, but as Ansible handles all the boiler plate, the interaction patterns are much the same compared to the newer API.

This article will talk about creating firewall rules for Kubernetes (K8S), but note Kubernetes is also able to insert firewall rules via the Network Policy API. If you don't have the right tooling and deployment model pipelines whereby you desired state is tracked via Git, you should avoid managing the NSX firewall directly and only used Kubernetes Network Policy. If you manage NSX firewall rules and Kubernetes deployments separately you will end up in a mess with stale rules and unexpected behaviour. Whereas is you manage K8S deployments and firewall rules from the same Git repo, with CI/CD pipeline owning the application life cycle, then how you drive the NSX firewall becomes less relevant.

# Assumed Knowledge
- Basic networking switching and routing concepts
- Basic firewalling concepts, such ports and protocols
- Virtualization concepts, including virtual networking
- Base Ansible knowledge to prepare and run playbooks
- Reading and writing YAML

# NSX-T Concepts
If you want to dig into the NSX-T architecture beyond firewalling, the VMware Validated Design guides are a good place to start.
https://docs.vmware.com/en/VMware-Validated-Design/5.0.1/vmware-validated-design-501-sddc-nsxt-workload-architecture-design.pdf

## Distributed Firewall (DFW) and Micro Segmentation 
Distrubuted firewalling is one of the best feature of software defined networking IMHO. In legacy networking if you wanted to separate 2 workloads you would need to put them on separate subnets/VLANs and then have a firewall running between the VLANs. This approach does not scale for a number of reasons, primarily because you can have a maximum of 4094 VLANs configured on your network swtiches, plus you have a huge amount of configuration overhead to give you layer 3 separation. In the virtualized and containerized worlds, it is possible to enforce firewall policy at the VM or container level. For example with VMs, the NSX-T software is run within the Hypervisor of the virtual switch that VMs connect to and it is this software within the virtual switch which can enforce firewall policy. By enforcing policy at the point traffic enters an overlay network it is possible to build out zero trust and micro segmented architectures, where every object can only have the absolute minimum of rules, even blocking traffic on objects in the same subnet that run on the same host.

## IP Sets
As the name may suggest IP Sets are sets of IP addresses.  You can define individual IPs, ranges or CIDR style based subnets, up to a maximum of 4000 per IP set.

## Services
With TCP/UDP, as service is a mapping of a name to a set of TCP of UDP ports. For example HTTPS-Alt equals TCP port 8443.

# ansible-for-nsxt
The modules you'll be using in a Git repo which is a fork of the official VMware ansible-for-nsxt repo. All new feature have been submitted back to VMware via a pull request and this tutorial will be updated to reference the VMware repo if the modules are accepted.

## Environment setup
- It is assumed that you already have a working NSX-T environment, ideally on version 2.4 or 2.3. 
- You'll need an Ansible control server, with Ansible 2.7. I've used Ubuntu 16.04.
- (Optional) A good editor with YAML linting. VSCode is quite nice and has good YAML extensions and the Remote SSH extension lets you remotely edit file on your Ansible control host from a non-Linux system.
- (Optional) A git repository to host your firewall rule YAML

On you Ansible Control server run git clone https://github.com/laidbackware/ansible-for-nsxt/

# Example Ruleset Answers
In this example we'll put in some a default rule for everything within the DNS Domain to connect to a DNS server. The YAML for each section needs to be added to a file called answers_demo.yml, which we'll be using later.

All file listed below should be saved in the root of the ansbile-for-nsxt folder where you've cloned the repo.

## IP Set
In this example we're making up an IP Set to represent the destination in our firewall rule for an imaginary Apt repo.

```
nsxt_ip_sets:
- display_name: google_dns
  ip_addresses:
  - '8.8.8.8'
  - '8.8.4.4'
  state: present
```


## Firewall rule section
As we want to enforce firewall policy on virtual machines controlled by NSX-T we want to insert a distributed firewall rule. Luckily this is the default type of rule and we will let the rule apply everywhere. To ensure that only TLS/SSL is used over this rule we're going to add the context aware section.

```
nsxt_firewall_section:
- display_name: Ubuntu default rules
  description: Testing edge1
  stateful: True
  state: present
  rules:
  - display_name: Dummy Repo
    action: ALLOW
    context_profiles:
    - target_display_name: 'SSL'
      target_type: NSProfile
    destinations: 
    - target_display_name: dummy_repo
      target_type: IPSet
    ip_protocol: IPV4
    resource_type: FirewallRule
    services: 
    - target_display_name: HTTPS-Alt
      target_type: NSService
    sources: 
    - target_display_name: ubuntu_vms
      target_type: NSGroup
```

## Manager File
Finally, I normally have separate answerfile file called secrets.yml which can be referenced from all playbooks containing the following fields.

`hostname: "nsxt-manager.lab.local"
username: "admin"
password: "FluffyBunn135"
validate_certs: False`

# Playbook
To apply this change requires a fairly simple Ansible playbook, as all the tasks are independant. By setting the values from our secrets file as vars before we declare tasks it means that we do no have to declare the params withing each task. The text below should be saved as demo_play.yml.

---
- hosts: 127.0.0.1
  connection: local
  become: yes
  vars_files:
    - secrets.yml
    - answers_demo.yml
  vars:
    - hostname: {{ hostname }}
    - username: {{ username }}
    - password: {{ password }}
    - validate_certs: {{ validate_certs }}
  tasks:  
    - name: Create a new IP Set
      nsxt_ip_sets:
        display_name: "{{ item.display_name }}"
        ip_addresses: "{{ item.ip_addresses }}"
        state: "{{ item.tags | default('present') }}"
      with_items:
        - "{{ nsxt_ip_sets }}"
    - name: Create DFW Section with Rules
      nsxt_firewall_section_with_rules:
        display_name: "{{ item.display_name }}"
        rules: "{{ item.rules | default([]) }}"
        state: "{{ item.tags | default('present') }}"
        stateful: "{{ item.stateful | default(True) }}"
      with_items:
        - "{{ nsxt_dfw_section_with_rules }}"

# Running the play



















