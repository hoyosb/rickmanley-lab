# 6 node gluster deployment with nfs-ganesha + cache drive
## Assumptions
This deployment assumes the following:
- The `1-gluster-deploy.yml` playbook is run from a control node, and accomplishes the following:
  - deploys 7 VMs (6 Gluster servers and 1 Gluster client)
  - The 6 Gluster server VMs have 2 additional disks attached, for a total of 3 (vda, vdb, and vdc)
  - registering with Customer Portal
  - enabling repos
  - configuring ssh-keys
  - configuring known_hosts
- Make sure to update any variables, hostnames, sizes, etc within each playbook. In the future, I may update this repo with a group_vars/all file to have 1 location for variables.

## Gotchas
Resetting the environment is still being tested. Testing the `ganesha_destroy.conf`, we found that the line for the gluster_shared_storage was still in `/etc/fstab`. I believe we need to add a `lineinfile` module to the `0-reset.yml` playbook to ensure that line gets deleted.

The 'corosync', 'pacemaker', and 'nfs-ganesha' daemons have been enabled to run at boot. This is ***not*** the default behavior. Enabling them to run at boot is good practice for HA, but it may not be the desired behavior for your use case. If you choose to disable these at boot, the following commands can be run on a node to get it joined back into the cluster and exporting the share:

- pcs status
- pcs cluster start #### --all should be added if we're starting the pcs cluster on EVERY node.
- Wait for `pcs status` to report all resources up and running
- ansible gluster -a "systemctl restart nfs-ganesha"

Last gotcha... I noticed some lag on a node rebooting where the 'nfs-ganesha' daemon did not initialize properly. Restarting the daemon fixed this... looks like a timing issue on bootup. We could probably add delays to the start to resolve this.

## General Steps:
1. Ensure that the Control Node's `/etc/ansible/hosts` file includes the following:
```
[gluster]
gluster-[1:6].rnelson-demo.com
```
2. Deploy the infrastructure from the Control Node: `ansible-playbook 1-gluster-deploy.yml`
3. Once deployed, log into `gluster-1.rnelson-demo.com` and ensure that the `/etc/ansible/hosts` file also includes the following:
```
[gluster]
gluster-[1:6].rnelson-demo.com
```
4. Deploy the Gluster cluster and configure NFS Ganesha from `gluster-1.rnelson-demo.com`: `gdeploy -c 2-gdeploy-ganesha.conf`
5. Configure the cache drive (by default, vdc) from `gluster-1.rnelson-demo.com`: `gdeploy -c 3-gdeploy-cache.conf`
6. Enable corosync and pacemaker to start on boot:
  - ansible gluster -a "systemctl enable corosync"
  - ansible gluster -a "systemctl enable pacemaker"
  - ansible gluster -a "systemctl enable nfs-ganesha"
7. ***only for troubleshooting*** If you need to reset the environment:
  - `gdeploy -c ganesha_destroy.conf`
  - `ansible-playbook 0-reset.yml`

## Exporting a subdirectory
### Prepare the filesystem from the client:
- mkdir /mnt/ganesha
- mount -t glusterfs gluster-1.rnelson-demo.com:/ganesha /mnt/ganesha/
- mkdir -p /mnt/ganesha/directory1/subdirectory1
- mkdir -p /mnt/ganesha/directory2/subdirectory2
- touch /mnt/ganesha/directory1/subdirectory1/file-{1..5}
- touch /mnt/ganesha/directory2/subdirectory2/file-{6..10}
- touch /mnt/ganesha/directory1/file-{11..15}
- touch /mnt/ganesha/directory2/file-{16..20}
- umount /mnt/ganesha

### Create 2 additional entries for subdirectory exports on gluster-1.rnelson-demo.com:
The `/var/run/gluster/shared_storage/nfs-ganesha/exports/export.ganesha.conf` file contains the exports. You can create separate export files, and reference them with 'include' lines in the original export file, but this example is a configuration of having all exports defined within one file:
```
[root@gluster-1 exports]# cat export.ganesha.conf
EXPORT{
      Export_Id = 2;
      Path = "/ganesha";
      FSAL {
           name = GLUSTER;
           hostname="localhost";
          volume="ganesha";
           }
      Access_type = RW;
      Disable_ACL = true;
      Squash="No_root_squash";
      Pseudo="/ganesha";
      Protocols = "3", "4" ;
      Transports = "UDP","TCP";
      SecType = "sys";
     }

EXPORT{
      Export_Id = 3;
      Path = "/ganesha/directory1";
      FSAL {
           name = GLUSTER;
           hostname="localhost";
          volume="ganesha";
	  volpath="/directory1";
           }
      Access_type = RW;
      Disable_ACL = true;
      Squash="No_root_squash";
      Pseudo="/ganesha/directory1";
      Protocols = "3", "4" ;
      Transports = "UDP","TCP";
      SecType = "sys";
     }

EXPORT{
      Export_Id = 4;
      Path = "/ganesha/directory2";
      FSAL {
           name = GLUSTER;
           hostname="localhost";
          volume="ganesha";
          volpath="/directory2";
           }
      Access_type = RW;
      Disable_ACL = true;
      Squash="No_root_squash";
      Pseudo="/ganesha/directory2";
      Protocols = "3", "4" ;
      Transports = "UDP","TCP";
      SecType = "sys";
     }
```
After updating/creating the exports, run the following on `gluster-1.rnelson-demo.com`:
```
/usr/libexec/ganesha/ganesha-ha.sh --refresh-config /var/run/gluster/shared_storage/nfs-ganesha/ ganesha
```
You'll get some errors similar to below... these can be ignored as we'll be restarting the nfs-ganesha daemon on all nodes. This block is just informational:
```
Refresh-config failed on gluster-2. Please check logs on gluster-2
Refresh-config failed on gluster-3. Please check logs on gluster-3
Refresh-config failed on gluster-4. Please check logs on gluster-4
Refresh-config failed on gluster-5. Please check logs on gluster-5
Refresh-config failed on gluster-6. Please check logs on gluster-6
Refresh-config failed on localhost.
```
Now restart nfs-ganesha on all gluster nodes.
- ansible gluster -a "systemctl restart nfs-ganesha"

Verify that the mounts show up:
- showmount -e gluster-3.rnelson-demo.com

### Testing from client:
- mount -t nfs -o vers=4.0 192.168.1.102:/ganesha/directory1/ /mnt/ganesha/
