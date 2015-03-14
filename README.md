# Redhat Cluster Suite version 7

## Description
A Vagrant-based configuration that brings up two CentOS 7 nodes that share a cluster disk with CLVM.
Based on:  https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/High_Availability_Add-On_Administration/ch-startup-HAAA.html
with clarification from: http://www.davidvossel.com/wiki/index.php?title=HA_LVM

## Note

try again, next time without the clvmd stuff

## Initial setup
1. Initial configuration of the cluster machines, backend0 and backend1:
  ```bash
  
  vagrant up
  ```
1. iSCSI server setup
from: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Storage_Administration_Guide/ch25.html#osm-target-setup
```bash
# install
yum install -y targetcli
# start and enable the service
systemctl start target && systemctl enable target

# map the backstore to a physical unformatted block device
targetcli /backstores/block create name=block_backend dev=/dev/sdb

# create an iSCSI target
targetcli /iscsi create iqn.2006-04.com.iscsi-is-awesome:1
#  Created target iqn.2006-04.com.iscsi-is-awesome:1
#  Created TPG 1.

# create the iSCSI portal (IP listener)
targetcli /iscsi/iqn.2006-04.com.iscsi-is-awesome:1/tpg1/portals/ create
# Using default IP port 3260
# Binding to INADDR_ANY (0.0.0.0)
# Created network portal 0.0.0.0:3260.

# Map the iSCSI target to the backstore
targetcli /iscsi/iqn.2006-04.com.iscsi-is-awesome:1/tpg1/luns/ create /backstores/block/block_backend
# Created LUN 0.

# crazy wild-west no-ACL mode, because this is a PoC :)
targetcli /iscsi/iqn.2006-04.com.iscsi-is-awesome:1/tpg1/ set attribute authentication=0 demo_mode_write_protect=0 generate_node_acls=1 cache_dynamic_acls=1
# Parameter demo_mode_write_protect is now '0'.
# Parameter authentication is now '0'.
# Parameter generate_node_acls is now '1'.
# Parameter cache_dynamic_acls is now '1'.


# sit back and marvel at your iSCSIs
targetcli ls
# o- / ........................................ [...]
#  o- backstores ............................. [...]
#  | o- block ................. [Storage Objects: 1]
#  | | o- block_backend  [/dev/sdb (1.0GiB) write-thru activated]
#  | o- fileio ................ [Storage Objects: 0]
#  | o- pscsi ................. [Storage Objects: 0]
#  | o- ramdisk ............... [Storage Objects: 0]
#  o- iscsi ........................... [Targets: 1]
#  | o- iqn.2006-04.com.iscsi-is-awesome:1  [TPGs: 1]
#  |   o- tpg1 .............. [no-gen-acls, no-auth]
#  |     o- acls ......................... [ACLs: 0]
#  |     o- luns ......................... [LUNs: 1]
#  |     | o- lun0  [block/block_backend (/dev/sdb)]
#  |     o- portals ................... [Portals: 1]
#  |       o- 0.0.0.0:3260 .................... [OK]
#  o- loopback ........................ [Targets: 0]

```

2. iSCSI client setup
```bash
# install and start the iscsi client service
yum install -y iscsi-initiator-utils
systemctl start iscsid.service && systemctl enable iscsid.service

# connect to the iscsi server
iscsiadm -m node -o new -T iqn.2006-04.com.iscsi-is-awesome:1 -p 33.33.33.20:3260

# login
iscsiadm -m node -T iqn.2006-04.com.iscsi-is-awesome:1 -p 33.33.33.20:3260 --login
# Logging in to [iface: default, target: iqn.2006-04.com.iscsi-is-awesome:1, portal: 33.33.33.20,3260] (multiple)
# Login to [iface: default, target: iqn.2006-04.com.iscsi-is-awesome:1, portal: 33.33.33.20,3260] successful.
```

