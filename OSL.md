Installing the latest OpenStack Swift doc

System Info
2core virtual CPU
6gb ram
Ubuntu 22

First adding users to the user group

```bash
sudo groupadd ${USER} && sudo gpasswd -a ${USER} ${USER} && newgrp ${USER}
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/b02bcbe7-fdab-403f-b657-23b72d48bf37)

**Installing dependencies**

```bash
sudo apt-get update
sudo apt-get install curl gcc memcached rsync sqlite3 xfsprogs \
                     git-core libffi-dev python-setuptools \
                     liberasurecode-dev libssl-dev
```
unfortunately, we found errors while installing other dependencies
```bash
sudo apt-get install python-coverage python-dev python-nose \
                     python-xattr python-eventlet \
                     python-greenlet python-pastedeploy \
                     python-netifaces python-pip python-dnspython \
                     python-mock
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/91fc9bdf-492a-4087-92f7-74950a1e8f56)

we will go one by one and recheck the version of Python packages to change
```bash
sudo apt-get install python3-coverage python3-dev python3-nose
sudo apt-get install python3-xattr python3-eventlet
sudo apt-get install python3-greenlet python3-pastedeploy
sudo apt-get install python3-netifaces python3-pip python3-dnspython
sudo apt-get install python3-mock
```
**Configuring storage**

Swift requires some space on XFS filesystems to store data and run tests.

I added a new VHD with 20GB of space, let's check it out
```bash
sudo lshw -C disk -short
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/3dc8e62c-36fb-4802-8dbf-0113a8c4bcc0)

Set up a single partition on the device (this will wipe the drive):
```bash
sudo parted /dev/sdb mklabel msdos mkpart p xfs 0% 100%
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/827dd0d4-f34e-41f9-b55d-144a7f3aa78a)

Create an XFS file system on the partition:
```bash
sudo mkfs.xfs /dev/sdb1
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/69424da0-8fb5-462e-ae99-53d2a8ad8813)

Find the UUID of the new partition:
```bash
sudo blkid
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/f97b3f4d-e3af-47f9-b1a5-0ad1265be3a9)

Edit /etc/fstab and add:

UUID="<UUID-from-output-above>" /mnt/sdb1 xfs noatime 0 0

Create the Swift data mount point and test that mounting works:

```bash
sudo mkdir /mnt/sdb1
sudo mount -a
```
**Common Post-Device Setup**

Create the individualized data links:
```bash
sudo mkdir /mnt/sdb1/1 /mnt/sdb1/2 /mnt/sdb1/3 /mnt/sdb1/4
sudo chown ${USER}:${USER} /mnt/sdb1/*
for x in {1..4}; do sudo ln -s /mnt/sdb1/$x /srv/$x; done
sudo mkdir -p /srv/1/node/sdb1 /srv/1/node/sdb5 \
              /srv/2/node/sdb2 /srv/2/node/sdb6 \
              /srv/3/node/sdb3 /srv/3/node/sdb7 \
              /srv/4/node/sdb4 /srv/4/node/sdb8
sudo mkdir -p /var/run/swift
sudo mkdir -p /var/cache/swift /var/cache/swift2 \
              /var/cache/swift3 /var/cache/swift4
sudo chown -R ${USER}:${USER} /var/run/swift
sudo chown -R ${USER}:${USER} /var/cache/swift*
# **Make sure to include the trailing slash after /srv/$x/**
for x in {1..4}; do sudo chown -R ${USER}:${USER} /srv/$x/; done
```
Restore appropriate permissions on reboot.

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/4148003c-1512-4152-92ec-9dd7ba68798c)

Creating an XFS tmp dir

Create the file for the tmp loopback device:


```bash
sudo mkdir -p /srv
sudo truncate -s 1GB /srv/swift-tmp  # create 1GB file for XFS in /srv
sudo mkfs.xfs /srv/swift-tmp

```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/58bc6bc8-34c5-42c9-a9be-de887a071142)

To mount the tmp loopback device at /tmp, do the following:
```bash
sudo mount -o loop,noatime /srv/swift-tmp /tmp
sudo chmod -R 1777 /tmp
```
To persist this, edit and add the following to /etc/fstab:

