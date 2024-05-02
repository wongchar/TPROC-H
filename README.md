# TPROC-H Benchmark using MySQL Database

## Setup NVMe Drives ##

Ensure NVME drives are balanced across NUMAs


Format all NVMes for the ext4 partition using the following command:
```
amd@amdbergamo-d4c2:~$ sudo mkfs.ext4 /dev/nvme0n1
mke2fs 1.45.5 (07-Jan-2020)
Discarding device blocks: done
Creating filesystem with 2335389696 4k blocks and 291926016 inodes
Filesystem UUID: 17f666ff-3284-4ae8-b034-c1bf02d25daf
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848, 512000000, 550731776, 644972544, 1934917632

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

Create mount points for each LV:
```
amd@amdbergamo-d4c2:~$ sudo mkdir /mnt1
amd@amdbergamo-d4c2:~$ sudo mkdir /mnt2
....

```

Add the mount points to the /etc/fstab file as follows:
```
amd@amdbergamo-d4c2:~$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/nvme5n1p2 during curtin installation
/dev/disk/by-uuid/980543b3-4a66-44f5-a2c1-9c19a28d353b / ext4 defaults 0 0
# /boot/efi was on /dev/nvme5n1p1 during curtin installation
/dev/disk/by-uuid/5274-4A80 /boot/efi vfat defaults 0 0
#/swap.img      none    swap    sw      0       0
/dev/nvme1n1 /mnt1 ext4 defaults 1 2
/dev/nvme2n1 /mnt2 ext4 defaults 1 2
...
```

Mount the drives:
```
amd@amdbergamo-d4c2:~$ sudo mount /mnt1
amd@amdbergamo-d4c2:~$ sudo mount /mnt2
...
```

Confirm the mount points:
```
lsblk
```


- - - -

## Install Kubernetes with ContainerD ##

Load br_netfilter module
```console
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

Allow iptables to see bridged traffic
```console
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply settings
```console
sudo sysctl --system
```

Install ContainerD

```console
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update -y 
sudo apt install -y containerd.io
```

Setup default config file for ContainerD
```console
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Set cgroupDriver to systemd in configuration file
```console
sudo vi /etc/containerd/config.toml
```

Change **SystemdCgroup = false** to **SystemdCgroup = true** \
Ensure your section matches the following:

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    BinaryName = ""
    CriuImagePath = ""
    CriuPath = ""
    CriuWorkPath = ""
    IoGid = 0
    IoUid = 0
    NoNewKeyring = false
    NoPivotRoot = false
    Root = ""
    ShimCgroup = ""
    SystemdCgroup = true
```

Restart ContainerD
```
sudo systemctl restart containerd
```

Install Kubernetes
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
```

Add hostnames and IP address of ALL nodes that will be on the cluster to **/etc/hosts** file for all nodes \
(Skip this step for Single-Node Cluster)
```
sudo vi /etc/hosts
```

An example of a host file for a 2-node cluster below:

```
127.0.0.1 localhost
10.216.179.66 ubuntu2004-milan-001
10.216.177.81 titanite-d4c2-os

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Disable swap on every node
```
sudo swapoff --a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Initialze control plane (make sure to record the output of kubeadm init if you want to add nodes to your K8s cluster)
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configure kubectl
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Set up CNI using Calico
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/custom-resources.yaml
```

Verify all pods are running
 ```
kubectl get pods -A
```

Output should look like the following
```console
amd@titanite-d4c2-os:~$ kubectl get pods -A
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-76b945d94f-2xnqj          1/1     Running   0          22s
calico-apiserver   calico-apiserver-76b945d94f-dm24c          1/1     Running   0          22s
calico-system      calico-kube-controllers-6b7b9c649d-wrbjj   1/1     Running   0          55s
calico-system      calico-node-v8xhj                          1/1     Running   0          55s
calico-system      calico-typha-7ff787f77b-6vwmp              1/1     Running   0          55s
calico-system      csi-node-driver-d5qtc                      2/2     Running   0          55s
kube-system        coredns-787d4945fb-7m5pz                   1/1     Running   0          87s
kube-system        coredns-787d4945fb-fr5d8                   1/1     Running   0          87s
kube-system        etcd-titanite-d4c2-os                      1/1     Running   0          102s
kube-system        kube-apiserver-titanite-d4c2-os            1/1     Running   0          101s
kube-system        kube-controller-manager-titanite-d4c2-os   1/1     Running   0          100s
kube-system        kube-proxy-c2rqb                           1/1     Running   0          87s
kube-system        kube-scheduler-titanite-d4c2-os            1/1     Running   0          101s
tigera-operator    tigera-operator-54b47459dd-fq8kk           1/1     Running   0          67s
```

OPTION: If you want to schedule pods on the control-plane, use the following command \ 
Run this step if you are using a single node
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

OPTION: If creating a multi-node cluster, use output command provided by kubeadm init on another node to add to the K8s cluster. \
Below is an example of join token for a worker node (NOTE: USE THE ONE PROVIDED BY YOUR KUBEADM INIT AND NOT THE EXAMPLE)
```
sudo kubeadm join 10.216.177.81:6443 --token pxskra.4lurssigp18i3h4v \
        --discovery-token-ca-cert-hash sha256:af6d5360b3874f31db97c8c0bc749821c17c65003a72b2c95a1dd6d0ceabd4f
