# rhosp16-manual-deploy


~~~
### 1.sh 

#!/bin/bash
### RHOSP-16.1  

hostnamectl set-hostname undercloud-0.example.local
hostnamectl set-hostname --transient undercloud-0.example.local
echo "127.0.0.1 undercloud-0.example.local undercloud-0" >> /etc/hosts

useradd stack
echo stack | passwd --stdin stack
echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
chmod 0440 /etc/sudoers.d/stack
clear

echo  -e "\033[42;5m Enter Your RHN Username \033[0m"
read username 
echo  -e "\033[42;5m Enter Your RHN Password \033[0m"
read -s password 

subscription-manager register  --username $username  --password $password --force 
subscription-manager list --available --all --matches="Red Hat OpenStack" > sub.txt
pool=$(cat  sub.txt | egrep -i "Employee SKU|pool"| awk {'print $3'} | tail -1)
echo $pool
subscription-manager attach --pool="$pool"
sudo subscription-manager release --set=8.2

echo rhel-8-for-x86_64-baseos-rpms > rhosp16.txt
echo rhel-8-for-x86_64-appstream-rpms >> rhosp16.txt
echo rhel-8-for-x86_64-highavailability-rpms >> rhosp16.txt
echo ansible-2.9-for-rhel-8-x86_64-rpms >> rhosp16.txt
echo satellite-tools-6.5-for-rhel-8-x86_64-rpms >> rhosp16.txt
echo openstack-16.1-for-rhel-8-x86_64-rpms  >> rhosp16.txt
echo fast-datapath-for-rhel-8-x86_64-rpms >> rhosp16.txt
echo advanced-virt-for-rhel-8-x86_64-rpms >> rhosp16.txt
echo rhceph-4-osd-for-rhel-8-x86_64-rpms >> rhosp16.txt 
echo rhceph-4-mon-for-rhel-8-x86_64-rpms >> rhosp16.txt 
echo rhceph-4-tools-for-rhel-8-x86_64-rpms >> rhosp16.txt 


subscription-manager repos --disable=*
for i in $(cat rhosp16.txt) ; do subscription-manager repos --enable=$i ; done

sudo dnf module disable -y container-tools:rhel8
sudo dnf module enable -y container-tools:2.0
sudo dnf module disable -y virt:rhel
sudo dnf module enable -y virt:8.2
sudo dnf update -y
reboot
~~~

### 2.sh

~~~
echo  -e "\033[42;5m Enter Your RHN Username \033[0m"
read username 
echo  -e "\033[42;5m Enter Your RHN Password \033[0m"
read -s password 

sudo yum install -y python3-tripleoclient
sudo dnf install -y ceph-ansible

openstack tripleo container image prepare default \
  --local-push-destination \
  --output-env-file containers-prepare-parameter.yaml


echo "
  ContainerImageRegistryCredentials:
    registry.redhat.io:
      $username: $password " >> containers-prepare-parameter.yaml


echo -e "[DEFAULT]
container_images_file = /home/stack/containers-prepare-parameter.yaml
local_interface = eth1
local_ip = 192.168.24.1/24
undercloud_public_host = 192.168.24.2
undercloud_admin_host = 192.168.24.3
undercloud_ntp_servers=clock.redhat.com
[ctlplane-subnet]
local_subnet = ctlplane-subnet
cidr = 192.168.24.0/24
dhcp_start = 192.168.24.5
dhcp_end = 192.168.24.24
gateway = 192.168.24.1
inspection_iprange = 192.168.24.100,192.168.24.120
masquerade = true
#TODO(skatlapa): add param to override masq" > ~/undercloud.conf


openstack undercloud install
~~~


### 3.sh

~~~
mkdir ~/images
mkdir ~/templates
cd  ~/images

source ~/stackrc
sudo yum install rhosp-director-images rhosp-director-images-ipa -y
for i in /usr/share/rhosp-director-images/ironic-python-agent-16.*.tar /usr/share/rhosp-director-images/overcloud-full-16*.tar ; do tar -xvf $i; done

openstack overcloud image upload --image-path /home/stack/images/
~~~

### 4.sh 
~~~
vi instackenv.json 
~~~

~~~
{
"nodes": [
### controller-0
{
"pm_user": "admin",
"mac": ["52:54:00:33:c1:6e"],
"pm_type": "pxe_ipmitool",
"pm_port": "6230",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:ctrl-0,boot_option:local",
"name": "overcloud-controller-0"
},

### controller-1
{
"pm_user": "admin",
"mac": ["52:54:00:df:8a:f6"],
"pm_type": "pxe_ipmitool",
"pm_port": "6231",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:ctrl-1,boot_option:local",
"name": "overcloud-controller-1"
},

### controller-2
{
"pm_user": "admin",
"mac": ["52:54:00:87:87:ec"],
"pm_type": "pxe_ipmitool",
"pm_port": "6232",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:ctrl-2,boot_option:local",
"name": "overcloud-controller-2"
},

### Compute-0
{
"pm_user": "admin",
"mac": ["52:54:00:81:ae:85"],
"pm_type": "pxe_ipmitool",
"pm_port": "6233",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:cmpt-0,boot_option:local",
"name": "overcloud-compute-0"
},

### Compute-1
{
"pm_user": "admin",
"mac": ["52:54:00:a8:0c:ba"],
"pm_type": "pxe_ipmitool",
"pm_port": "6234",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:cmpt-1,boot_option:local",
"name": "overcloud-compute-1"
},

### Ceph-0
{
"pm_user": "admin",
"mac": ["52:54:00:c2:b2:22"],
"pm_type": "pxe_ipmitool",
"pm_port": "6235",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:ceph-0,boot_option:local",
"name": "ceph-0"
},

### Ceph-1
{
"pm_user": "admin",
"mac": ["52:54:00:32:6a:95"],
"pm_type": "pxe_ipmitool",
"pm_port": "6236",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:ceph-1,boot_option:local",
"name": "ceph-1"
},

### Ceph-2
{
"pm_user": "admin",
"mac": ["52:54:00:a0:9d:71"],
"pm_type": "pxe_ipmitool",
"pm_port": "6237",
"pm_password": "redhat",
"pm_addr": "192.168.24.250",
"capabilities" : "node:ceph-2,boot_option:local",
"name": "ceph-2"
}

]
}
~~~

### 5.sh
~~~
openstack overcloud node import /home/stack/instackenv.json

for i in $(openstack baremetal node list -c UUID -c Name -f value | grep -i ceph | awk {'print $1'}) ; do openstack baremetal node set --property root_device='{"name":"/dev/vda"}'  $i ; done 

openstack overcloud node introspect --all-manageable --provide 
~~~
