# Ceph Handbook

For a quick start, we will use ceph-deploy tool to create a basic cluster with 1 node Ceph Monitor and 2 nodes Ceph OSDs. 

## Environment Preparation

ceph-deploy will be used to create a 3 node cluster in this handbook. The IP addresses of the cluster is as below:

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

If Ceph was installed before, we can use below command to clean up the previous setup and start over again.

```shell
# remove previously installed ceph binarary
$ ceph-deploy purge node1 node2
# remove ceph node data
$ ceph-deploy purgedata node1 node2
# remove keys
$ ceph-deploy forgetkeys
# remove current directory ceph-deploy generated files
$ rm ceph.*
```

### Setup the Admin(Deployer) Node

Install ceph-deploy on the admin node. Replace `{ceph-stable-release}` to a stable release. The latest stable release is mimic by the time when this is written.

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
A super user is required to setup on all nodes for ceph-deploy to use. This super user needs to be granted permissions to run `sudo` without typing the password because ceph-deploy does not prompt for password during the installation. The username of the super user does not matter, as long as it is the same user across all nodes. I will use `cephd` for this handbook.

Add `cephd` user on all nodes (admin, ceph-mon, node3 and node4):
```shell
# Add user cephd
$ useradd -d /home/cephd -m cephd
# Set password for cephd
$ passwd cephd

# Add no password sudo permission
$ echo "cephd ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephd
$ sudo chmod 0440 /etc/sudoers.d/cephd
```

On admin node, switch the current user to `cephd` and generate the ssh key for this user:
```shell
$ su - cephd

$ ssh-keygen
Generating public/private rsa key pair.
...
```
Copy the key pair to the monitor and osd nodes. This step requires password.

```shell
$ ssh-copy-id cephd@ceph-mon
$ ssh-copy-id cephd@node3
$ ssh-copy-id cephd@node4
```

After the ssh key is copied to all nodes, try to ssh to each node. This time, no password should be required. If it still requires password, please check if the above steps are done correctly.

### Install Ceph Node

Now, we have a clean environment ready for the cluster. ceph-deploy will generate all necessary keys and configuration files for the cluster in the directory where ceph-deploy is invoked. Hence, to keep the files organised, let's create a new folder called `ceph-cluster` and enter this folder.

The first step is to generate the configuration files.  

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
`ceph-mon` in the command is the hostname of the Ceph Monitor node. Also, we need to add `osd pool default size = 2` to `ceph.conf` file because we have 2 OSD nodes in the cluster. 

Now, install Ceph on all nodes. 

```shell
$ ceph-deploy install nod3 node4 ceph-mon
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
...
[ceph-mon][DEBUG ] Complete!
[ceph-mon][INFO  ] Running command: sudo ceph --version
[ceph-mon][DEBUG ] ceph version 13.2.4 (b10be4d44915a4d78a8e06aa31919e74927b142e) mimic (stable)
```

Please make sure all nodes (node3, node4, ceph-mon) have ceph installed correctly. This can be done by checking the ceph version. For example, on node3: 
```shell
$ ssh node3
$ ceph --version
ceph version 13.2.4 (b10be4d44915a4d78a8e06aa31919e74927b142e) mimic (stable)
```

### Install Ceph Monitor (mon)

```shell
$ ceph-deploy mon create-initial
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephd/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /bin/ceph-deploy mon create-initial
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create-initial
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f25174d5128>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7f25179437d0>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  keyrings                      : None
...
[ceph-mon][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --admin-daemon=/var/run/ceph/ceph-mon.ceph-mon.asok mon_status
[ceph-mon][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-mon/keyring auth get client.admin
[ceph-mon][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-mon/keyring auth get client.bootstrap-mds
[ceph-mon][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-mon/keyring auth get client.bootstrap-mgr
[ceph-mon][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-mon/keyring auth get client.bootstrap-osd
[ceph-mon][INFO  ] Running command: sudo /usr/bin/ceph --connect-timeout=25 --cluster=ceph --name mon. --keyring=/var/lib/ceph/mon/ceph-ceph-mon/keyring auth get client.bootstrap-rgw
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.client.admin.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mds.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mgr.keyring
[ceph_deploy.gatherkeys][INFO  ] keyring 'ceph.mon.keyring' already exists
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-osd.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-rgw.keyring
[ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/tmpn_x5ZR
```

After this step, we should see more files are created in the current directory.
```shell
$ ls -l
-rw------- 1 cephd cephd    113 Jan 25 21:48 ceph.bootstrap-mds.keyring
-rw------- 1 cephd cephd    113 Jan 25 21:48 ceph.bootstrap-mgr.keyring
-rw------- 1 cephd cephd    113 Jan 25 21:48 ceph.bootstrap-osd.keyring
-rw------- 1 cephd cephd    113 Jan 25 21:48 ceph.bootstrap-rgw.keyring
-rw------- 1 cephd cephd    151 Jan 25 21:48 ceph.client.admin.keyring
-rw-rw-r-- 1 cephd cephd    224 Jan 24 21:57 ceph.conf
-rw-rw-r-- 1 cephd cephd 172588 Jan 25 21:48 ceph-deploy-ceph.log
-rw------- 1 cephd cephd     73 Jan 24 21:56 ceph.mon.keyring
```