```bash
/srv/swift-tmp /tmp xfs rw,noatime,attr2,inode64,noquota 0 0
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/813e22c7-d866-4c05-a9d8-5a92463a8974)


To mount the tmp loopback at an alternate location (for example, /mnt/tmp):

```bash
sudo mkdir -p /mnt/tmp
sudo mount -o loop,noatime /srv/swift-tmp /mnt/tmp
sudo chown ${USER}:${USER} /mnt/tmp
```
To persist this, edit and add the following to /etc/fstab:

```bash
/srv/swift-tmp /mnt/tmp xfs rw,noatime,attr2,inode64,noquota 0 0
```

Set your TMPDIR environment dir so that Swift looks in the right location:

```bash
export TMPDIR=/mnt/tmp
echo "export TMPDIR=/mnt/tmp" >> $HOME/.bashrc
```

**Getting the code**

the python-swiftclient 

```bash
cd $HOME; git clone https://opendev.org/openstack/python-swiftclient.git
```
Build a development installation of python-swiftclient:

```bash
cd $HOME/python-swiftclient; sudo python3 setup.py develop; cd -
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/c75b1d77-2968-49f1-aab4-3617a66ae576)

clonning swift repo:

```bash
git clone https://github.com/openstack/swift.git
```
Build a development installation of Swift:


```bash
cd $HOME/swift; sudo pip install --no-binary cryptography -r requirements.txt; sudo python3 setup.py develop; cd -
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/71e09050-b51b-40d2-9cef-2ce5b7316b6a)

Install Swift’s test dependencies:


```bash
cd $HOME/swift; sudo pip install -r test-requirements.txt

```
**Setting up rsync**
Create /etc/rsyncd.conf:

```bash
sudo cp $HOME/swift/doc/saio/rsyncd.conf /etc/
sudo sed -i "s/<your-user-name>/${USER}/" /etc/rsyncd.conf
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/c121c486-8158-4ca6-916d-df2d3e731b07)

Here is the default rsyncd.conf file contents maintained in the repo that is copied and fixed up above:

```bash
uid = <your-user-name>
gid = <your-user-name>
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = 0.0.0.0

[account6212]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/account6212.lock

[account6222]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/account6222.lock

[account6232]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/account6232.lock

[account6242]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/account6242.lock

[container6211]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/container6211.lock

[container6221]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/container6221.lock

[container6231]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/container6231.lock

[container6241]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/container6241.lock

[object6210]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/object6210.lock

[object6220]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/object6220.lock

[object6230]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/object6230.lock

[object6240]
max connections = 25
path = /srv/4/node/
read only = false
lock file = /var/lock/object6240.lock
```
Start the rsync daemon

```bash
sudo systemctl enable rsync
sudo systemctl start rsync
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/c5a17c69-788f-4eeb-a87f-59a9186fbb06)

Verify rsync is accepting connections for all servers:
```bash
rsync rsync://pub@localhost/
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/22763e27-4505-4dad-9dee-2c16d38cd3c3)

**Starting memcached**

```bash
sudo systemctl enable memcached
sudo systemctl start memcached
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/482caca2-43d9-4977-a916-a148f97b5972)

**Configuring each node**

After performing the following steps, be sure to verify that Swift has access to resulting configuration files (sample configuration files are provided with all defaults in line-by-line comments).

Optionally remove an existing swift directory:
```bash
sudo rm -rf /etc/swift
```

Populate the /etc/swift directory itself:

```bash
cd $HOME/swift/doc; sudo cp -r saio/swift /etc/swift; cd -
sudo chown -R ${USER}:${USER} /etc/swift
```
Update <your-user-name> references in the Swift config files:

```bash
find /etc/swift/ -name \*.conf | xargs sudo sed -i "s/<your-user-name>/${USER}/"
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/6a49c1a4-77b1-41e0-ab38-84dd3fb0a0d0)

The contents of the configuration files provided by executing the above commands are as follows:


/etc/swift/swift.conf

```bash
[swift-hash]
# random unique strings that can never change (DO NOT LOSE)
# Use only printable chars (python -c "import string; print(string.printable)")
swift_hash_path_prefix = changeme
swift_hash_path_suffix = changeme

[storage-policy:0]
name = gold
policy_type = replication
default = yes

[storage-policy:1]
name = silver
policy_type = replication

[storage-policy:2]
name = ec42
policy_type = erasure_coding
ec_type = liberasurecode_rs_vand
ec_num_data_fragments = 4
ec_num_parity_fragments = 2
```
/etc/swift/proxy-server.conf

