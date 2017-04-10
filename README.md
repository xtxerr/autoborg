# Borgbackup automation of LVM-based virtual-machines

Connects to REMOTE_HOSTS via SSH pubkey authentication (root user), 
checks for running VMs and backup those to the local REPO. Also checks for
running VMs on LOCALHOST itself and backups those as well.

#### ALL VMS NEED TO HAVE THEIR DISKS FROM LVM-BASED VOLUMES
eg. /dev/myvolumegroup/host_jimmy

Each VM is snapshotted and the filesystem is freezed for a short moment (1-5s).
Then the VM is backuped, while only the chunks/blocks which differ from the last
backup are stored. The backups are stored as a complete VM disk file inside of
a borg-based repository. Between different backups only the chunks/blocks which
have changed are backuped, so that a complete backup of a 100G host-volume can
lead to just few MegaBytes beeing backuped.
 
More details about borg on the borg website: http://borgbackup.readthedocs.io/
