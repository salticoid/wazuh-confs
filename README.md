# Sample configurations for syslog

## Installing rsyslog
```
apt update && apt install rsyslog-gnutls -y
```

## rsyslog.conf:

Modules can only be loaded once, so they are loaded in rsyslog.conf if used more than once in rsyslog.d/

In this case: 
```
module(load="imudp")
module(load="imtcp")
```

## 10-tls.conf
Conf used with FortiAppSec, requirements:
- DNS A record
- Valid TLS certificate and private key, preferably not a wildcard if rsyslog is installed alongside Wazuh

Followed [rsyslog doc](https://docs.rsyslog.com/doc/tutorials/tls.html)

## 11-split-tcp.conf and 12-split-udp.conf
Writes TCP and UDP logs to files

## ossec.conf

Remove syslog listener
```
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>any</allowed-ips>
</remote>
```

And add as needed:

- Monitor a single log file
```
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/file.log</location>
</localfile>
```
- Monitor a dir of log files
```
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/hosts/*.log</location>
</localfile>
```
Restart wazuh-manager to release 514 udp socket, then start rsyslog

Check `ss -tulpn` output for "rsyslogd" process

## nginx.conf

If there is already an Nginx instance, it can be used as TLS termination proxy for Syslog over TLS

It listens on 6514 (any port can be chosen) and passes raw TCP to Wazuh, where rsyslog handles it. Wazuh can't natively handle syslog over TCP or TLS

Followed [nginx doc](https://nginx.org/en/docs/stream/ngx_stream_proxy_module.html)
