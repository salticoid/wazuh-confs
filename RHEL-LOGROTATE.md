# Building logrotate 3.19.0+
[Logrotate repo](https://github.com/logrotate/logrotate) has compiling instructions, short version is
```bash
# clone to tmp
cd /tmp
git clone https://github.com/logrotate/logrotate.git -b main

# build
cd logrotate
yum install autoconf automake libtool make popt-devel xz -y
cd logrotate
autoreconf -fiv
./configure
make

# check it worked, version should be >3.19.0
./logrotate -v
```

# Find logrotate path
```bash
which logrotate
```

Replace the given path in the line below
```bash
sudo cp /tmp/logrotate/logrotate [path]
```

The reason we replace the >3.19.0 logrotate binary instead of uninstalling is to reuse the existing systemd timer and service 

# Check current logrotate version
```bash
logrotate -v
```
Should be >3.19.0
# Verify it can run
``` bash
systemctl restart logrotate
```

## If issues are found, any of the commands below might help

``` bash
chmod 644 /etc/logrotate.d/wazuh-*
chmod 644 /var/lib/logrotate/logrotate.status
chown root:root /var/lib/logrotate/logrotate.status
restorecon -v /etc/logrotate.d/wazuh-json
restorecon -v /etc/logrotate.d/wazuh-rsyslog
restorecon -v /var/lib/logrotate/logrotate.status
restorecon -v /var/lib/logrotate.status
```

# Ignore logrotate 

Add logrotate to ignored packages so dnf doesn't replace it with an "updated" version that is still <3.19.0

To `/etc/dnf/dnf.conf` add the line 
```bash
excludepkgs=logrotate
```