And if we ssh to `ceph-mon` node, we should see that the monitor is already running.
```shell
$ ssh ceph-mon
$ ps -ef | grep ceph
ceph       14152       1  0 21:47 ?        00:00:00 /usr/bin/ceph-mon -f --cluster ceph --id ceph-mon --setuser ceph --setgroup ceph
```

The Ceph monitor is setup as a `systemd` daemon service. To stop it, use `systemctl stop ceph-mon@ceph-mon.service`.

> Note: If you are using Debian distros Linux, use `service stop` command. Also, the service name may vary between different versions of Ceph so you need to figure out the exact service name by youself.

### Distribute the Keys
Use ceph-deploy to copy the configuration file and admin key to your admin node and your Ceph Nodes so that you can use the ceph CLI without having to specify the monitor address and ceph.client.admin.keyring each time you execute a command.

```shell
$ ceph-deploy admin ceph-mon node3 node4
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephd/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /bin/ceph-deploy admin ceph-mon node3 node4
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fa6cd4ff368>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  client                        : ['ceph-mon', 'node3', 'node4']
[ceph_deploy.cli][INFO  ]  func                          : <function admin at 0x7fa6cdda15f0>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-mon
[ceph-mon][DEBUG ] connection detected need for sudo
[ceph-mon][DEBUG ] connected to host: ceph-mon
[ceph-mon][DEBUG ] detect platform information from remote host
[ceph-mon][DEBUG ] detect machine type
[ceph-mon][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to node3
[node3][DEBUG ] connection detected need for sudo
[node3][DEBUG ] connected to host: node3
[node3][DEBUG ] detect platform information from remote host
[node3][DEBUG ] detect machine type
[node3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to node4
[node4][DEBUG ] connection detected need for sudo
[node4][DEBUG ] connected to host: node4
[node4][DEBUG ] detect platform information from remote host
[node4][DEBUG ] detect machine type
[node4][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
```

### Install Ceph Manager (mgr)

From Ceph Luminous (12.x) release, the Ceph Manager daemon is required to run alongside monitor daemons to provide additional monitoring services.

```shell
$ ceph-deploy mgr create ceph-mon
# output
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephd/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /bin/ceph-deploy mgr create ceph-mon
[ceph_deploy.cli][INFO  ] ceph-deploy options:
...
[ceph-mon][INFO  ] Running command: sudo ceph --cluster ceph --name client.bootstrap-mgr --keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring auth get-or-create mgr.ceph-mon mon allow profile mgr osd allow * mds allow * -o /var/lib/ceph/mgr/ceph-ceph-mon/keyring
[ceph-mon][INFO  ] Running command: sudo systemctl enable ceph-mgr@ceph-mon
[ceph-mon][INFO  ] Running command: sudo systemctl start ceph-mgr@ceph-mon
[ceph-mon][INFO  ] Running command: sudo systemctl enable ceph.target
```

From the log, we can see `ceph-mgr@ceph-mon` has been created and started. Check its running status on `ceph-mon` node:

```shell
ceph       15394  0.0  3.9 486588 39808 ?        Ssl  10:58   0:03 /usr/bin/ceph-mon -f --cluster ceph --id ceph-mon --setuser ceph --setgroup ceph
ceph       15793  0.4  8.3 712972 84556 ?        Ssl  12:04   0:04 /usr/bin/ceph-mgr -f --cluster ceph --id ceph-mon --setuser ceph --setgroup ceph
```

### Install Ceph OSD Nodes

So far, we have successfully installed Ceph Monitor and Manager. The next step is to install OSDs, the actual storage nodes. Before this can be done, it is required that an empty disk is installed for the storage. We can use `fdisk -l` to list all disks and check their size etc. In my environment, the disks on node3 and node4 are called `/dev/sdb`.

```shell
$ ceph-deploy osd create --data /dev/sdb node3
# output
...
[ceph_deploy.osd][DEBUG ] Host node3 is now ready for osd use.

$ ceph-deploy osd create --data /dev/sdb node4
...
[ceph_deploy.osd][DEBUG ] Host node4 is now ready for osd use.
```
And ssh to the node to double check the status of the node.
```shell
$ ssh node3
$ sudo ceph health
HEALTH_OK
```

### Add a Ceph Metadata Server (MDS) - Optional

Ceph Metadata Servers is only required by Ceph File System. Ceph Block Devices and Ceph Object Storage do not use MDS. A Ceph Metadata Server (ceph-mds) stores metadata on behalf of the Ceph Filesystem. Ceph Metadata Servers allow POSIX file system users to execute basic commands (like ls, find, etc.) without placing an enormous burden on the Ceph Storage Cluster.

```shell
$ ceph-deploy mds create ceph-mon
```

### ADD an Object Gateway

Ceph Object Gateway is an object storage interface that provides a RESTful gateway to Ceph Storage Clusters. 

### Test the Cluster 

To store object data in Ceph, the client must:
1. Specify a pool
2. Set an object name

Below is a sample to save `testfile.txt` which contains text of `Test data"`.
```
# create test data
$ echo "Test data" > testfile.txt
# create a pool
$ ceph osd pool create mytest 8
# put the data in the pool
$ rados put test-object-1 testfile.txt --pool=mytest
```

To verify the data is saved:
```shell
$ sudo rados -p mytest ls
```

Locate the object:
```shell
$ sudo ceph osd map mytest test-object-1
```

And read the object into a file:
```shell
$ sudo rados -p mytest get test-object-1 out.txt
$ cat out.txt
Test data
```
