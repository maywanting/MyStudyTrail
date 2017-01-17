title: "linux下配置squid实现手机抓包"
date: 2017-01-12 22:28:09
tags:
- linux
- squid
categories:
- experince
---


/etc/squid/squid.conf

```
http_port 8086
http_access allow all
visible_hostname maywanting
access_log daemon:/var/log/squid/access.log squid
cache_dir ufs /var/spool/squid 100 16 256
pid_filename /var/run/squid/squid.pid
```

squiz -z

```
FATAL: Failed to make swap directory /var/spool/squid/00: (13) Permission denied

Squid Cache (Version 3.5.12): Terminated abnormally.

CPU Usage: 0.012 seconds = 0.008 user + 0.004 sys

Maximum Resident Size: 44384 KB

Page faults with physical i/o: 0
```
chmod -R 777 /var/log/squid/




```
2016/09/01 11:26:44 kid1| Current Directory is /var/log/squid

2016/09/01 11:26:44 kid1| Creating missing swap directories

2016/09/01 11:26:44 kid1| /var/spool/squid exists

FATAL: Failed to make swap directory /var/spool/squid/00: (13) Permission denied
```
chmod -R 777 /var/spool/squid/



squid -NCd1

```
2016/09/01 11:29:07| /var/run/squid.pid: (13) Permission denied

2016/09/01 11:29:07| Closing HTTP port [::]:8086

FATAL: Could not write pid file

[1]    4007 abort (core dumped)  squid -NCd1
```


touch /var/run/squid.pid

sudo chmod 777 /var/run/squid.pid
