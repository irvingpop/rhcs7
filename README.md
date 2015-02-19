# Redhat Cluster Suite version 7

## Description
A Vagrant-based configuration that brings up two CentOS 7 nodes that share a cluster disk with CLVM.
Based on:  https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/High_Availability_Add-On_Administration/ch-startup-HAAA.html


## Quick start

1. Initial configuration of the cluster machines, backend0 and backend1:
  ```bash
  
  vagrant up
  ```
2. Shut them down, to finish attaching the cluster shared disk:
  ```bash
  
  vagrant halt
  VBoxManage storageattach backend1.opscode.piab --storagectl "SAS" --port 0 --device 0 --nonrotational on --type hdd --medium cluster_shared.vdi --mtype shareable
  vagrant up
  ```
3. Open two Terminal windows/tabs, and 'vagrant ssh' to each machine and become root
  ```bash
  vagrant ssh backend0
  sudo -i
  vagrant ssh backend1
  sudo -i
  
  # OR
  csshX vagrant@33.33.33.21 vagrant@33.33.33.22
  sudo -i
  ```
4. Run the following commands on both nodes
  ```bash
  # install clustering packages
  yum -y install pcs fence-agents-all lvm2-cluster
  
  # set hacluster user password
  echo "hacluster" | passwd hacluster --stdin
  
  # start clustering services
  systemctl start pcsd.service
  systemctl enable pcsd.service
  
  # create the mountpoint directory
  mkdir -p /var/opt/opscode/drbd/data
  ```
5. Run the following commands on the first cluster node only
  ```bash
  # authorize cluster
  pcs cluster auth backend0 backend1 -u hacluster -p hacluster
  
  # setup cluster
  pcs cluster setup --start --name chef-ha backend0 backend1
  
  # enable the cluster and examine status
  pcs cluster enable --all
  pcs cluster status
  
  # enable SCSI fence mode (uses SPC-3)
  pcs stonith create scsi fence_scsi devices=/dev/sdb meta provides=unfencing
  sleep 5
  # this might show stopped
  pcs stonith show
  
  # enable LVM clustering with clvmd
  lvmconf --enable-cluster
  pcs resource create dlm ocf:pacemaker:controld clone on-fail=fence interleave=true ordered=true
  pcs resource create clvmd ocf:heartbeat:clvm clone on-fail=fence interleave=true ordered=true
  pcs constraint order start dlm-clone then clvmd-clone
  pcs constraint colocation add clvmd-clone with dlm-clone
  
  # stop lvmetad
  killall lvmetad
  ```
6. Run the following on the second cluster node:
  ```
  pcs cluster auth backend0 backend1 -u hacluster -p hacluster
  lvmconf --enable-cluster
  
  # stop lvmetad
  killall lvmetad
  ```
7. Back to the first cluster node:
  ```
  
  # examine cluster status and ensure all resources are Started/Online
  pcs status
  
  # create the LVM PV and VG
  pvcreate /dev/sdb
  vgcreate -cy shared_vg /dev/sdb
  lvcreate -l 80%VG -n ha_lv shared_vg
  
  # deactivate the ha_lv, and then reactivate it with an exclusive lock
  lvchange -an shared_vg/ha_lv
  lvchange -aey shared_vg/ha_lv
  
  # run lvs on both nodes to ensure it only says active ("a") on backend0
  [root@backend0 ~]# lvs
    LV    VG        Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
    root  centos    -wi-ao---- 38.48g
    swap  centos    -wi-ao----  1.03g
    ha_lv shared_vg -wi-a-----  3.20g
  [root@backend1 ~]# lvs
    LV    VG        Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
    root  centos    -wi-ao---- 38.48g
    swap  centos    -wi-ao----  1.03g
    ha_lv shared_vg -wi-------  3.20g
  
  # format the volume and mount it
  mkfs.xfs /dev/shared_vg/ha_lv
  mount /dev/shared_vg/ha_lv /var/opt/opscode/drbd/data
  ```
8. Update the initramfs device on all your cluster nodes, so that the CLVM volume is never auto-mounted:
  ```
  determine your root VG using the "vgs" command.
  
  vi /etc/lvm/lvm.conf
  on backend0, set:
    volume_list = [ "centos", "@backend0" ]
  
  on backend1, set:
    volume_list = [ "centos", "@backend1" ]
  
  dracut -H -f /boot/initramfs-$(uname -r).img $(uname -r)
  ```

Now you're done!


## Failing over

### The nice way

1. On the active node
  ```
  [root@backend0 ~]# umount /var/opt/opscode/drbd/data
  [root@backend0 ~]# lvchange -an shared_vg/ha_lv
  
  # ensure the LV isn't active
  [root@backend0 ~]# lvs
    LV    VG        Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
    root  centos    -wi-ao---- 38.48g
    swap  centos    -wi-ao----  1.03g
    ha_lv shared_vg -wi-------  3.20g
  ```
2. on the standby node
  ```bash
  
  [root@backend1 ~]# lvs
    LV    VG        Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
    root  centos    -wi-ao---- 38.48g
    swap  centos    -wi-ao----  1.03g
    ha_lv shared_vg -wi-------  3.20g
  [root@backend1 ~]# lvchange -aey shared_vg/ha_lv
  [root@backend1 ~]# mount /dev/shared_vg/ha_lv /var/opt/opscode/drbd/data
  ```


### Forcing yourself to go standby

TBD

### Forcing the other node to go standby (STONITH)

TBD

## Troubleshooting

TBD