```

- - - -

## Install Docker CE on Client/Load Generator ##
https://docs.docker.com/engine/install/ubuntu/
- - - -

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Run Docker as non-root:
```
sudo groupadd docker
sudo usermod -aG docker $USER
```
Restart terminal session for changes to apply

## Setup HammerDB TPROC-C MySQL Database for SUT ##

Set Static CPU Management Policy and Static NUMA-aware Memory Manager
```
sudo vi /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

Modify the file by adding the following flags:
```
ExecStart=/usr/bin/kubelet --cpu-manager-policy=static --reserved-cpus=0-7,256-263 $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

Perform the following to enable the Static policies:
```
sudo rm -rf /var/lib/kubelet/cpu_manager_state
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
\
On the SUT, install Multus:
```
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git
kubectl apply -f multus-cni/deployments/multus-daemonset.yml
```
\
Copy the conf.d folder of this repository to your drive mount point for each NUMA
```
cp -r mysql/conf.d /mnt1
...
```

Update the required memory for the MySQL database in conf.d/my.cnf file here (currently set at 20G):
```
innodb_buffer_pool_size=20G
```

Create a empty directory called db1 on your mount point. This will be where the TPC database is built.
```
mkdir /mnt/db1
```

Update the tpc-db.yaml file found under the mysql directory to reflect your setup. \
Ensure the yaml file reflects your NIC physical function name under the NetworkAttachmentDefinition:
```
"master": "enp129s0f0np0",
```
You may also change the address under the NetworkAttachmentDefintion. \
Ensure the yaml file reflects the name of your mount paths (do not change volumeMounts):
```
      volumes:
        - name: tpc-db1-conf
          hostPath:
            path: /mnt1/conf.d
        - name: tpc-db1-vol
          hostPath:
            path: /mnt1/db1
```
Launch the pod that will run a MySQL container listening at the specified address at port 3306:
```
kubectl apply -f tpc-db.yaml
```
MySQL pod will restart in about 10 seconds in order to populate filesystem. This is normal.

## Setup HammerDB TPROC for Load Generator ##

Unzip the HammerDB repository from tar file \

Build the HammerDB Docker Image:
```
cd HammerDB/Docker
docker build -t hammerdb .
```

Change access permission mode on tp_run
```
cd TPROC-H/tpch-loadgen
chmod +x tp_run
```

Copy the tpcc directory of this repository to the load generator. Create the load generator image:
```
docker build -t tpch .
```

Run the container in interactive mode:
```
docker run -it --rm tpch bash
```

Set the environment variables and build the initial database: \

```
root@c10b1e311091:/home/hammerdb/HammerDB-4.5# export DBHOST=192.168.1.101    
root@c10b1e311091:/home/hammerdb/HammerDB-4.5# export DBPORT=3306         
root@c10b1e311091:/home/hammerdb/HammerDB-4.5# export MYSQL_USER=root 
root@c10b1e311091:/home/hammerdb/HammerDB-4.5# export MYSQL_PASSWORD=password 
root@c10b1e311091:/home/hammerdb/HammerDB-4.5# export SF=30 
root@c10b1e311091:/home/hammerdb/HammerDB-4.5# export TH=4
root@c10b1e311091:/home/hammerdb/HammerDB-4.5# export VU=4
root@c10b1e311091:/home/hammerdb/HammerDB-4.5# ./hammerdbcli
hammerdb> source build.tcl
```
It will take approx 3 hours to build the database and will be ~74GB

Once the output shows that the database build is complete, exit the container on the load generator and delete the pod on the SUT. \
The database that will be used will be saved at your specified mount point. You can check the size using the following:
```
cd /mnt
sudo du -h
```

## Scaling the TPC-H Benchmark ##
Increase IO on the SUT:
```
sudo sysctl -w fs.aio-max-nr=1048576
sudo sysctl -w fs.file-max=6815744
sudo sysctl --system
```


