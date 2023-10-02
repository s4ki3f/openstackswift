Installing the latest OpenStack Swift doc

System Info

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