## Base Cluster Setup
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
  #
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
  # this might show stopped, no problem
  pcs stonith show
  ```

## Option 1: Leader Election and LVM Volume failover without CLVM (using tagging and fencing)
1. on the first cluster node:
  ```bash
  
  # examine cluster status and ensure all resources are Started/Online
  pcs status
  
  # create the LVM PV and VG
  pvcreate /dev/sdb
  vgcreate shared_vg /dev/sdb
  lvcreate -l 80%VG -n ha_lv shared_vg
  
  # deactivate the shared_vg, and then reactivate it with an exclusive lock
  vgchange -an shared_vg
  #   0 logical volume(s) in volume group "shared_vg" now active
  lvchange -aey shared_vg
  #   1 logical volume(s) in volume group "shared_vg" now active
  
  # run lvs on both nodes to ensure it only says active ("a") on backend0
  lvs # on backend0
  #  LV    VG        Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
  #  root  centos    -wi-ao---- 38.48g
  #  swap  centos    -wi-ao----  1.03g
  #  ha_lv shared_vg -wi-a-----  3.20g
  lvs # on backend1
  #  LV    VG        Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
  #  root  centos    -wi-ao---- 38.48g
  #  swap  centos    -wi-ao----  1.03g
  #  ha_lv shared_vg -wi-------  3.20g
  
  # format the volume
  mkfs.xfs /dev/shared_vg/ha_lv

  ```
8. Update the initramfs device on all your cluster nodes, so that the CLVM volume is never auto-mounted:
  ```
  determine your root VG using the "vgs" command.
  
  vi /etc/lvm/lvm.conf
  # go to line 783
  on backend0, set:
    volume_list = [ "centos", "@backend0" ]
  
  on backend1, set:
    volume_list = [ "centos", "@backend1" ]
  
  # update initramfs and reboot
  dracut -H -f /boot/initramfs-$(uname -r).img $(uname -r)
  shutdown -h now
  # then vagrant up again once they're down
  ```

9. Add an LVM resource
  ```
  # first unmount and deactivate
  umount /var/opt/opscode/drbd/data
  lvchange -an shared_vg/ha_lv


  pcs resource create ha_lv ocf:heartbeat:LVM volgrpname=shared_vg exclusive=true --group chef_ha
  pcs resource debug-start ha_lv
  pcs resource create chef_data Filesystem device="/dev/shared_vg/ha_lv" directory="/var/opt/opscode/drbd/data" fstype="xfs" --group chef_ha
  pcs resource debug-start chef_data
  pcs resource create backend_vip IPaddr2 ip=33.33.33.5 cidr_netmask=24 --group chef_ha
  pcs resource debug-start backend_vip


  # debugging
  export OCF_RESKEY_volgrpname=shared_vg OCF_RESKEY_exclusive=true OCF_ROOT=/usr/lib/ocf
  bash -x /usr/lib/ocf/resource.d/heartbeat/LVM start
  ```


## Option 2:  Using CLVMD (note: I can't actually make this work)

1. On the first node: Enable clvmd and LVM clustering
  ```
  
  # enable LVM clustering with clvmd
  lvmconf --enable-cluster
  pcs resource create dlm ocf:pacemaker:controld clone on-fail=fence interleave=true ordered=true
  pcs resource create clvmd ocf:heartbeat:clvm clone on-fail=fence interleave=true ordered=true
  pcs constraint order start dlm-clone then clvmd-clone
  pcs constraint colocation add clvmd-clone with dlm-clone
  
  # stop lvmetad
  killall lvmetad
  #
  ```
6. Run the following on the second cluster node:
  ```
  pcs cluster auth backend0 backend1 -u hacluster -p hacluster
  lvmconf --enable-cluster
  
  # stop lvmetad
  killall lvmetad
  #
  ```
7. Back to the first cluster node:
  ```bash
  
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
  lvs # on backend0
  #  LV    VG        Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
  #  root  centos    -wi-ao---- 38.48g
  #  swap  centos    -wi-ao----  1.03g
  #  ha_lv shared_vg -wi-a-----  3.20g
  lvs # on backend1
  #  LV    VG        Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
  #  root  centos    -wi-ao---- 38.48g
  #  swap  centos    -wi-ao----  1.03g
  #  ha_lv shared_vg -wi-------  3.20g
  
  # format the volume and mount it
  mkfs.xfs /dev/shared_vg/ha_lv
  mount /dev/shared_vg/ha_lv /var/opt/opscode/drbd/data

  ```

9. Add an LVM resource (this is where shit gets whacky)
  ```
  # first unmount and deactivate
  umount /var/opt/opscode/drbd/data
  lvchange -an shared_vg/ha_lv


  pcs resource create ha_lv ocf:heartbeat:LVM volgrpname=shared_vg exclusive=true --group chef_ha
  pcs resource debug-start ha_lv
  pcs resource create chef_data Filesystem device="/dev/shared_vg/ha_lv" directory="/var/opt/opscode/drbd/data" fstype="xfs" --group chef_ha
  pcs resource debug-start chef_data
  pcs resource create backend_vip IPaddr2 ip=33.33.33.5 cidr_netmask=24 --group chef_ha
  pcs resource debug-start backend_vip


  # debugging
  export OCF_RESKEY_volgrpname=shared_vg OCF_RESKEY_exclusive=true OCF_ROOT=/usr/lib/ocf
  bash -x /usr/lib/ocf/resource.d/heartbeat/LVM start
  ```

Now you're done!


## Failing over

### the pacemaker way
1. bringing down backend0
  * Option 1:  shut down backend0 `shutdown -h now`
  * Option 2:  make it a standby via Pacemaker `pcs cluster standby backend0`
2. failover of resources should be automatic to the secondary node:
  ```bash
  #
  pcs status
  # Cluster name: chef-ha
  # Last updated: Sat Mar 14 21:55:56 2015
  # Last change: Sat Mar 14 21:43:21 2015 via cibadmin on backend0
  # Stack: corosync
  # Current DC: backend1 (2) - partition with quorum
  # Version: 1.1.10-32.el7_0.1-368c726
  # 2 Nodes configured
  # 4 Resources configured
  #
  #
  # Online: [ backend1 ]
  # OFFLINE: [ backend0 ]
  #
  # Full list of resources:
  #
  #  scsi (stonith:fence_scsi): Started backend1
  #  Resource Group: chef_ha
  #      ha_lv  (ocf::heartbeat:LVM): Started backend1
  #      chef_data  (ocf::heartbeat:Filesystem):  Started backend1
  #      backend_vip  (ocf::heartbeat:IPaddr2): Started backend1
  #
  # PCSD Status:
  #   backend0: Offline
  #   backend1: Online
  #
  # Daemon Status:
  #   corosync: active/enabled
  #   pacemaker: active/enabled
  #   pcsd: active/enabled
  ```
3. Bringing backend0 back up and logging in, it won't want to run pacemaker
  ```bash
  pcs status
  # Error: cluster is not currently running on this node
  systemctl pacemaker start

  # now checking status
  pcs status
  # Cluster name: chef-ha
  # Last updated: Sat Mar 14 22:06:25 2015
  # Last change: Sat Mar 14 21:43:21 2015 via cibadmin on backend0
  # Stack: corosync
  # Current DC: backend1 (2) - partition with quorum
  # Version: 1.1.10-32.el7_0.1-368c726
  # 2 Nodes configured
  # 4 Resources configured
  #
  #
  # Node backend0 (1): pending
  # Online: [ backend1 ]
  #
  # Full list of resources:
  #
  #  scsi (stonith:fence_scsi): Started backend1
  #  Resource Group: chef_ha
  #      ha_lv  (ocf::heartbeat:LVM): Started backend1
  #      chef_data  (ocf::heartbeat:Filesystem):  Started backend1
  #      backend_vip  (ocf::heartbeat:IPaddr2): Started backend1
  #
  # PCSD Status:
  #   backend0: Online
  #   backend1: Online
  #
  # Daemon Status:
  #   corosync: active/enabled
  #   pacemaker: active/enabled
  #   pcsd: active/enabled
  ```
4. If backend0 is considered standby, you can just unstandby it:
  ```
  pcs cluster unstandby backend0
  pcs status
  # Cluster name: chef-ha
  # Last updated: Sat Mar 14 22:11:53 2015
  # Last change: Sat Mar 14 22:11:52 2015 via crm_attribute on backend0
  # Stack: corosync
  # Current DC: backend1 (2) - partition with quorum
  # Version: 1.1.10-32.el7_0.1-368c726
  # 2 Nodes configured
  # 4 Resources configured
  #
  #
  # Online: [ backend0 backend1 ]
  ```
5. What happens if you try to mount the disk on the inactive node?
  ```bash
  # LVM will be nice, but won't let you
  vgchange -aey shared_vg
  # 0 logical volume(s) in volume group "shared_vg" now active
  lvchange -aey shared_vg/ha_lv
  lvs
  # LV    VG        Attr       LSize   Pool Origin Data%  Move Log Cpy%Sync Convert
  # root  centos    -wi-ao----  38.48g
  # swap  centos    -wi-ao----   1.03g
  # ha_lv shared_vg -wi------- 816.00m
  ```


### the CLVM way

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

### Do SCSI SPC-3 persistent reservations work on my device?
good:
```bash
# exampling talking to the Linux iSCSI service:
sg_persist --in --report-capabilities -v /dev/sdb
#    inquiry cdb: 12 00 00 00 24 00
#  LIO-ORG   block_backend     4.0
#  Peripheral device type: disk
#    Persistent Reservation In cmd: 5e 02 00 00 00 00 00 20 00 00
# Report capabilities response:
#  Compatible Reservation Handling(CRH): 1
#  Specify Initiator Ports Capable(SIP_C): 1
#  All Target Ports Capable(ATP_C): 1
#  Persist Through Power Loss Capable(PTPL_C): 1
#  Type Mask Valid(TMV): 1
#  Allow Commands: 1
#  Persist Through Power Loss Active(PTPL_A): 0
#    Support indicated in Type mask:
#      Write Exclusive, all registrants: 1
#      Exclusive Access, registrants only: 1
#      Write Exclusive, registrants only: 1
#      Exclusive Access: 1
#      Write Exclusive: 1
#      Exclusive Access, all registrants: 1
```

bad:
```bash
# example using Virtualbox SCSI/SAS/etc disks which are not SPC-compliant:
sg_persist --in --report-capabilities -v /dev/sdb
#    inquiry cdb: 12 00 00 00 24 00
#  VBOX      HARDDISK          1.0
#  Peripheral device type: disk
#    Persistent Reservation In cmd: 5e 02 00 00 00 00 00 20 00 00
# persistent reservation in:  Fixed format, current;  Sense key: Illegal Request
#  Additional sense: Invalid command operation code
#   Info fld=0x0 [0]
# PR in (Report capabilities): command not supported

```

TBD
