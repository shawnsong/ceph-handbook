# Ceph Handbook

## Ceph Cluster Environment Setup
`ceph-deploy` will be used to create a 3 node cluster in this handbook. The IP addresses of the cluster is as below:

| Hostname      | IP Address     |
|---------------|----------------|
| ceph-admin    | 192.168.1.60   |
| node3         | 192.168.1.61   |
| node4         | 192.168.1.62   |
| ceph-mon-node | 192.168.1.63   |

Ceph admin node:
```shell
$ hostname
ceph-admin

$ cat /etc/hosts
192.168.1.60    ceph-admin
192.168.1.61    node3
192.168.1.62    node4
192.168.1.63    ceph-mon
```

Ceph monitor node:
```shell
$ hostname
ceph-mon

$ cat /etc/hosts
192.168.1.60    ceph-admin
192.168.1.61    node3
192.168.1.62    node4
```

Node 3:
```shell
$ hostname
node3

$ cat /etc/hosts
192.168.1.60    ceph-admin
192.168.1.51    node1
192.168.1.52    node2
192.168.1.62    node4
192.168.1.63    ceph-mon
```

Node 4:
```shell
$ hostname
node4

$ cat /etc/hosts
192.168.1.60    ceph-admin
192.168.1.51    node1
192.168.1.52    node2
192.168.1.61    node3
192.168.1.63    ceph-mon
```

> Note: node3 and node4 have `node1` and `node2` in their `/etc/hosts` file because these two nodes are setup as Kubernetes minions and storage solutions for the cluster. Please refer to my [kubernetes-handbook](https://github.com/shawnsong/kubernetes-handbook) for more details.

### Setup Admin(Deployer) Node

Install `ceph-deploy` on the admin node. Replace `{ceph-stable-release}` to a stable release. The latest stable release is mimic by the time when this is written.

```shell
$ cat << EOM > /etc/yum.repos.d/ceph.repo
[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-{ceph-stable-release}/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
EOM

$ sudo yum update
$ sudo yum install ceph-deploy
```


### Install Ceph Monitor


### Install Ceph Node