Copy the database as your gold database for reuse so you do not have to rebuild the database:
```
cp -r /mnt/db1 /mnt/golddb
```

Copy the gold database to the number of instances you are trying to scale to. Keep in mind to balance the pods across NUMAs.
```
cp -r /mnt/golddb /mnt/db2
...
cp -r /mnt/golddb /mnt/db16
cp -r /mnt/golddb /mnt5/db17
...
cp -r /mnt/golddb /mnt9/db32
```

Next, use script-gen.sh found under mysql/scripts in this repository to create your yaml file. \
Be sure to update the physical function name, the first three numbers of the address and CPU/Memory limits and request before running. \
Be sure to balance pods across drives \
```
./script-gen.sh mnt1 1 4 > mnt1.yaml
./script-gen.sh mnt2 5 8 > mnt2.yaml
...
```

Launch the pods and wait for them to reach a running state (should take less than a minute):
```
kubectl apply -f mnt1.yaml
...
```
Pods are assigned CPUs in numerical order (and smt-thread). Check CPU assignment here (do not modify this file, file is dynamic):
```
sudo cat /var/lib/kubelet/cpu_manager_state
```

If pods fail to initialize due to MySQL, delete the pods and apply the following:
```
sudo sysctl -w fs.aio-max-nr=1048576
sudo sysctl -w fs.file-max=6815744
sudo sysctl --system
```
Relaunch the pods. \
\
On the load generator, modify run.sh file to reflect all your IPs
```
#!/bin/bash

if test "$#" -ne 2; then
        echo "$0 pwr|tp vmX"
fi

IMG=tpch:latest
DBPORT=3306
SF=30
TH=4

CMD=/home/hammerdb/HammerDB-4.5/$1_run

vm1_HOST=192.78.1.1
vm2_HOST=192.78.1.2
vm3_HOST=192.78.1.3
vm4_HOST=192.78.1.4
vm5_HOST=192.78.1.5
vm6_HOST=192.78.1.6
vm7_HOST=192.78.1.7
vm8_HOST=192.78.1.8
vm9_HOST=192.78.1.9
vm10_HOST=192.78.1.10
vm11_HOST=192.78.1.11
vm12_HOST=192.78.1.12
vm13_HOST=192.78.1.13
vm14_HOST=192.78.1.14
vm15_HOST=192.78.1.15
vm16_HOST=192.78.1.16
vm17_HOST=192.78.1.17
vm18_HOST=192.78.1.18
vm19_HOST=192.78.1.19
vm20_HOST=192.78.1.20
vm21_HOST=192.78.1.21
vm22_HOST=192.78.1.22
vm23_HOST=192.78.1.23
vm24_HOST=192.78.1.24
vm25_HOST=192.78.1.25
vm26_HOST=192.78.1.26
vm27_HOST=192.78.1.27
vm28_HOST=192.78.1.28
vm29_HOST=192.78.1.29
vm30_HOST=192.78.1.30
vm31_HOST=192.78.1.31

NAME=tpch-$2
declare -n DBHOST
DBHOST=$2_HOST

OPTS="-e MYSQL_USER=root -e MYSQL_PASSWORD=password -e DBHOST=${DBHOST} -e DBPORT=${DBPORT} -e SF=${SF} -e TH=${TH}"
LOG=$2.txt

docker run --name ${NAME} --rm ${OPTS} ${IMG} ${CMD} > ${LOG} &
```

Modify bm.sh to run the desired number of instances \
Run the test:
```
bash ./bm.sh tp
```

Once the test is complete, use the following command to obtain results:
```
grep Completed vm1.txt
...
```
Use the slowest time to calculate QPH \
NOTE: use capital C in 'grep Completed vm1.txt'. Do not use 'grep completed vm1.txt'
\

Script to extract last VU Completed time for all vm text files:
```
#!/bin/bash

for ((I=1; I<=4; I++))
do
  grep Completed vm${I}.txt | tail -n 1 | awk '{print$(NF-1)}'
done
```

On the SUT, delete the pods
```
kubectl delete -f mnt1.yaml
kubectl delete -f mnt2.yaml
...
...
```

On the SUT, delete the databases
```
sudo rm -rf /mnt1/db*
sudo rm -rf /mnt2/db*
...
```

Recreate databases and redeploy pods as needed.


 
Remove Kubernetes and ContainerD:
```
sudo kubeadm reset
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni
sudo rm -rf /etc/cni/net.d
sudo rm -rf ~/.kube
sudo apt-get purge containerd.io
sudo apt-get autoremove
```


