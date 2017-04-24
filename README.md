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


# License and Copyright

Copyright 2017 Christian Meutes <christian@errxtx.net>


Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