```bash
[DEFAULT]
bind_ip = 127.0.0.1
bind_port = 8080
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL1
eventlet_debug = true

[pipeline:main]
# Yes, proxy-logging appears twice. This is so that
# middleware-originated requests get logged too.
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache etag-quoter listing_formats bulk tempurl ratelimit crossdomain container_sync tempauth staticweb copy container-quotas account-quotas slo dlo versioned_writes symlink proxy-logging proxy-server

[filter:catch_errors]
use = egg:swift#catch_errors

[filter:healthcheck]
use = egg:swift#healthcheck

[filter:proxy-logging]
use = egg:swift#proxy_logging

[filter:bulk]
use = egg:swift#bulk

[filter:ratelimit]
use = egg:swift#ratelimit

[filter:crossdomain]
use = egg:swift#crossdomain

[filter:dlo]
use = egg:swift#dlo

[filter:slo]
use = egg:swift#slo

[filter:container_sync]
use = egg:swift#container_sync
current = //saio/saio_endpoint

[filter:tempurl]
use = egg:swift#tempurl

[filter:tempauth]
use = egg:swift#tempauth
user_admin_admin = admin .admin .reseller_admin
user_test_tester = testing .admin
user_test_tester2 = testing2 .admin
user_test_tester3 = testing3
user_test2_tester2 = testing2 .admin

[filter:staticweb]
use = egg:swift#staticweb

[filter:account-quotas]
use = egg:swift#account_quotas

[filter:container-quotas]
use = egg:swift#container_quotas

[filter:cache]
use = egg:swift#memcache

[filter:etag-quoter]
use = egg:swift#etag_quoter
enable_by_default = false

[filter:gatekeeper]
use = egg:swift#gatekeeper

[filter:versioned_writes]
use = egg:swift#versioned_writes
allow_versioned_writes = true
allow_object_versioning = true

[filter:copy]
use = egg:swift#copy

[filter:listing_formats]
use = egg:swift#listing_formats

[filter:domain_remap]
use = egg:swift#domain_remap

[filter:symlink]
use = egg:swift#symlink

# To enable, add the s3api middleware to the pipeline before tempauth
[filter:s3api]
use = egg:swift#s3api
s3_acl = yes
check_bucket_owner = yes
cors_preflight_allow_origin = *

# Example to create root secret: `openssl rand -base64 32`
[filter:keymaster]
use = egg:swift#keymaster
encryption_root_secret = changeme/changeme/changeme/changeme/change/=

# To enable use of encryption add both middlewares to pipeline, example:
# <other middleware> keymaster encryption proxy-logging proxy-server
[filter:encryption]
use = egg:swift#encryption

[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true
account_autocreate = true
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/51dcecbf-e185-4ad5-992e-a2c44570e851)

/etc/swift/object-expirer.conf

```bash
[DEFAULT]
# swift_dir = /etc/swift
user = <your-user-name>
# You can specify default log routing here if you want:
log_name = object-expirer
log_facility = LOG_LOCAL6
log_level = INFO
#log_address = /dev/log
#
# comma separated list of functions to call to setup custom log handlers.
# functions get passed: conf, name, log_to_console, log_route, fmt, logger,
# adapted_logger
# log_custom_handlers =
#
# If set, log_udp_host will override log_address
# log_udp_host =
# log_udp_port = 514
#
# You can enable StatsD logging here:
# log_statsd_host =
# log_statsd_port = 8125
# log_statsd_default_sample_rate = 1.0
# log_statsd_sample_rate_factor = 1.0
# log_statsd_metric_prefix =

[object-expirer]
interval = 300
# report_interval = 300
# concurrency is the level of concurrency to use to do the work, this value
# must be set to at least 1
# concurrency = 1
# processes is how many parts to divide the work into, one part per process
#   that will be doing the work
# processes set 0 means that a single process will be doing all the work
# processes can also be specified on the command line and will override the
#   config value
# processes = 0
# process is which of the parts a particular process will work on
# process can also be specified on the command line and will override the config
#   value
# process is "zero based", if you want to use 3 processes, you should run
#  processes with process set to 0, 1, and 2
# process = 0

