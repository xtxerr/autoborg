### Backup automation of LVM-based virtual-machines based on borgbackup ###
-----

- VMs are snapshotted and filesystems freezed
- Backups only differing chunks/blocks out of the VMs logical-volume
- hosts (libvirt-supporting hypervisors) can be local or remote
- loops through the list of hosts and backups to a local repo (borgbackup)
