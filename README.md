# sqlbackup
Automated sql backup job for mariadb sql servers

Note: sqlbackup.service is designed to be used with podman. Change the ExecStart and ExecStartPre for baremetal servers

## Installation
In your podman container:
- Copy sqlbackup.sh to /usr/bin/
- Copy sqlbackup.conf to /etc/sqlbackup
- Create a new .conf file in /etc/sqlbackup/conf.d that contains at a minimum the line
```
password = <sql user password>
```
On your podman host:
- Copy sqlbackup.service and sqlbackup.timer to /etc/systemd/system/ and then run
```
systemctl daemon-reload
systemctl enable --now sqlbackup.timer
```

## Defaults
- The script expects your podman mariadb sql server container name to be 'sql'
- Defaults can be overriden in /etc/sqlbackup/conf.d/ .conf files
- Backups are stored in /backups/full/<set> and /backups/inc/<set> for full and incremental backups
- The 'root' account is used by default for the backups
- The retention is set to be 1 full and 6 incremental backups until a new full backup cycle. In other words, 1 week.
- The most recent 8 backup sets are retained, or about 2 months
  
## Running
- This script is designed to be run by the 'mysql' user (in podman that's uid 999)
- Backup sets are numbered so full backup set 2 is in the same series as incremental set 2
