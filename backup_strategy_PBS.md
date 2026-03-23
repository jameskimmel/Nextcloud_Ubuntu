# This is a backup strategy for Proxmox PVE users that backup to Proxmox PBS.
Everyone else should probably look into borg backup.  

03:00 
start maintenance mode with crontab. That way a user that is currently logged in an maybe editing a file with collabora will not loose the last keystroke letter ‘R’. Also it is transparent to the user that the system will soon go down.

Run 'sudo crontab -e'
and insert: 
'''bash
0 3 * * tue..sat docker exec --user www-data nextcloud-aio-nextcloud php occ maintenance:mode --on
'''
* Note: this crontab only backups after a weekday, from tue to sat.

03:01 
Proxmox starts backup in stop mode. It issues a shutdown command to the VM. Since AIO has no problem at all handling a OS shutdown, every container will be shut down correctly and smoothly.

03:02 
VM is down. Proxmox can do a data 100% guaranteed consistent QEMU bitmap snapshot of the VM (not a ZFS snapshot!).

03:02 
Proxmox starts the VM again. VM is getting started, while Proxmox itself is still doing a Backup to PBS.

*Note: PVE backup works with a bitmap. After the reboot, ever write the VM is doing, QEMU will first have to make sure it has already written that block to PBS, before changing it. That way it can guarantee consistency and otherwise it could wrongfully backup a newer state of the block. If your PBS destination is slow, this could severely delay the boot and start of the containers and even lead to timeouts for some applications (MSSQL, QEMU guest agent). If that is the case you should look into fleecing to mitigate this problem.

03:03 
all AIO containers have started, Nextcloud is up and running again. Maintenance mode gets automatically disabled.

03:20 
backup to PBS is finished. At what time does not matter much.