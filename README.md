# ossec.conf

According to [Wazuh docs for local configuration](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/global.html) we don't need to write logs files at all, so logging alerts.log and archives.log can be disabled

In ossec.conf add
```yml
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>no</alerts_log>
  <logall>no</logall>
  <logall_json>yes</logall_json>
  <max_output_size>1G</max_output_size>
</global>
```
What each configuration does:
- For alerts:
  - [jsonout_output](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/global.html#jsonout-output): write alerts.json
  - [alerts_log](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/global.html#alerts-log): don't write alerts.log
- For archives:
  - [logall_json](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/global.html#logall-json): write archive.json
  - [logall](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/global.html#logall): don't write archive.log
- For rotation:
  - [max-output-size](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/global.html#max-output-size): rotate archives.json and alerts.json when alerts.json reaches the value defined in max-output-size

I also tried [rotate_interval](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/global.html#logall) for rotation, it rotates archives.json and alerts.json, but doesn't compress them if multiple rotations happened in the same day (this is an [open bug](https://github.com/wazuh/wazuh/issues/35021)), so the workaround is splitting files by size and compress them with logrotate.

First remove syslog listener
```
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>any</allowed-ips>
</remote>
```

And add the following for Wazuh to monitor local files:
```
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/hosts/*.log</location>
</localfile>
```
Restart `wazuh-manager` to release 514 udp socket

# local_internal_options.conf
Because of the aforementioned compression bug, we'll be using using logrotate for compression and disabling Wazuh compression by adding this line:
```
monitord.compress=1
```

# Rsyslog

## Installing rsyslog
```
sudo apt update && sudo apt install -y rsyslog rsyslog-gnutls
```

## rsyslog.conf:
Modules can only be loaded once, so they are loaded in rsyslog.conf if used more than once in `/etc/rsyslog.d/`

In this case:
```
module(load="imudp")
module(load="imtcp")
```

## 10-tls.conf
Conf used with FortiAppSec and KSC cloud, requirements:
- DNS A record
- Valid TLS certificate and private key, preferably not a wildcard if rsyslog is installed alongside Wazuh

Followed [rsyslog doc](https://docs.rsyslog.com/doc/tutorials/tls.html)

## 11-split-tcp.conf and 12-split-udp.conf
Writes TCP and UDP logs to files at `/var/log/hosts/`

# nginx.conf

If there is already an Nginx instance, it can be used as TLS termination proxy for Syslog over TLS

It listens on 6514 (any port can be chosen) and passes raw TCP to Wazuh, where rsyslog handles it. Wazuh can't natively handle syslog over TCP or TLS

Followed [nginx doc](https://nginx.org/en/docs/stream/ngx_stream_proxy_module.html)

# logrotate

Don't forget to uncomment `compress` in `/etc/logrotate.conf`
```
# uncomment this if you want your log files compressed
compress
```

Then copy files in `logrotate.d` to `/etc/logrotate.d/`

## What each logrotate does:

### wazuh-json
Compresses alerts and archives.
- `rotate -1` keeps unlimited compressed files
```
/var/ossec/logs/alerts/*/*/*.json
/var/ossec/logs/archives/*/*/*.json
{
  rotate -1
  compress
  missingok
  notifempty
}
```

### wazuh-rsyslog
Compresses logs captured by rsyslog
- `rotate -1` keeps unlimited compressed files
- `sharedscripts` runs postrotate after all compressions rather than for each
- `postrotate` runs `/usr/lib/rsyslog/rsyslog-rotate`, a helper script that refreshes `rsyslog` file handlers
```
/var/log/hosts/*.log {
  rotate -1
  compress
  missingok
  notifempty
  sharedscripts
  postrotate
    /usr/lib/rsyslog/rsyslog-rotate
  endscript
}
```

# Cron scripts

Copy `wazuh-cron` to `/etc/cron.hourly`

```bash
#!/usr/bin/sh

logrotate -f /etc/logrotate.d/wazuh-json
logrotate -f /etc/logrotate.d/wazuh-rsyslog

# delete week old alerts
find /var/ossec/logs/alerts -type f -mtime +7 -delete

# delete 3 month old archives and rsyslog logs
find /var/ossec/logs/archives -type f -mtime +90 -delete
find /var/log/hosts -type f -mtime +90 -delete

# delete empty month directories
find /var/ossec/logs/alerts -mindepth 1 -type d -empty -delete
find /var/ossec/logs/archives -mindepth 1 -type d -empty -delete
```
- Calls logrotate
- Deletes files and directories based on a retention policy. In this case: 7 days for alerts, 3 months for archives