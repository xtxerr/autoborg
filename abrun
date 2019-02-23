#!/bin/bash
#
#
################################################################################
# Borgbackup automation of LVM-based virtual-machines
#
# This script needs to be run by cron
# Connects to REMOTE_HOSTS via SSH pubkey authentication, 
# checks for running VMs and backup those to the local REPO. 
#
# *** ALL VMS NEED TO HAVE THEIR DISKS FROM LVM-BASED VOLUMES ***
#     eg. /dev/myvolumegroup/host_jimmy
#
# Each VM is snapshotted and the filesystem is freezed for a short moment (1-5s)
# Then the VM is backuped, while only the chunks/blocks which differ from the last
# backup are stored. The backups are stored as a complete VM disk file inside of
# a borg-based repository. Between different backups only the chunks/blocks which
# have changed are backuped, so that a complete backup of a 100G host-volume can
# lead to just few MegaBytes beeing backuped.
# 
# More details about borg on the borg website: http://borgbackup.readthedocs.io/
################################################################################
#
### BEGIN INIT ###
#
# Borg repository where backups should be stored on LOCALHOST
REPO=/mnt/backup/borg/
#
# Host where this script is running on and where the borg repo lies
LOCALHOST="my_backup_host_named_mcdata"
#
# remote hosts with running VMs on LVM to be backuped
# space separated list
REMOTE_HOSTS="remote_host_named_snoopy"
#
# keep full backups for a period of days
KEEP_DAYS=5
#
# ssh user account of localhost which is used from remote host by Borg
BORG_SSH_USER=root
#
# LVM snapshot size
SIZE=50G
#
# backup partition / mount point
DISK=/mnt/backup
#
# disk space usage on $DISK which is allowed. If its hitting this threshold backup won't run.
ALLOWED=80
#
### END INIT ###

# date + hour + mine
DATETIME=$(date +"%Y-%m-%d-%H%M")

# borg environemnt vars (do need to bet set explicitely in remote SSH sessions)
ENV="BORG_UNKNOWN_UNENCRYPTED_REPO_ACCESS_IS_OK=yes BORG_RELOCATED_REPO_ACCESS_IS_OK=yes"
export ${ENV}

# return values of various commands to check for success
REGEX_FREEZE="Froze [1-9]+ filesystem"
REGEX_UNFREEZE="Thawed [1-9]+ filesystem"
REGEX_LVCREATE="Logical volume \".*\" created"
REGEX_LVREMOVE="successfully removed"

# start time of this script
START=$(date +%s.%N)

# check for enough disk space and exit if it's below $SPACE
USED=$(df ${DISK} | awk '{print $5}' | sed -ne 2p | cut -d"%" -f1)
if [[ ${USED} -gt ${ALLOWED} ]]; then
	echo "${USED}% of disk space on backup disk ${DISK} used, but you want to have" \
		 "${SPACE}% to be free, FATAL!!! Not backuping anything."
	exit 1
fi

# check that another borgbackup isn't already running
if pidof -x $(basename $0) > /dev/null; then
  for p in $(pidof -x $(basename $0)); do
    if [ $p -ne $$ ]; then
      echo "Script $0 is already running: exiting"
      exit
    fi
  done
fi

# Create borg repository for the local host
borg init --encryption=none ${REPO}${LOCALHOST}

if [[ "$?" -eq 0 ]]; then
	echo "Created new repository: ${REPO}${LOCALHOST}"
fi

