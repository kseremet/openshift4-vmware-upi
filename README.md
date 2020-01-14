# Automating OpenShift 4.x UPI installation 

This repository contains the playbooks and other required Ansible content to automate OpenShift 4.x UPI-based installation on VMware vSphere 6.x platfrom.
>Official UPI on VMware documentation can be found on https://docs.openshift.com/container-platform/latest/installing/installing_vsphere/installing-vsphere.html

To deploy OpenShift 4.x, one needs to configure DHCP, create required DNS records, and provision the required load balancers, an HTTP server and virtual machines before starting the actual installation. For details please refer to official documentation.

The automation playbooks first creates a network utility vm which is configured with the all required roles (i.e. DHCP, DNS, HTTP and LB) automatically. For DHCP and DNS roles, dnsmasq software is used. DHCP is **only used during the first boot** after which all OpenShift nodes are assigned the predefined IP addresses. DHCP server is also configured to serve **only to OpenShift nodes** using their mac addresses, so it's safe to have this DHCP server in the network.
For load balancing role, HAProxy is used. If the base domain is **example.com** and cluster name is **openshift**, then the endpoints for **api.openshift.example.com**, **api-int.openshift.example.com** and ***.apps.openshift.example.com** are automatically configured on this HAProxy to load balance api requests to **bootstrap** and **master** nodes, ingress traffic to all **worker** nodes.

Initially, all the OpenShift nodes will use this network utility server as their DNS server and all the API and Ingress traffic will flow through it. After configuring the required DNS records on the enterprise DNS servers and provisioning the Load Balancers, one can easily eliminate and power off this network utility server by pointing all the nodes to enterprise DNS servers by updating the /etc/resolv.conf on OpenShift nodes.

# Prerequisites

Nearly all of the manual steps in the UPI method is automated by the playbooks in this repository. The only pre-requisites to start the automation is downloading the required Red Hat CoreOS OVA file and import it into VMware vSphere platform, determine the IP addresses that will be assigned statically to the OpenShift nodes and provide internet connectivity (either directly or through a web proxy) to the OpenShift nodes. There is no need to create any DNS records or configure any enterprise load balancers to initiate the installation.

For the smallest OpenShift cluster, 3 master, and 2 worker nodes are required. In addition to these, a bootstrap node is used to initiate deployment and a network utility server is used to provide basic networking services as described above. Therefore, **7 IP addresses** must be allocated at minimum before starting the installation.he

# Deployment

To start the installation, one must set the VMware vCenter credentials as environment variables and then simply run the following command on an Ansible control node:

    export VMWARE_PASSWORD='vCenter Password'
    export VMWARE_USER='vCenter Username'
    export VMWARE_HOST='vCenter IP/FQDN'
    
    time ansible-playbook site.yaml -e @cluster_vars.yaml

Some defaults are defined in the **default_vars.yaml** file which is included by all the playbooks/plays. However, some of these defaults must be updated for individual clusters, like openshift cluster name or datacenter name, and it's better to consolidate all the updates in a new file and specify this file as Ansible extra variables to override the defaults (i.e. **cluster_vars.yaml** as shown in the example above).

# Configuration Reference

Here you can find the list of variables used and their roles explained.

| Variable Name  |Definition                     |Example                      |
|----------------|-------------------------------|-----------------------------|
|cluster_name    |OpenShift cluster name         |cluster_name: openshift      |
|domain_name     |Base DNS domain.           |domain_name: example.com          |
|datacenter      |vSphere datacenter in which openshift vms will be created | datacenter: TestDC|
|vmware_cluster  |vSphere cluster in which openshift vms will be created | vmware_cluster: TestCLS01|
|datastore      |The default vSphere datastore in which openshift vms will be created | datastore: testDS01|
|vm_network    |VM network/Portgroup name in vCenter in which vms will located         |vm_network: vlan_100      |
|vm_rootdisk_size  |Size of the root disk in GiB of the OpenShift VMs. Min 120G is required |vm_rootdisk_size: 120      |
|network_utility_ip     |Ip address to assign to network utility vm.|network_utility_ip: 192.168.0.10          |
|bootstrap_ip          |Ip address to assign to bootstrap vm. | bootstrap_ip: 192.168.0.11|
|master_ips          |Comma separated ip addresses to assign to OpenShift master vms. | master_ips: 192.168.0.12,192.168.0.13,192.168.0.14|
|worker_ips          |Comma separated ip addresses to assign to OpenShift worker vms. | worker_ips: 192.168.0.15,192.168.0.16|
|pull_secret          |The pull secret that you obtained from the [Pull Secret](https://cloud.redhat.com/openshift/install/pull-secret) page on the Red Hat OpenShift Cluster Manager site. | pull_secret: '{"auths": ...}'|
|authorized_ssh_keys |The public portion of the default SSH key for the core user in Red Hat Enterprise Linux CoreOS | authorized_ssh_keys: 'ssh-ed25519 AAAA...'|
|rhcos_template          |Red Hat Enterprise Linux CoreOS template name in vCenter. OVA file must be downloaded from [RHCOS Download Mirror](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/) and imported into vCenter. | rhcos_template: rhcos-4.2.0-20191015.0|
|openshift_major_version          |OpenShift 4.**x**.y major version to be installed | openshift_major_version: 4.2|
|openshift_minor_version |OpenShift 4.x.**y** minor version to be installed | openshift_minor_version: 12|
|http_proxy          |Http(s) proxy server to use if any. Direct internet connectivity is assumed if not defined |http_proxy: http://192.168.191.20:8888|
|additional_trust_bundle   |Provide the contents of the certificate file that are required for proxying HTTPS connections. You can find the deails [here](https://docs.openshift.com/container-platform/4.2/networking/configuring-a-custom-pki.html) |additional_trust_bundle: -----BEGIN CERTIFICATE-----|
|openshift_minor_version |OpenShift 4.x.**y** minor version to be installed | openshift_minor_version: 12|
|network_utility_netmask          |Netmask to use when assigning static ip addresses to nodes |network_utility_netmask: 255.255.255.0|
|network_utility_gateway |Network gateway ip address to set on all nodes | network_utility_gateway: 192.168.0.1|
|network_utility_dns_server          |DNS server ip address to use on network utility server. |network_utility_dns_server: 192.168.1.2|
|(master\|worker)_vm_memory |Memory in MiB to assign different OpenShift node roles | master_vm_memory: 16384|
|(master\|worker)_vm_cpus |Number of vCPUs to assign different OpenShift node roles | master_vm_cpus: 4|
|(bootstrap\|master\|worker)_vm_name_prefix |VM name prefixes of different OpenShift vm roles. A numeric suffix will be appended | master_vm_name_prefix: "{{ cluster_name }}-master"|
|(bootstrap\|master\|worker)_vm_name_offset |VM name numeric suffix to start naming vms of different OpenShift vm roles.| master_vm_name_offset: "0"|


