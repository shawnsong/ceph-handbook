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

### Setup Super User
A super user is required to setup on all nodes for ceph-deploy to use. This super user needs to be granted permission to run `sudo` without typing the password. The username does not matter, as long as it is the same user across all nodes. I will use `cephd` for this tutorial.

Add `cephd` user on all nodes (admin, ceph-mon, node3 and node4):
```shell
# Add cephd user
$ useradd -d /home/cephd -m cephd
# Set password for cephd
$ passwd cephd

# Add sudo permission
$ echo "cephd ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephd
$ sudo chmod 0440 /etc/sudoers.d/cephd
```

On admin node, switch user to `cephd`:
```shell
$ su - cephd

$ ssh-keygen
Generating public/private rsa key pair.
...
```
Copy the key pair to monitor and osd nodes. This step requires password.

```shell
$ ssh-copy-id cephd@ceph-mon
$ ssh-copy-id cephd@node3
$ ssh-copy-id cephd@node4
```

After the ssh key is copied to all nodes, try to ssh to each node. This time, no password should be required. If it still requires password, please check if each step is done correctly. Otherwise, ceph-deploy will not work correctly.

### Install Ceph Node
If Ceph was installed before, use below command to clean the previous setup.
```shell
$ ceph-deploy purge node1 node2
$ ceph-deploy purgedata node1 node2
$ ceph-deploy forgetkeys
```

Now, we have a clean environment for the cluster. Create a new folder `ceph-cluster` and enter this folder.

```shell
$ ceph-deploy new ceph-mon
...
$ ls
ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring
$ cat ceph.conf
[global]
fsid = 8eaaeb92-6c52-4691-8e2e-20939a4e6e34
mon_initial_members = ceph-mon
mon_host = 192.168.1.63
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```
Add `osd pool default size = 2` to `ceph.conf` file to indicate there are 2 OSD nodes in the cluster. Then deploy node3 and node4: 

```shell
$ ceph-deploy install nod1 node2
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephd/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /bin/ceph-deploy install node3 node4
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  verbose                       : False
...
[node3][DEBUG ] Complete!
[node3][INFO  ] Running command: sudo ceph --version
[node3][DEBUG ] ceph version 13.2.4 (b10be4d44915a4d78a8e06aa31919e74927b142e) mimic (stable)
[ceph_deploy.install][DEBUG ] Detecting platform for host node4 ...
[node4][DEBUG ] connection detected need for sudo
[node4][DEBUG ] connected to host: node4
...
[node4][DEBUG ] Complete!
[node4][INFO  ] Running command: sudo ceph --version
[node4][DEBUG ] ceph version 13.2.4 (b10be4d44915a4d78a8e06aa31919e74927b142e) mimic (stable)
```

### Install Ceph Monitor