[pipeline:main]
pipeline = catch_errors cache proxy-server

[app:proxy-server]
use = egg:swift#proxy
# See proxy-server.conf-sample for options

[filter:cache]
use = egg:swift#memcache
# See proxy-server.conf-sample for options

[filter:catch_errors]
use = egg:swift#catch_errors
# See proxy-server.conf-sample for options
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/f8032cf3-cf36-4ac7-aca0-481bc68b084c)

/etc/swift/container-sync-realms.conf

```bash
[saio]
key = changeme
key2 = changeme
cluster_saio_endpoint = http://127.0.0.1:8080/v1/
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/e0e954b4-6936-405d-891d-1ebb03793b07)


/etc/swift/account-server/1.conf

```bash
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.1
bind_port = 6212
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[account-replicator]
rsync_module = {replication_ip}::account{replication_port}

[account-auditor]

[account-reaper]
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/a744272a-38a5-4d3a-9f1b-ac083c537c31)

/etc/swift/container-server/1.conf

```bash
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.1
bind_port = 6211
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[container-replicator]
rsync_module = {replication_ip}::container{replication_port}

[container-updater]

[container-auditor]

[container-sync]

[container-sharder]
auto_shard = true
rsync_module = {replication_ip}::container{replication_port}
# This is intentionally much smaller than the default of 1,000,000 so tests
# can run in a reasonable amount of time
shard_container_threshold = 100
# The probe tests make explicit assumptions about the batch sizes
shard_scanner_batch_size = 10
cleave_batch_size = 2

```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/9caadf0a-fc68-4878-b125-4dad3ce67e17)

/etc/swift/container-reconciler/1.conf

```bash
[DEFAULT]
# swift_dir = /etc/swift
user = <your-user-name>
# You can specify default log routing here if you want:
# log_name = swift
log_facility = LOG_LOCAL2
# log_level = INFO
# log_address = /dev/log
#
# comma separated list of functions to call to setup custom log handlers.
# functions get passed: conf, name, log_to_console, log_route, fmt, logger,
# adapted_logger
# log_custom_handlers =
#
# If set, log_udp_host will override log_address
# log_udp_host =
# log_udp_port = 514
#
# You can enable StatsD logging here:
# log_statsd_host =
# log_statsd_port = 8125
# log_statsd_default_sample_rate = 1.0
# log_statsd_sample_rate_factor = 1.0
# log_statsd_metric_prefix =

[container-reconciler]
# reclaim_age = 604800
# interval = 300
# request_tries = 3
processes = 4
process = 0

[pipeline:main]
pipeline = catch_errors proxy-logging cache proxy-server

[app:proxy-server]
use = egg:swift#proxy
# See proxy-server.conf-sample for options

[filter:cache]
use = egg:swift#memcache
# See proxy-server.conf-sample for options

[filter:proxy-logging]
use = egg:swift#proxy_logging

[filter:catch_errors]
use = egg:swift#catch_errors
# See proxy-server.conf-sample for options
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/b0e116a1-e531-448e-bcd5-31e130758f5c)

/etc/swift/object-server/1.conf

```bash
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.1
bind_port = 6210
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL2
recon_cache_path = /var/cache/swift
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[object-replicator]
rsync_module = {replication_ip}::object{replication_port}

[object-reconstructor]

[object-updater]

[object-auditor]

[object-relinker]
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/6bc10ab3-3e9a-4cc0-a12d-9340a38f35a9)

/etc/swift/account-server/2.conf

```bash
[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.2
bind_port = 6222
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL3
recon_cache_path = /var/cache/swift2
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[account-replicator]
rsync_module = {replication_ip}::account{replication_port}

[account-auditor]

[account-reaper]
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/7c784d26-c1f0-4c71-8bc7-185311fe84b2)

/etc/swift/container-reconciler/2.conf

```bash
[DEFAULT]
# swift_dir = /etc/swift
user = <your-user-name>
# You can specify default log routing here if you want:
# log_name = swift
log_facility = LOG_LOCAL3
# log_level = INFO
# log_address = /dev/log
#
# comma separated list of functions to call to setup custom log handlers.
# functions get passed: conf, name, log_to_console, log_route, fmt, logger,
# adapted_logger
# log_custom_handlers =
#
# If set, log_udp_host will override log_address
# log_udp_host =
# log_udp_port = 514
#
# You can enable StatsD logging here:
# log_statsd_host =
# log_statsd_port = 8125
# log_statsd_default_sample_rate = 1.0
# log_statsd_sample_rate_factor = 1.0
# log_statsd_metric_prefix =

