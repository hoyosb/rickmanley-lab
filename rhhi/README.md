# RHHI HomeLab

This guide focuses on deploying a 3-node Red Hat Hyperconverged Infrastructure (RHHI) through Ansible Playbooks, fully functioning with little to no additional configuration needed, all within 1.5 hour.

## Assumptions
The kickstart files set up the OS disk (`sda`) with additional partitions (`sda4` and `sda5`)to be used for RHHI's storage domains. These Supermicro servers don't have a lot of physical space in them for another 2.5" drive, so I elected to use a larger `sda` disk and partition it out.

I've included the kickstart files for each RHHI node under the `files/` directory in this repo, just as a reference.

Device layout:
- /dev/sda is 500 GiB:
  - /dev/sda1 - 256 MiB - /boot/efi
  - /dev/sda2 - 1024 MiB - /boot
  - /dev/sda3 - 49 GiB - Operating System
  - /dev/sda4 - 75 GiB - engine volume
  - /dev/sda5 - 375 GiB - ssdvmstore volume
- /dev/nvme0n1 - 225 GiB - nvmevmstore volume

## Requirements
1. This deployment expects 3 partitions/disks. I want to change this to allow more flexibility, which means first order of business is to switch the `configure-gluster` role from using `gdeploy` to using the LVM ansible modules. Coming soon...
2. Update your variable declarations in `group_vars/all.bak` and rename this to `group_vars/all`.
3. Ensure the control node has the `python-ovirt-engine-sdk4` RPM installed.
4. Update your control node's `/etc/ansible/hosts` file to use 'rhhi' as the host selector:
```
[rhhi]
rhhi[1:3].rnelson-demo.com
```

## General Steps
Below are the general steps.

1. Power on system, boot from network, select node's specific kickstart file.
2. ansible-playbook -u root 1-prep.yml -k
3. ansible-playbook -u root 2-storage-he-hosts-network.yml -e @group_vars/all
4. ansible-galaxy install ovirt.image-template
5. Update ovirt.image-template variables.
6. Download copy of kvm image and upload to a shared folder. I'm using an FTP share, so my `qcow_url` value is "ftp://{{qcow_url_ip}}/PXE/isos/rhel-7-kvm-image.qcow2"
7. ansible-playbook 3-ovirt-image-template.yml -e @group_vars/all

## Testing Roles General Steps
Below are the general steps.

1. Power on system, boot from network, select node's specific kickstart file.
2. ansible-playbook -u root main.yml -k
3. ansible-galaxy install ovirt.image-template
4. Update ovirt.image-template variables.
5. Download copy of kvm image and upload to a shared folder. I'm using an FTP share, so my `qcow_url` value is "ftp://{{qcow_url_ip}}/PXE/isos/rhel-7-kvm-image.qcow2"
6. ansible-playbook 3-ovirt-image-template.yml -e @group_vars/all

## Remaining tasks to complete:
- document ovirt.image-template... possibly just link to their github role
- test without /etc/hosts failed. Test again (use ip addr for gluster network)
- convert gdeploy conf file to playbook
- tag taxonomy

## Nice to have:
- The SYS-E200-8D systems require an OOB license to be able to use the Redfish APIs through a Redfish enabled firmware.
