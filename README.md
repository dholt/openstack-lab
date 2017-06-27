# GTC 2017 OpenStack Lab

## Configuring devstack for GPU passthrough

_WARNING: The devstack setup script makes a large number of changes to the system where it's run, you should not run this on a machine you care about_

### Links:
* Devstack: https://github.com/openstack-dev/devstack
* Up-to-date passthrough info: https://docs.openstack.org/admin-guide/compute-pci-passthrough.html

### Notes:
* Provision node with Ubuntu 16.04 LTS
* Make sure GPU driver is not loaded on the host

### Steps:

Add the following line to `/etc/default/grub` to configure the host kernel parameters: 

> GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt pci=realloc pci=nocrs"

```
ubuntu@sas03:~$ sudo vim /etc/default/grub
ubuntu@sas03:~$ sudo update-grub
ubuntu@sas03:~$ sudo reboot
```

Clone the devstack repo

```
ubuntu@sas03:~$ git clone https://github.com/openstack-dev/devstack.git
ubuntu@sas03:~$ cd devstack/
ubuntu@sas03:~$ git checkout stable/ocata
```

Customize

```
ubuntu@sas03:~/devstack$ cp samples/local.conf .
```

Configure floating IP range and nova scheduler to allow PCI passthrough devices

```
ubuntu@sas03:~/devstack$ cat <<EOF >> local.conf
FLOATING_RANGE=192.168.111.0/24
[[post-config|\$NOVA_CONF]]
[DEFAULT]
# Add scheduler filter for PCI passthrough devices
scheduler_default_filters = RetryFilter,AvailabilityZoneFilter,RamFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,SameHostFilter,DifferentHostFilter,PciPassthroughFilter
scheduler_available_filters = nova.scheduler.filters.all_filters
EOF
```

Build

```
ubuntu@sas03:~$ ./stack.sh
...
This is your host IP address: 10.31.241.154
This is your host IPv6 address: ::1
Horizon is now available at http://10.31.241.154/dashboard
Keystone is serving at http://10.31.241.154/identity/
The default users are: admin and demo
The password: nomoresecret
```

Note:
The devstack install script removes double quotes for the pci passthrough variables from local.conf which causes the script to fail when nova fails to launch.

https://bugs.launchpad.net/devstack/+bug/1374118

Until this is figured out, manually edit the nova configuration to add passthrough devices:

Get GPU device IDs

```
ubuntu@sas03:~/devstack$ lspci -n | grep 10de
04:00.0 0302: 10de:102d (rev a1)
05:00.0 0302: 10de:102d (rev a1)
86:00.0 0302: 10de:102d (rev a1)
87:00.0 0302: 10de:102d (rev a1)
```

```
ubuntu@sas03:~$ sudo vim /etc/nova/nova.conf
```

Add the following lines to configure passthrough GPU devices in nova in a new `[pci]` section of `/etc/nova/nova.conf`, substituting in the proper device IDs from `lspci`:

> [pci]

> passthrough_whitelist={"vendor_id":"10de","product_id":"102d"}

> alias={"vendor_id":"10de","product_id":"102d","name":"K80","device_type":"type-PCI"}

After modifying `nova.conf`, connect to the running screen session and restart nova-api and nova-compute:

```
ubuntu@sas03:~$ screen -x stack
```

> ctrl-a ' 6 enter ("n-api"), ctrl-c, ctrl-p, enter

> ctrl-a ' 16 enter ("n-cpu"), ctrl-c, ctrl-p, enter

> ctrl-a d (detach from screen session)

Set environment variables to connect to OpenStack using 'admin' account

```
ubuntu@sas03:~/devstack$ source openrc admin
```

Add GPU device to existing flavor

```
ubuntu@sas03:~/devstack$ openstack flavor set m1.xlarge --property "pci_passthrough:alias"="K80:1"
```

Create instance:

```
ubuntu@sas03:~/devstack$ openstack server create --flavor m1.xlarge --image cirros-0.3.5-x86_64-disk --wait test-pci
ubuntu@sas03:~/devstack$ openstack floating ip create public
ubuntu@sas03:~/devstack$ openstack server add floating ip test-pci $(openstack floating ip list -f value -c 'Floating IP Address')
```