[container-reconciler]
# reclaim_age = 604800
# interval = 300
# request_tries = 3
processes = 4
process = 1

[pipeline:main]
pipeline = catch_errors proxy-logging cache proxy-server

[app:proxy-server]
use = egg:swift#proxy
# See proxy-server.conf-sample for options

[filter:cache]
use = egg:swift#memcache
# See proxy-server.conf-sample for options

[filter:proxy-logging]
use = egg:swift#proxy_logging

[filter:catch_errors]
use = egg:swift#catch_errors
# See proxy-server.conf-sample for options
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/299df35a-103c-4c63-94a1-eea5e7c71ef7)

/etc/swift/object-server/2.conf

```bash
[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.2
bind_port = 6220
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL3
recon_cache_path = /var/cache/swift2
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[object-replicator]
rsync_module = {replication_ip}::object{replication_port}

[object-reconstructor]

[object-updater]

[object-auditor]

[object-relinker]
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/f19ccb90-2f3d-4ab2-8140-92dc785aebff)

/etc/swift/account-server/3.conf

```bash
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.3
bind_port = 6232
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[account-replicator]
rsync_module = {replication_ip}::account{replication_port}

[account-auditor]

[account-reaper]
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/65398752-b27c-4e22-bac9-ffa02fa15f31)

/etc/swift/container-server/3.conf

```bash
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.3
bind_port = 6231
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[container-replicator]
rsync_module = {replication_ip}::container{replication_port}

[container-updater]

[container-auditor]

[container-sync]

[container-sharder]
auto_shard = true
rsync_module = {replication_ip}::container{replication_port}
# This is intentionally much smaller than the default of 1,000,000 so tests
# can run in a reasonable amount of time
shard_container_threshold = 100
# The probe tests make explicit assumptions about the batch sizes
shard_scanner_batch_size = 10
cleave_batch_size = 2
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/fa49d64a-9b82-4c52-97a8-e9248a21f694)

/etc/swift/container-reconciler/3.conf

```bash
[DEFAULT]
# swift_dir = /etc/swift
user = <your-user-name>
# You can specify default log routing here if you want:
# log_name = swift
log_facility = LOG_LOCAL4
# log_level = INFO
# log_address = /dev/log
#
# comma separated list of functions to call to setup custom log handlers.
# functions get passed: conf, name, log_to_console, log_route, fmt, logger,
# adapted_logger
# log_custom_handlers =
#
# If set, log_udp_host will override log_address
# log_udp_host =
# log_udp_port = 514
#
# You can enable StatsD logging here:
# log_statsd_host =
# log_statsd_port = 8125
# log_statsd_default_sample_rate = 1.0
# log_statsd_sample_rate_factor = 1.0
# log_statsd_metric_prefix =

[container-reconciler]
# reclaim_age = 604800
# interval = 300
# request_tries = 3
processes = 4
process = 2

[pipeline:main]
pipeline = catch_errors proxy-logging cache proxy-server

[app:proxy-server]
use = egg:swift#proxy
# See proxy-server.conf-sample for options

[filter:cache]
use = egg:swift#memcache
# See proxy-server.conf-sample for options

[filter:proxy-logging]
use = egg:swift#proxy_logging

[filter:catch_errors]
use = egg:swift#catch_errors
# See proxy-server.conf-sample for options
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/88198fed-841c-4a95-a967-8fc1e13e628c)

/etc/swift/object-server/3.conf

