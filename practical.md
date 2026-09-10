vm@VCHOWDARY-MAC AWS % chmod 400 AWS-Login.pem 
vm@VCHOWDARY-MAC AWS % ls -ltr
total 8
-r--------@ 1 vm  staff  1678 Aug 30 14:06 AWS-Login.pem
vm@VCHOWDARY-MAC AWS % 



ssh -i /Users/vm/Documents/AWS/AWS-Login.pem  ec2-user@16.144.28.74


[ec2-user@ip-172-31-37-13 ~]$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0   10G  0 disk 
├─nvme0n1p1 259:1    0    1M  0 part 
├─nvme0n1p2 259:2    0  200M  0 part /boot/efi
└─nvme0n1p3 259:3    0  9.8G  0 part /
nvme1n1     259:4    0   10G  0 disk   ###new disk added
[ec2-user@ip-172-31-37-13 ~]$ 


sudo fdisk /dev/nvme1n1

Inside fdisk, enter:n
Then:p,Enter -->W

lsblk
nvme1n1     259:4    0   10G  0 disk 
└─nvme1n1p1 259:6    0   10G  0 part 

Create XFS filesystem
[ec2-user@ip-172-31-37-13 ~]$ sudo mkfs.xfs /dev/nvme1n1p1

Create mount directory

[ec2-user@ip-172-31-37-13 ~]$ sudo mkdir /data
[ec2-user@ip-172-31-37-13 ~]$ sudo mount /dev/nvme1n1p1 /data
[ec2-user@ip-172-31-37-13 ~]$ df -h


extended EBS from 10 to 15
lsblk
nvme1n1     259:4    0   15G  0 disk 
└─nvme1n1p1 259:6    0   10G  0 part /data
[ec2-user@ip-172-31-37-13 ~]$ 

[ec2-user@ip-172-31-37-13 ~]$ sudo growpart /dev/nvme1n1 1
CHANGED: partition=1 start=2048 old: size=20969472 end=20971519 new: size=31455199 end=31457246
[ec2-user@ip-172-31-37-13 ~]$ 


lsblk
nvme1n1     259:4    0   15G  0 disk 
└─nvme1n1p1 259:6    0   15G  0 part /data

sudo xfs_growfs /data

df -hP --shows extended to 15GB


and you add a new EBS disk:

nvme1n1    20G

you can extend /data using:

sudo pvcreate /dev/nvme1n1
sudo vgextend <VG_NAME> /dev/nvme1n1
sudo lvextend -l +100%FREE /dev/<VG_NAME>/<LV_NAME>
sudo xfs_growfs /data
