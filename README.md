devstack: https://github.com/openstack-dev/devstack

up-to-date passthrough info: https://docs.openstack.org/admin-guide/compute-pci-passthrough.html

after installing, run screen -x stack to connect to screen session
     window 1 has stats

seems like there's a bug... https://bugs.launchpad.net/nova/+bug/1496565

make sure GPU driver is not loaded

# configure host (compute node) kernel parameters (required)
ubuntu@sas03:~/devstack$ grep ^GRUB_CMDLINE_LINUX= /etc/default/grub
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt pci=realloc pci=nocrs"
ubuntu@sas03:~/devstack$ sudo vim /etc/default/grub
ubuntu@sas03:~/devstack$ sudo update-grub
ubuntu@sas03:~$ sudo reboot

# clone repo
ubuntu@sas03:~$ git clone https://github.com/openstack-dev/devstack.git
ubuntu@sas03:~$ cd devstack/

# customize
ubuntu@sas03:~/devstack$ cp samples/local.conf .

## get GPU device IDs
ubuntu@sas03:~/devstack$ lspci -n | grep a1
04:00.0 0302: 10de:102d (rev a1)
05:00.0 0302: 10de:102d (rev a1)
86:00.0 0302: 10de:102d (rev a1)
87:00.0 0302: 10de:102d (rev a1)

## configure passthrough GPU devices in nova (and modify floating ip range if it will conflict with local networks)
ubuntu@sas03:~/devstack$ cat <<EOF >> local.conf
FLOATING_RANGE=192.168.111.0/24
[[post-config|\$NOVA_CONF]]
[DEFAULT]
# Add scheduler filter for PCI passthrough devices
scheduler_default_filters = RetryFilter,AvailabilityZoneFilter,RamFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,SameHostFilter,DifferentHostFilter,PciPassthroughFilter
scheduler_available_filters = nova.scheduler.filters.all_filters
# whitelist GPU devices for passthrough on compute node
pci_passthrough_whitelist={\"vendor_id\":\"10de\", \"product_id\":\"102d\"}
# create aliases for GPU devices on head node
pci_alias={\"vendor_id\":\"10de\", \"product_id\":\"102d\", \"name\":\"K80\", \"device_type\":\"type-PCI\"}
EOF

### the devstack install script will remove the double quotes if they are not escaped
### nova.conf does not need to escape double-quotes but does need to be valid json so requires the double quotes

# build
ubuntu@sas03:~$ ./stack.sh
...
This is your host IP address: 10.31.241.154
This is your host IPv6 address: ::1
Horizon is now available at http://10.31.241.154/dashboard
Keystone is serving at http://10.31.241.154/identity/
The default users are: admin and demo
The password: nvidia

## manually edit nova configuration to fix quotes in 'pci_passthrough_whitelist' and 'pci_alias' json
ubuntu@sas03:~$ sudo vim /etc/nova/nova.conf

## connect to screen session and restart nova-api and nova-compute
ubuntu@sas03:~$ screen -x stack
### ctrl-a ' 6 enter ("n-api"), ctrl-c, ctrl-p, enter
### ctrl-a ' 16 enter ("n-cpu"), ctrl-c, ctrl-p, enter
### ctrl-a, d (exit screen)

# set environment variables to connect to OpenStack using 'admin' account
ubuntu@sas03:~/devstack$ source openrc admin

# add GPU device to existing flavor
ubuntu@sas03:~/devstack$ openstack flavor set m1.xlarge --property "pci_passthrough:alias"="K80:1"

# create instance
ubuntu@sas03:~/devstack$ openstack server create --flavor m1.xlarge --image cirros-0.3.5-x86_64-disk --wait test-pci

ubuntu@sas03:~/devstack$ openstack floating ip create public
ubuntu@sas03:~/devstack$ openstack server add floating ip test-pci 172.24.4.6
ubuntu@sas03:~/devstack$ ssh cirros@172.24.4.6
password: cubswin:)

# tear down
ubuntu@sas03:~$ ./unstack.sh
# re-deploy
ubuntu@sas03:~$ ./stack.sh

# NOTE: these are deprecated but it does not work without pci_ prefix, will throw error about 'alias not defined'
Option "pci_passthrough_whitelist" from group "DEFAULT" is deprecated. Use option "passthrough_whitelist" from group "pci".
Option "pci_alias" from group "DEFAULT" is deprecated. Use option "alias" from group "pci".

had to manually add pcipassthrough filter to scheduler in nova.conf, already existed
 may have been because the variable was not correct

lab ideas:
provision image with packer, deploy with terraform, maybe a couple of terraform examples for small cluster
nodes pre-installed with devstack, packer, terraform, ubuntu image to base packer image on, packer and terraform files
need to look into numa node, scheduling, affinity, etc.