```bash
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.3
bind_port = 6230
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL4
recon_cache_path = /var/cache/swift3
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[object-replicator]
rsync_module = {replication_ip}::object{replication_port}

[object-reconstructor]

[object-updater]

[object-auditor]

[object-relinker]
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/df2a32f6-ab6f-48fe-b6f0-b78f3e132b2d)


/etc/swift/account-server/4.conf

```bash
[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.4
bind_port = 6242
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[account-replicator]
rsync_module = {replication_ip}::account{replication_port}

[account-auditor]

[account-reaper]
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/033c1a02-b2c7-41ea-b8f2-5d3271bd9564)

/etc/swift/container-server/4.conf

```bash
[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.4
bind_port = 6241
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[container-replicator]
rsync_module = {replication_ip}::container{replication_port}

[container-updater]

[container-auditor]

[container-sync]

[container-sharder]
auto_shard = true
rsync_module = {replication_ip}::container{replication_port}
# This is intentionally much smaller than the default of 1,000,000 so tests
# can run in a reasonable amount of time
shard_container_threshold = 100
# The probe tests make explicit assumptions about the batch sizes
shard_scanner_batch_size = 10
cleave_batch_size = 2
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/5f385e6e-15f8-4ac4-84bc-bbf6ab8d6131)

/etc/swift/container-reconciler/4.conf

```bash
[DEFAULT]
# swift_dir = /etc/swift
user = <your-user-name>
# You can specify default log routing here if you want:
# log_name = swift
log_facility = LOG_LOCAL5
# log_level = INFO
# log_address = /dev/log
#
# comma separated list of functions to call to setup custom log handlers.
# functions get passed: conf, name, log_to_console, log_route, fmt, logger,
# adapted_logger
# log_custom_handlers =
#
# If set, log_udp_host will override log_address
# log_udp_host =
# log_udp_port = 514
#
# You can enable StatsD logging here:
# log_statsd_host =
# log_statsd_port = 8125
# log_statsd_default_sample_rate = 1.0
# log_statsd_sample_rate_factor = 1.0
# log_statsd_metric_prefix =

[container-reconciler]
# reclaim_age = 604800
# interval = 300
# request_tries = 3
processes = 4
process = 3

[pipeline:main]
pipeline = catch_errors proxy-logging cache proxy-server

[app:proxy-server]
use = egg:swift#proxy
# See proxy-server.conf-sample for options

[filter:cache]
use = egg:swift#memcache
# See proxy-server.conf-sample for options

[filter:proxy-logging]
use = egg:swift#proxy_logging

[filter:catch_errors]
use = egg:swift#catch_errors
# See proxy-server.conf-sample for options
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/83ae3141-0f39-487a-b22c-c6bba45fe36d)

/etc/swift/object-server/4.conf

