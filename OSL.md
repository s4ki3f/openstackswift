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

