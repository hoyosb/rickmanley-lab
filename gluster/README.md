


on client:
register, enable repo, update, install glusterfs-fuse
[root@gluster-client ~]# mount -t glusterfs gluster-1.rnelson-demo.com:/ganesha /mnt/client/
[root@gluster-client ~]# cd /mnt/client/
[root@gluster-client client]# mkdir -p directory1/subdirectory1
[root@gluster-client client]# mkdir -p directory2/subdirectory2
[root@gluster-client client]# touch directory1/subdirectory1/file-{1..5}
[root@gluster-client client]# touch directory2/subdirectory2/file-{6..10}
[root@gluster-client client]# touch directory1/file-{11..15}
[root@gluster-client client]# touch directory2/file-{16..20}


On gluster-1:
/var/run/gluster/shared_storage/nfs-ganesha/exports/export.ganesha.conf

[root@gluster-1 exports]# cat export.ganesha.conf
# WARNING : Using Gluster CLI will overwrite manual
# changes made to this file. To avoid it, edit the
# file and run ganesha-ha.sh --refresh-config.
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

/usr/libexec/ganesha/ganesha-ha.sh --refresh-config /var/run/gluster/shared_storage/nfs-ganesha/ ganesha
Then restart nfs-ganesha on gluster-1, gluster-2, and gluster-3