```bash
[DEFAULT]
devices = /srv/4/node
mount_check = false
disable_fallocate = true
bind_ip = 127.0.0.4
bind_port = 6240
workers = 1
user = <your-user-name>
log_facility = LOG_LOCAL5
recon_cache_path = /var/cache/swift4
eventlet_debug = true

[pipeline:main]
pipeline = healthcheck recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[filter:healthcheck]
use = egg:swift#healthcheck

[object-replicator]
rsync_module = {replication_ip}::object{replication_port}

[object-reconstructor]

[object-updater]

[object-auditor]

[object-relinker]
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/eaedade3-22ca-4f7f-ac5b-6cc7b276bfb2)

**Setting up scripts for running Swift**
Copy the SAIO scripts for resetting the environment:

```bash
mkdir -p $HOME/bin
cd $HOME/swift/doc; cp saio/bin/* $HOME/bin; cd -
chmod +x $HOME/bin/*
```
Edit the $HOME/bin/resetswift script

The template resetswift script looks like the following:

```bash
#!/bin/bash

set -e

swift-init all kill
swift-orphans -a 0 -k KILL

# Remove the following line if you did not set up rsyslog for individual logging:
sudo find /var/log/swift -type f -exec rm -f {} \;
if cut -d' ' -f2 /proc/mounts | grep -q /mnt/sdb1 ; then
    sudo umount /mnt/sdb1
fi
# If you are using a loopback device set SAIO_BLOCK_DEVICE to "/srv/swift-disk"
sudo mkfs.xfs -f ${SAIO_BLOCK_DEVICE:-/dev/sdb1}
sudo mount /mnt/sdb1
sudo mkdir /mnt/sdb1/1 /mnt/sdb1/2 /mnt/sdb1/3 /mnt/sdb1/4
sudo chown ${USER}:${USER} /mnt/sdb1/*
mkdir -p /srv/1/node/sdb1 /srv/1/node/sdb5 \
         /srv/2/node/sdb2 /srv/2/node/sdb6 \
         /srv/3/node/sdb3 /srv/3/node/sdb7 \
         /srv/4/node/sdb4 /srv/4/node/sdb8
sudo rm -f /var/log/debug /var/log/messages /var/log/rsyncd.log /var/log/syslog
find /var/cache/swift* -type f -name *.recon -exec rm -f {} \;
if [ "`type -t systemctl`" == "file" ]; then
    sudo systemctl restart rsyslog
    sudo systemctl restart memcached
else
    sudo service rsyslog restart
    sudo service memcached restart
fi
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/261caae8-9d28-4755-825d-c2aab98305a8)

If you did not set up rsyslog for individual logging, remove the find /var/log/swift... line:

```bash
sed -i "/find \/var\/log\/swift/d" $HOME/bin/resetswift
```
Install the sample configuration file for running tests:

```bash
cp $HOME/swift/test/sample.conf /etc/swift/test.conf
```

The template test.conf looks like the following:

```bash
[s3api_test]
# You just enable advanced compatibility features to pass all tests.  Add the
# following non-default options to the s3api section of your proxy-server.conf
# s3_acl = True
# check_bucket_owner = True
endpoint = http://127.0.0.1:8080
#ca_cert=/path/to/ca.crt
region = us-east-1
# First and second users should be account owners
access_key1 = test:tester
secret_key1 = testing
access_key2 = test:tester2
secret_key2 = testing2
# Third user should be unprivileged
access_key3 = test:tester3
secret_key3 = testing3

[func_test]
# Sample config for Swift with tempauth
auth_uri = http://127.0.0.1:8080/auth/v1.0
# Sample config for Swift with Keystone v2 API.
# For keystone v2 change auth_version to 2 and auth_prefix to /v2.0/.
# And "allow_account_management" should not be set "true".
#auth_version = 3
#auth_uri = http://localhost:5000/v3/

# Used by s3api functional tests, which don't contact auth directly
#s3_storage_url = http://127.0.0.1:8080/
#s3_region = us-east-1

# Primary functional test account (needs admin access to the account)
account = test
username = tester
password = testing
s3_access_key = test:tester
s3_secret_key = testing

# User on a second account (needs admin access to the account)
account2 = test2
username2 = tester2
password2 = testing2

# User on same account as first, but without admin access
username3 = tester3
password3 = testing3
# s3api requires the same account with the primary one and different users
# one swift owner:
s3_access_key2 = test:tester2
s3_secret_key2 = testing2
# one unprivileged:
s3_access_key3 = test:tester3
s3_secret_key3 = testing3

# Fourth user is required for keystone v3 specific tests.
# Account must be in a non-default domain.
#account4 = test4
#username4 = tester4
#password4 = testing4
#domain4 = test-domain

# Fifth user is required for service token-specific tests.
# The account must be different from the primary test account.
# The user must not have a group (tempauth) or role (keystoneauth) on
# the primary test account. The user must have a group/role that is unique
# and not given to the primary tester and is specified in the options
# <prefix>_require_group (tempauth) or <prefix>_service_roles (keystoneauth).
#account5 = test5
#username5 = tester5
#password5 = testing5

# The service_prefix option is used for service token-specific tests.
# If service_prefix or username5 above is not supplied, the tests are skipped.
# To set the value and enable the service token tests, look at the
# reseller_prefix option in /etc/swift/proxy-server.conf. There must be at
# least two prefixes. If not, add a prefix as follows (where we add SERVICE):
#     reseller_prefix = AUTH, SERVICE
# The service_prefix must match the <prefix> used in <prefix>_require_group
# (tempauth) or <prefix>_service_roles (keystoneauth); for example:
#    SERVICE_require_group = service
#    SERVICE_service_roles = service
# Note: Do not enable service token tests if the first prefix in
# reseller_prefix is the empty prefix AND the primary functional test
# account contains an underscore.
#service_prefix = SERVICE

# Sixth user is required for access control tests.
# Account must have a role for reseller_admin_role(keystoneauth).
#account6 = test
#username6 = tester6
#password6 = testing6

collate = C

# Only necessary if a pre-existing server uses self-signed certificate
insecure = no

# Tests that are dependent on domain_remap middleware being installed also
# require one of the domain_remap storage_domain values to be specified here,
# otherwise those tests will be skipped.
storage_domain =

[unit_test]
fake_syslog = False

[probe_test]
# check_server_timeout = 30
# validate_rsync = false
# proxy_base_url = http://localhost:8080

[swift-constraints]
# The functional test runner will try to use the constraint values provided in
# the swift-constraints section of test.conf.
#
# If a constraint value does not exist in that section, or because the
# swift-constraints section does not exist, the constraints values found in
# the /info API call (if successful) will be used.
#
# If a constraint value cannot be found in the /info results, either because
# the /info API call failed, or a value is not present, the constraint value
# used will fall back to those loaded by the constraints module at time of
# import (which will attempt to load /etc/swift/swift.conf, see the
# swift.common.constraints module for more information).
#
# Note that the cluster must have "sane" values for the test suite to pass
# (for some definition of sane).
#
#max_file_size = 5368709122
#max_meta_name_length = 128
#max_meta_value_length = 256
#max_meta_count = 90
#max_meta_overall_size = 4096
#max_header_size = 8192
#extra_header_count = 0
#max_object_name_length = 1024
#container_listing_limit = 10000
#account_listing_limit = 10000
#max_account_name_length = 256
#max_container_name_length = 256

# Newer swift versions default to strict cors mode, but older ones were the
# opposite.
#strict_cors_mode = true
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/c665b63d-4639-4d6c-b184-5cc1a6ee26a9)

**Configure environment variables for Swift**
Add an environment variable for running tests below:

```bash
echo "export SWIFT_TEST_CONFIG_FILE=/etc/swift/test.conf" >> $HOME/.bashrc
```
Be sure that your PATH includes the bin directory:

```bash
echo "export PATH=${PATH}:$HOME/bin" >> $HOME/.bashrc
```
If you are using a loopback device for Swift Storage, add an environment var to substitute /dev/sdb1 with /srv/swift-disk:

```bash
echo "export SAIO_BLOCK_DEVICE=/srv/swift-disk" >> $HOME/.bashrc
```
If you are using a device other than /dev/sdb1 for Swift storage (for example, /dev/vdb1), add an environment var to substitute it:

```bash
echo "export SAIO_BLOCK_DEVICE=/dev/vdb1" >> $HOME/.bashrc
```
If you are using a location other than /tmp for Swift tmp data (for example, /mnt/tmp), add TMPDIR environment var to set it:

```bash
export TMPDIR=/mnt/tmp
echo "export TMPDIR=/mnt/tmp" >> $HOME/.bashrc
```
Source the above environment variables into your current environment:

```bash
. $HOME/.bashrc
```

**Constructing initial rings**
Construct the initial rings using the provided script:

```bash
remakerings
```
**Testing Swift**
Verify the unit tests run:
```bash
$HOME/swift/.unittests
```
Start the “main” Swift daemon processes (proxy, account, container, and object):

```bash
startmain
```

![image](https://github.com/s4ki3f/openstackswift/assets/29111757/ddcc9225-59b4-4ad1-ae56-fc144f9fa2f6)

if startmain comes up with error then change permission and it will look like this
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/e96d855e-3c31-4e32-ac0a-f7379c098248)


Get an X-Storage-Url and X-Auth-Token:

```bash
curl -v -H 'X-Storage-User: test:tester' -H 'X-Storage-Pass: testing' http://127.0.0.1:8080/auth/v1.0
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/6fa24bb7-20dd-472e-8ac1-3d04f6f2c05e)

Check that you can GET account:

```bash
curl -v -H 'X-Auth-Token: <token-from-x-auth-token-above>' <url-from-x-storage-url-above>
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/797a3a8a-7963-4664-837f-2b3caaf27da7)

Check that swift command provided by the python-swiftclient package works:

```bash
swift -A http://127.0.0.1:8080/auth/v1.0 -U test:tester -K testing stat
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/58856258-da7d-41a1-8d6e-4a8b71fffa6e)