Modify security group for demo project to allow access to new instance

```
ubuntu@sas03:~/devstack$ openstack security group rule create --proto icmp --dst-port 0 $(openstack security group list --project demo -c ID -f value)
ubuntu@sas03:~/devstack$ openstack security group rule create --proto tcp --dst-port 22 $(openstack security group list --project demo -c ID -f value)
```

Connect to instance:

```
ubuntu@sas03:~/devstack$ ssh cirros@$(openstack floating ip list -f value -c 'Floating IP Address')
password: cubswin:)
```

Check for NVIDIA GPU devices:

```
$ dmesg | grep 10de
[    0.261861] pci 0000:00:05.0: [10de:102d] type 0 class 0x000302
```

Delete test instance:

```
ubuntu@sas03:~/devstack$ openstack server delete test-pci
```

## Create new flavor with multiple GPUs:

```
ubuntu@sas03:~/devstack$ openstack flavor create g1.k80x2 --ram 32768 --disk 20 --vcpus 16 --property "pci_passthrough:alias"="K80:2"
```

## Launch a multi-GPU instance, provision NVIDIA drivers, nvidia-docker and run DIGITS:

Create a new image based on Ubuntu 16.04

```
ubuntu@sas03:~/devstack$ wget http://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
ubuntu@sas03:~/devstack$ openstack image create --disk-format qcow2 --container-format bare --public --file xenial-server-cloudimg-amd64-disk1.img ubuntu1604
```

Create an ssh keypair to use with the Ubuntu image:

```
ubuntu@sas03:~/devstack$ openstack keypair create ubuntu | tee ~/.ssh/id_rsa
ubuntu@sas03:~/devstack$ chmod 600 ~/.ssh/id_rsa
```

Launch new instance:

```
ubuntu@sas03:~/devstack$ IP=$(openstack floating ip list -f value -c 'Floating IP Address')
ubuntu@sas03:~/devstack$ openstack server create --flavor g1.k80x2 --image ubuntu1604 --key-name ubuntu --wait ubuntu
ubuntu@sas03:~/devstack$ openstack server add floating ip ubuntu $IP
ubuntu@sas03:~/devstack$ ssh-keygen -f ~/.ssh/known_hosts -R $IP
ubuntu@sas03:~/devstack$ ssh -L0.0.0.0:8080:localhost:8080 ubuntu@$IP
ubuntu@ubuntu:~$ lspci | grep -i nv
00:05.0 3D controller: NVIDIA Corporation GK210GL [Tesla K80] (rev a1)
00:06.0 3D controller: NVIDIA Corporation GK210GL [Tesla K80] (rev a1)
ubuntu@ubuntu:~$ curl -s https://raw.githubusercontent.com/dholt/bootstrap/master/bootstrap.sh | bash -
ubuntu@ubuntu:~$ nvidia-smi -L
GPU 0: Tesla K80 (UUID: GPU-cbc911b4-7c6a-cd5f-3e33-0da557a8717f)
GPU 1: Tesla K80 (UUID: GPU-22cfc3bd-8ef0-bf38-f656-b4c9f0a722a6)
ubuntu@ubuntu:~$ sudo systemctl start nvidia-docker
ubuntu@ubuntu:~$ newgrp docker
ubuntu@ubuntu:~$ nvidia-docker run --name digits -d -p 8080:5000 nvidia/digits
```

Visit the DIGITS web interface at the host IP: http://1.2.3.4:8080/

## Notes:

To re-deploy:

* Tear down: `ubuntu@sas03:~$ ./unstack.sh`
* Re-deploy: `ubuntu@sas03:~$ ./stack.sh`

If the system needs to be rebooted, re-run `stack.sh` to re-deploy.

If you encounter an error regarding an existing volume group when re-running `stack.sh`, remove the volume group with:

```
ubuntu@dgx09:~/devstack$ sudo vgremove stack-volumes-default
  Volume group "stack-volumes-default" successfully removed
```
