# Amazon-Linux-2023-image-with-Openstack-Cloud
This repo is to use Amazon Linux 2023 images for use with Openstack Cloud

There are few ways that I have tried to use the Amazon Linux 2023 image on Openstack cloud :

==================================Using the volume======================================================

1) First we need to download the Amazon Linux 2023 image for use with Kvm using the below link in the format qcow2

https://cdn.amazonlinux.com/al2023/os-images/2023.5.20240903.0/kvm/al2023-kvm-2023.5.20240903.0-kernel-6.1-x86_64.xfs.gpt.qcow2

2) Upload the qcow2 image to the image repository (glance) using the openstack dashboard or CLI

openstack image create --tag al2023 --disk-format qcow2 --container-format bare --file <al2023-kvm-2023.5.20240819.0-kernel-6.1-x86_64.xfs.gpt.qcow2> --private amazon-linux-2023-image --progress 

3) Once the image has been uploaded, create a volume out of this image 

openstack volume create --size 40 --image amazon-linux-2023-image amazon-linux-2023-volume

4) Attached the volume to any running linux vm instance on Openstack cloud using the dashboard GUI

5) Once attached the volume you will see it in the lsblk command then mount the volume and then chroot the mount point to access the file system. 

mkdir /als

mount /dev/vdf1 /als

chroot /als

note: in my case volume was /dev/vdf1 

5) make changes to the /etc/cloud/cloud.cfg as per the standard image like adding the “devops/itops/cloud” user instead of the ec2-user and other config changes like installing packages etc as per your company image standards. 

6) edit the /etc/cloud/cloud.cfg.d/02_amazon-onprem.cfg and add openstack as datasource 

datasource_list: [ OpenStack, NoCloud, AltCloud, ConfigDrive, OVF, VMware, None ]

cloud-init clean

cloud-init init 

6) exit the file system and detach the volume from the existing vm instance 

7) create image out of the volume using dashboard or CLI :

openstack image create --container-format bare --disk-format qcow2 --volume amazon-linux-2023-volume amazon-linux-2023-image-updated


8) You can make this image as public, private or shared as per your needs. 

9) Create a vm instance using this image “”amazon-linux-2023-image-updated” and do a ssh and type the below command to check the output 

[root@amazon-linux-2023-updated home]# cloud-id

openstack


[root@amazon-linux-2023-updated home]# cat /etc/system-release

Amazon Linux release 2023.5.20240819 (Amazon Linux)




2) Using the seed.iso method
   =================================================================

   


1) First we need to download the Amazon Linux 2023 image for use with Kvm using the below link in the format qcow2

https://cdn.amazonlinux.com/al2023/os-images/2023.5.20240903.0/kvm/al2023-kvm-2023.5.20240903.0-kernel-6.1-x86_64.xfs.gpt.qcow2

2) Upload the qcow2 image to the image repository (glance) using the openstack dashboard or CLI

openstack image create --tag al2023 --disk-format qcow2 --container-format bare --file <al2023-kvm-2023.5.20240819.0-kernel-6.1-x86_64.xfs.gpt.qcow2> --private amazon-linux-2023-image --progress 

3) Create a vm with the Amazon Linux 2023 image and it will not allow you to ssh to the machine without any user data
   
4) You need to create seed.iso so that the instance can access user-data and meta-data
5) You need to create files user-data and meta data and enter the details for the cloud config  
#cloud-config
#vim:syntax=yaml
users:
  - default
  - name: devops
    groups: sudo
    sudo: ALL=(ALL) NOPASSWD:ALL
    plain_text_passwd: <try it>
    lock_passwd: false
    ssh_pwauth: True
    ssh-authorized-keys:

6) Create a seed.iso using the below command:

mkisofs -output seed.iso -volid cidata -joliet -rock user-data meta-data

7) Create a seed-iso-volume from the iso image and attach the seed-iso-volume to running vm instance as the volume .

8) Reboot the instance for the seed-iso-volume to take affect.

9) SSH to the instance using the cloud config data from the user-data . Datasource will show” nocloud” and seed=/dev/vdb 

[itops@al2023-ec2 ~]$ cloud-id

nocloud


[itops@al2023-ec2 ~]$ cloud-init status --long

status: done
time: Thu, 05 Sep 2024 08:23:56 +0000
detail:
DataSourceNoCloud [seed=/dev/vdb][dsmode=net]

10) In order to vm to boot without the seed.iso volume you need to create a file under /var/lib/cloud/seed by name of “nocloud” and under this create user-data and meta-data with the above config. 

11) Reboot the vm instance and make changes to the /etc/cloud/cloud.cfg as per the standard image like adding the “devops” user instead of the ec2-user and other config changes like installing packages etc as per your company image standards. 

edit the /etc/cloud/cloud.cfg.d/02_amazon-onprem.cfg and add openstack as datasource 

datasource_list: [ OpenStack, NoCloud, AltCloud, ConfigDrive, OVF, VMware, None ]

cloud-init clean

cloud-init init 

12) Detach the seed.iso volume from the instance and reboot the vm instance and this time it will pick from the /var/lib/cloud/seed/nocloud/user-data

13) ssh to the instance and check the cloud datasource and it will show openstack and :

[itops@al2023-bare ~]$ cloud-id

openstack

14) Now remove /var/lib/cloud/seed/nocloud directory to remove the user-data and meta-data from the instance so that next time it can pick cloud-config data from the cloud.cfg .

Run the commands:

cloud-init clean

cloud-init init 


15) Create a snapshot of the vm and the upload to the image client glance 

NOte: You can make this image as public, private or shared as per your needs. 



3) Using the Qemu-nbd method
   ==========================================================================================

   

1) First we need to download the Amazon Linux 2023 image for use with Kvm using the below link in the format qcow2

wget https://cdn.amazonlinux.com/al2023/os-images/2023.5.20240903.0/kvm/al2023-kvm-2023.5.20240903.0-kernel-6.1-x86_64.xfs.gpt.qcow2

2) Add the module nbd to the qemu

modprobe nbd max_part=8

3) Connect the qcow2 file we just downloaded to the nwtwork block device on the host 

qemu-nbd --connect=/dev/nbd0 al2.qcow2

4) Check the device partition on the disk :

fdisk /dev/nbd0 -l

5) mount the Linux filesystem partition to the directory

mount /dev/nbd0p1 /als/

6) chroot /als to access the file system

7) You can now make changes to the /etc/cloud/cloud.cfg as per the standard image like adding the “devops/itops/cloud” user instead of the ec2-user and other config changes like installing packages etc as per your company image standards. 

8) edit the /etc/cloud/cloud.cfg.d/02_amazon-onprem.cfg and add openstack as datasource 

datasource_list: [ OpenStack, NoCloud, AltCloud, ConfigDrive, OVF, VMware, None ]

cloud-init clean

cloud-init init 

9) exit the file system and detach the volume from the existing vm instance .

qemu-nbd --disconnect /dev/nbd0

10) unmount the directory

umount /als/
