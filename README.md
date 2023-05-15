# openstackswift installation in ubuntu 18
First We need to **Installing dependencies**
1. Hit super user

```bash
sudo su
apt-get update -y
apt-get install curl gcc memcached rsync sqlite3 xfsprogs \
                     git-core libffi-dev python-setuptools \
                     liberasurecode-dev libssl-dev
 ```             
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/242b0d3d-691e-45cd-b02e-34278aa12b98)

```bash
apt-get install python-coverage python-dev python-nose \
                     python-xattr python-eventlet \
                     python-greenlet python-pastedeploy \
                     python-netifaces python-pip python-dnspython \
                     python-mock
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/3460b4a8-0adc-477a-8537-9d2fedd570db)

Then, **Git Cloning swift python client**
```bash
sudo apt-get update
git clone https://github.com/openstack/python-swiftclient.git


```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/d346a90b-6bf4-4003-bd16-1a9e1720357f)

git checkout is optional but good thing to check
```bash
git checkout stable/train  
pip install -r requirements.txt 
python setup.py install

```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/c8ebcce5-8d5c-4c48-908c-efc35a6cab8c)
**Cloning Swift**
```bash
git clone https://github.com/openstack/swift.git
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/2ccf582b-b7be-487f-8622-ca4e21b8f061)

```bash
git checkout stable/train
python setup.py install
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/48990406-e421-408c-95ca-9d44a94f4bcc)

**Copying Swift Configuration Files**

```bash
mkdir -p /etc/swift 
cd swift/etc
cp account-server.conf-sample /etc/swift/account-server.conf 
cp container-server.conf-sample /etc/swift/container-server.conf
cp object-server.conf-sample /etc/swift/object-server.conf
cp proxy-server.conf-sample /etc/swift/proxy-server.conf 
cp drive-audit.conf-sample /etc/swift/drive-audit.conf 
cp swift.conf-sample /etc/swift/swift.conf

```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/9225488f-c362-4caa-a751-efdf71fcbed0)

```bash
swift-init -h
```
**Here I needed to install swift**
```bash
apt install python-swift
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/f4522ded-53a9-45b1-91c4-d58a838800d5)

**Adding Drives to swift**
checking blocks
```bash
ls /sys/block
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/7a64710a-5c73-44d0-bd4c-658b0f3be539)
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/ea3854d6-b0c9-40bd-a240-0ea9d0e8d00d)
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/fc441aed-c158-405f-b4cd-755d5cb70ada)

```bash
mkdir -p /srv/node/d1
mkdir -p /srv/node/d2
mkdir -p /srv/node/d3
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/86311b53-d785-4061-bde5-27040afb0409)

**Add User in Swift**
create user swift then add
```bash
chown -R swift:swift /srv/node
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/4e1e1011-8d11-4d9e-8ad9-d50e7e79ffc3)

**Mount**
```bash
cd swift/bin
nano mount_devices.sh
cat mount_devices.sh
#!/bin/bash
sudo mount -t xfs -o noatime,nodiratime,logbufs=8 -L d1 /srv/node/d1
sudo mount -t xfs -o noatime,nodiratime,logbufs=8 -L d2 /srv/node/d2
sudo mount -t xfs -o noatime,nodiratime,logbufs=8 -L d3 /srv/node/d3
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/5b073ef9-c80f-45d3-94a9-7141b41ad96e)
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/c303d2c2-d040-4181-b7f9-8461f362046a)
**Reboot**
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/a88fe376-d162-45ba-879e-70fe2b565d5a)

check everything mounted properly or not
```bash
cat /etc/mtab | grep /dev/sd
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/62276700-a688-4486-8c58-62c6e6a162ec)

**Create Swift Ring-file**
```bash
cd /etc/swift
swift-ring-builder account.builder create 3 3 1
swift-ring-builder container.builder create 3 3 1
swift-ring-builder object.builder create 3 3 1
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/ceba1569-cd54-4dbb-b6fc-eb7f5c2e577b)

```bash
swift-ring-builder account.builder add r1z1-127.0.0.1:6002/d1 100
swift-ring-builder container.builder add r1z1-127.0.0.1:6001/d1 100
swift-ring-builder object.builder add r1z1-127.0.0.1:6000/d1 100

swift-ring-builder account.builder add r1z1-127.0.0.1:6002/d2 100
swift-ring-builder container.builder add r1z1-127.0.0.1:6001/d2 100
swift-ring-builder object.builder add r1z1-127.0.0.1:6000/d2 100

swift-ring-builder account.builder add r1z1-127.0.0.1:6002/d3 100
swift-ring-builder container.builder add r1z1-127.0.0.1:6001/d3 100
swift-ring-builder object.builder add r1z1-127.0.0.1:6000/d3 100
```
**Balancing the rings**
```bash
swift-ring-builder account.builder rebalance
swift-ring-builder container.builder rebalance
swift-ring-builder object.builder rebalance
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/440ce53e-3f1a-471e-ae99-ef716412844a)
check everything with
```bash
ls
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/eb4fe58a-cfdc-4c75-97c3-a2f27b5e7390)

**Changing the binding ports**
account
```bash
swift-ring-builder account.builder
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/e760865e-46b6-40eb-b57a-54baf92e1658)

```bash
nano account-server.conf
bind_port : 6002
```
container
```bash
swift-ring-builder container.builder
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/a5e8f8ff-1613-4bef-be4e-0cf47bddf979)
```bash
nano container-server.conf
bind_port : 6001
```
object

```bash
swift-ring-builder object.builder
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/70484048-2fd1-4df5-9726-73c532b49bdc)
```bash
nano object-server.conf
bind_port : 6000
```
bind port for proxy server
```bash
nano proxy-server.conf
Bind_port : 8080
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/c4a2d814-a839-4916-b9ae-d15edb6415b0)
Then change swift conf
```bash
cd /etc
cd rsyslog.d
nano 0-swift.conf
local0.* /var/log/swift/all.log
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/b5bad0ea-4c64-466b-9f20-6738573088d2)
 
 ```bash
cd /etc/rsyslog.d
mkdir /var/log/swift
chown -R syslog.adm /var/log/swift
chmod -R g+w /var/log/swift
service rsyslog restart
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/7dcca869-e290-44c3-95b3-3e1b47ffc1d9)
changing hash

```bash
cd /etc/swift
nano swift.conf
swift_hash_path_suffix = RzUfDdu32L7J2ZBDYgsD6YI3Xie7hTVO8/oaQbpTbI8=
swift_hash_path_prefix = OZ1uQJNjJzTuFaM8X3v%fsJ1iR#F8wJjf9uhRiABevQ4
```
![image](https://github.com/s4ki3f/openstackswift/assets/29111757/475b35fe-a1e7-465d-b025-838842c5fcd5)

```bash
cd
cd ~/python-swiftclient
swift-init proxy restar
```

























