# VM-auto-reboot-troubleshoot

### Inspect Reboot Time  
  You can check when the system reboot happened with who and last commands
```  
$ who -b
system boot 2021-02-13 20:51

$ last -x | head | tac
abhishek pts/0 192.168.1.16 Sat Feb 13 19:53 - 19:55 (00:02)
reboot system boot 3.10.0-1160.11.1 Sat Feb 13 19:55 - 20:54 (00:58)
runlevel (to lvl 3) 3.10.0-1160.11.1 Sat Feb 13 19:55 - 20:04 (00:08)
abhishek pts/0 192.168.1.16 Sat Feb 13 19:56 - 20:04 (00:07)
reboot system boot 3.10.0-1160.11.1 Sat Feb 13 20:04 - 20:54 (00:49)
runlevel (to lvl 3) 3.10.0-1160.11.1 Sat Feb 13 20:04 - 20:51 (00:46)
abhishek pts/0 192.168.1.16 Sat Feb 13 20:04 - 20:50 (00:46)
reboot system boot 3.10.0-1160.11.1 Sat Feb 13 20:51 - 20:54 (00:03)
runlevel (to lvl 3) 3.10.0-1160.11.1 Sat Feb 13 20:51 - 20:54 (00:02)
abhishek pts/0 192.168.1.16 Sat Feb 13 20:51 still logged in
$
```
### Check System Messages  
  You can further correlate the reboot you want to diagnose with system messages.
  
  For CentOS/RHEL systems, you’ll find the logs at /var/log/messages while for Ubuntu/Debian systems, its logged at /var/log/syslog. You can simply use the tail command or your favorite text editor to filter out or find specific data.
  
  As can be inferred from the below logs, such entries suggest a shutdown/reboot initiated by an administrator or root user. These messages can vary depending upon OS type and the way reboot/shutdown is triggered but you will always find useful information by looking at system logs though it may not be explicit enough to pin-point the cause every time.
```
Feb 13 19:56:20 centos7vm chronyd[637]: Source 72.30.35.89 replaced with 142.147.92.5
Feb 13 20:00:40 centos7vm chronyd[637]: Selected source 162.159.200.123
Feb 13 20:01:01 centos7vm systemd: Created slice User Slice of root.
Feb 13 20:01:01 centos7vm systemd: Started Session 2 of user root.
Feb 13 20:04:09 centos7vm systemd-logind: System is powering down.
Feb 13 20:04:09 centos7vm systemd: Closed LVM2 poll daemon socket.
Feb 13 20:04:09 centos7vm systemd: Stopped target Multi-User System.
```
One such command that you can use to filter out system logs is given below:
```
sudo grep -iv ': starting\|kernel: .*: Power Button\|watching system buttons\|Stopped Cleaning Up\|Started Crash recovery kernel' \
  /var/log/messages /var/log/syslog /var/log/apcupsd* \
  | grep -iw 'recover[a-z]*\|power[a-z]*\|shut[a-z ]*down\|rsyslogd\|ups'
```
Captured events may not always be specific. Always trace out events that give signs of warnings or errors which may lead to the system powering off/crashing.
   
### Verify auditd Logs
For systems with auditd, it is a great place to check different events using ausearch tool. Use the below command to check the last two entries from audit logs.
```
$ sudo ausearch -i -m system_boot,system_shutdown | tail -4
```
This will report the two most recent shutdowns or reboots. If this reports a SYSTEM_SHUTDOWN followed by a SYSTEM_BOOT, everything should be good. But, if it reports two SYSTEM_BOOT lines in a row or only a single SYSTEM_BOOT line, then most likely, the system did not shut down gracefully. A normal output should be something like below:
```  
$ sudo ausearch -i -m system_boot,system_shutdown | tail -4
----
type=SYSTEM_SHUTDOWN msg=audit(Saturday 13 February 2021 A.852:8) : pid=621 uid=root auid=unset ses=unset subj=system_u:system_r:init_t:s0 msg=' comm=systemd-update-utmp exe=/usr/lib/systemd/systemd-update-utmp hostname=? addr=? terminal=? res=success'
----
type=SYSTEM_BOOT msg=audit(Saturday 13 February 2021 A.368:8) : pid=622 uid=root auid=unset ses=unset subj=system_u:system_r:init_t:s0 msg=' comm=systemd-update-utmp exe=/usr/lib/systemd/systemd-update-utmp hostname=? addr=? terminal=? res=success'
$
The below output lists two consecutive SYSTEM_BOOT messages, which may indicate an ungraceful shutdown though it needs to be correlated with system logs.

$ sudo ausearch -i -m system_boot,system_shutdown | tail -4
----
type=SYSTEM_BOOT msg=audit(Saturday 13 February 2021 A.852:8) : pid=621 uid=root auid=unset ses=unset subj=system_u:system_r:init_t:s0 msg=' comm=systemd-update-utmp exe=/usr/lib/systemd/systemd-update-utmp hostname=? addr=? terminal=? res=success'
----
type=SYSTEM_BOOT msg=audit(Saturday 13 February 2021 A.368:8) : pid=622 uid=root auid=unset ses=unset subj=system_u:system_r:init_t:s0 msg=' comm=systemd-update-utmp exe=/usr/lib/systemd/systemd-update-utmp hostname=? addr=? terminal=? res=success'
$
Analyze systemd journal
You should have a persistent systemd-journal in order to keep a persistent journal on disk else the logs won’t persist on reboot. For this, you can either make the changes in /etc/systemd/journald.conf or create the directory yourself with the below commands:

$ sudo mkdir /var/log/journal
$ sudo systemd-tmpfiles --create --prefix /var/log/journal 2>/dev/null
$ sudo systemctl -s SIGUSR1 kill systemd-journald
```
  Once done, you can optionally reboot the system to capture more than one reboot entry in the journal though it is not required.
  
  Use the below command to list logged boots from the journal:
