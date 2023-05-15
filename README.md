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