# remote backups, everything is done via SSH remotely
backup() {
	HOST=$1

	# dont prepend each command with ssh when its the localhost
	if [[ ${HOST} != ${LOCALHOST} ]]; then
		SSH="ssh ${HOST} ${ENV} "
		SSH_REPO="ssh://${BORG_SSH_USER}@${LOCALHOST}"
	else
		SSH=""
		SSH_REPO=""
	fi
	for VM in $(${SSH}virsh list --name); do
		echo "----------------------------------------"
		echo "----------------------------------------"
		echo "Backing up KVM guest ${VM} on ${HOST}"
		echo ${DATETIME}
		echo "----------------------------------------"

		# Make sure, snapshot merging was successfull
		LV=$(${SSH}virsh domblklist ${VM} --details | \
					grep vda | \
					awk '{ print $4; }' | \
					cut -d "/" -f -5)

		# Freeze the VM filesystem
		echo "Freezing filesystem of ${VM}"

		FREEZE=$(${SSH}virsh domfsfreeze ${VM})

		if [[ ! $FREEZE =~ $REGEX_FREEZE ]]; then
			echo "couldn't freeze the filesystem of ${VM}, FATAL!!!" >&2
			exit 1
		else
			echo "successfully froze filesystems of ${VM} on ${HOST}"

			echo "creating LVM snapshot of ${LV} on ${HOST}"

			LVCREATE=$(${SSH}lvcreate --size ${SIZE} --snapshot --name ${VM}_snapshot ${LV})

			if [[ ! $LVCREATE =~ $REGEX_LVCREATE ]]; then
				echo "couldn't create snapshot of ${VM} on ${HOST}, FATAL!!!" >&2

				# unfreeze the filesystem again after snapshot failed
				UNFREEZE=$(${SSH}virsh domfsthaw ${VM})

				if [[ ! $UNFREEZE =~ $REGEX_UNFREEZE ]]; then
					echo "couldn't unfreeze the filesystem of ${VM} on ${HOST}, FATAL!!!!" >&2
					exit 1
				fi
				echo "unfreezed filesystem on ${VM} running on ${HOST}"
				exit 1
			else
				# successfully created snapshot
				echo "successfully created snapshot ${VM}_snapshot on ${HOST}"

				# unfreeze the filesystem again after snapshot was taken
				UNFREEZE=$(${SSH}virsh domfsthaw ${VM})

				if [[ ! $UNFREEZE =~ $REGEX_UNFREEZE ]]; then
					echo "couldn't unfreeze the filesystem of ${VM} on ${HOST}, FATAL!!!!" >&2
				else
					echo "unfreezed the filesystem of ${VM} on ${HOST}"

					ARCHIVE_PATH="${REPO}${HOST}::${VM}_${DATETIME} ${LV}_snapshot"
					echo "doing borgbackup on ${HOST}:"
					echo ${ARCHIVE_PATH}

					VM_START=$(date +%s.%N)

					# successfully unfreezed again and now doing backups of the snapshot
					${SSH}borg create \
						--error \
						--list \
						--stats \
						--compress lz4 \
						--read-special ${SSH_REPO}${ARCHIVE_PATH}

					if [[ "$?" -ne 0 ]]; then
						echo "Unable to backup ${VM} on ${HOST}, FATAL!!!" >&2
					else
						echo "successfully backuped ${VM} on ${HOST}"
						VM_END=$(date +%s.%N)
						VM_DIFF=$(echo "($VM_END - $VM_START) / 60" | bc)
						echo "successfully backuped ${VM} in ${VM_DIFF} minutes."
					fi
				fi
				# remove the snapshot
				LVREMOVE=$(${SSH}lvremove -f ${LV}_snapshot)

				if [[ $LVREMOVE =~ $REGEX_LVREMOVE ]]; then
					echo "successfully remove snapshot ${VM}_snapshot on ${HOST}"
				else 
					echo "couldn't remove snapshot ${VM}_snapshot on ${HOST}, FATAL!!!" >&2
					exit 1
				fi
			fi
		fi
	done
}

prune_backup() {
	PRUNE_REPO=${REPO}$1
	borg prune -v --list --keep-within=${KEEP_DAYS}d $PRUNE_REPO
}

HOSTLIST="${LOCALHOST} ${REMOTE_HOSTS}"

# remote backups via SSH to local machine
for HOST in ${HOSTLIST}; do 
	# Create borg repository for the local host
	echo "============ Host ${HOST} ============="
	borg init --encryption=none ${REPO}${HOST}

	if [[ "$?" -eq 0 ]]; then
		echo "Created new repository: ${REPO}${HOST}"
	fi
	# do backups for each remote host
	backup ${HOST}

	# prune backups for each remote host (as they have all their own repo)
	prune_backup ${HOST}
done

# print execution time of this script
END=$(date +%s.%N)
DIFF=$(echo "($END - $START) / 60" | bc)
echo "--------------------------------------------------"
echo "--------------------------------------------------"
echo "Complete Backup run took ${DIFF} minutes in total."
echo "Good bye"
echo "=================================================="