```  
$ journalctl --list-boots
```
  Here’s its output on my server:
```  
$ journalctl --list-boots
-15 8a7c8034da804ebb9cb063a7553ed0bf Wed 2020-11-18 23:09:05 IST—Wed 2020-11-18 23:17:10 IST
-14 7bbb9542778a4057a91b9d22fcf91735 Wed 2020-11-18 23:17:22 IST—Wed 2020-11-18 23:20:08 IST
-13 f2ee8a61bf4c4f67a12e012855d8b1c3 Wed 2020-11-18 23:20:17 IST—Wed 2020-11-18 23:23:01 IST
-12 1277d19a959f4c33ba944a68c5874d2a Fri 2020-12-11 10:32:44 IST—Fri 2020-12-11 10:43:39 IST
-11 eb4ff97f112445888a5946d1155de1b8 Fri 2020-12-11 10:43:55 IST—Fri 2020-12-11 10:48:18 IST
-10 bf46eff3f9a344d2b28a03ffbf7fff32 Fri 2020-12-11 19:04:30 IST—Fri 2020-12-11 19:31:01 IST
 -9 2acf08368667423c89086579f98efd82 Tue 2020-12-15 17:36:52 IST—Tue 2020-12-15 19:13:10 IST
 -8 b826f223a67d454b94d4413678870f08 Sat 2020-12-19 00:31:54 IST—Sat 2020-12-19 00:44:52 IST
 -7 011e1b29339041b0ae48bbb93fce792f Wed 2020-12-23 23:01:15 IST—Wed 2020-12-23 23:02:44 IST
 -6 f41f5880572e4394938c6dcb4a8b683c Mon 2020-12-28 16:54:11 IST—Mon 2020-12-28 22:54:22 IST
 -5 a2e638dc292a4db2b0a50dd442129c28 Tue 2020-12-29 17:02:16 IST—Tue 2020-12-29 19:39:38 IST
 -4 f6c738df872a48d48daee1962727cca5 Wed 2020-12-30 19:09:30 IST—Wed 2020-12-30 19:20:23 IST
 -3 c876e60ea371460b94e247b40270b18f Thu 2020-12-31 14:36:07 IST—Thu 2020-12-31 15:45:36 IST
 -2 a23c70804ec243f7868c18737f4b7e55 Sat 2021-02-13 20:09:30 IST—Sat 2021-02-13 20:10:44 IST
 -1 94b604a6bf75462dac8c4a4576fdc863 Sat 2021-02-13 20:10:59 IST—Sat 2021-02-13 20:23:18 IST
  0 3ff7e29fa0a34878b7574b7d4d3ccfb5 Sat 2021-02-13 20:24:57 IST—Sat 2021-02-13 21:13:15 IST
$
```
As you can see it is listing lasts several boots. To further analyze a particular reboot, use:
  
  ```
$ journalctl -b {num} -n
Here {num} will be the index given in journalctl --list-boots command in the first column.

$ journalctl -b -1 -n
-- Logs begin at Wed 2020-11-18 23:09:05 IST, end at Sat 2021-02-13 21:13:39 IST. --
Feb 13 20:23:18 ubuntumate20vm systemd[1]: lvm2-monitor.service: Succeeded.
Feb 13 20:23:18 ubuntumate20vm systemd[1]: Stopped Monitoring of LVM2 mirrors, snapshots etc. using dmeventd or progress polling.
Feb 13 20:23:18 ubuntumate20vm systemd[1]: Reached target Shutdown.
Feb 13 20:23:18 ubuntumate20vm systemd[1]: Reached target Final Step.
Feb 13 20:23:18 ubuntumate20vm systemd[1]: systemd-poweroff.service: Succeeded.
Feb 13 20:23:18 ubuntumate20vm systemd[1]: Finished Power-Off.
Feb 13 20:23:18 ubuntumate20vm systemd[1]: Reached target Power-Off.
Feb 13 20:23:18 ubuntumate20vm systemd[1]: Shutting down.
Feb 13 20:23:18 ubuntumate20vm systemd-shutdown[1]: Syncing filesystems and block devices.
Feb 13 20:23:18 ubuntumate20vm systemd-journald[304]: Journal stopped
$
```
You can observe messages logged in the journal in the above output and can trace out the anomalies if any.

