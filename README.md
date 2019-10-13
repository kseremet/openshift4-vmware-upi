# openshift4-vmware-upi
Ansible playbooks to automate UPI installation on VMware vSphere

After preparing your install-config.yaml file, you can run:

ansible-playbook upi.yaml -e 'install_config_file=install-config.yaml master_ips=<ip1,ip2,ip3> worker_ips=<ip4,ip5,ip6> bootstrap_ip=<ip7> httpd_ip=<http server ip> cidr_prefix=24 gateway=<gateway ip> dns_server=<dns server ip> rhcos_template=<rhcos 4.x template name>'
