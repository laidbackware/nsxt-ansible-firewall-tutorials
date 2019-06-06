# WIP!!!!!

# NSX-T Security with Ansible - Pt3. Kubernetes Integration

## Disclaimer
This article is my opinion and not that of my employed.

# Intro
The previous part of the series covered NS Groups and how you can create a group with matching criteria for virtual machines. This next part will go down to the container level and show how you can build out groups and firewall rule based on labels which are assign pods in Kubernetes. If you haven't already done so, please check out the first part of this series as it cover environment setup.

# Background
This article will talk about creating firewall rules for Kubernetes (K8S), but note K8S is also able to insert firewall rules via the Network Policy API. If you don't have the right tooling and deployment pipelines whereby you application life-cycle is managed via Git and CI/CD, you should avoid managing the NSX firewall directly and only used K8S Network Policy. If you manage NSX firewall rules and K8S deployments separately you will end up in a mess, with stale rules and unexpected behaviour. Whereas is you manage K8S deployments and firewall rules from the same Git repo, with a CI/CD pipeline owning the application life cycle, then how you drive the NSX firewall becomes less relevant.

When we go an see the integration between Kubernetes and NSX-T, we'll see that there is little in the form of truly unique identifiers passed into NSX-T, which means control of how labels are applied should be considered. Especially if you want to make firewall rules be on the labels.

# NSX-T Concepts
## Tags
In NSX-T it is possible to insert tags against most of the objects you can create, such as ports of switches. A tag consists of a scope string and a tag string, either or both. With objects being able to have up to 30 tags. When NSX-T is integrated into a container platform such a Kubernetes, any labels added to deployment spec will automatically appear as tag.

#####tag picture