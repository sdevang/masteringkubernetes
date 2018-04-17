# Mastering Kubernetes

Please go follow the following steps in order to install 6 nodes kubernetes cluster using Kubeadm. The tutorial will help you setup 3 master nodes which will be also be hosting etcd cluster. 

The overall setup will look like as following.

![Alt text](K8-Setup.png?raw=true "Kubernetes Setup")

## Infrastructure Setup

So lets start creating the infrastructure needed for our kubernetes setup.

This tutorial is going assume you have vagrant and virtualbox installed. Feel free to download them from https://www.vagrantup.com/ and https://www.virtualbox.org/wiki/Downloads in case if you dont have it.

After you have vagrant and virtualbox installed, Please install vagrant plug-in called landrush.

```bash
$vagrant plugin install landrush
```

After that,

```bash
$mkdir $HOME/masteringkubernetes
```
Create a hosts file inside $HOME/masteringkubernetes with following content.

```bash
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
192.168.33.20 master0.devopsgeek.co.uk master0
192.168.33.21 master1.devopsgeek.co.uk master1
192.168.33.22 master2.devopsgeek.co.uk master2
192.168.33.23 node1.devopsgeek.co.uk node1
192.168.33.24 node2.devopsgeek.co.uk node2
192.168.33.25 node3.devopsgeek.co.uk node3
```

After that create a file called Vagrantfile with following content inside the directory you just created.

```bash
domain   = 'devopsgeek.co.uk'

# use two digits id below, please
nodes = [
  { :hostname => 'master0',  :ip => '192.168.33.20', :id => '20' },
  { :hostname => 'master1',  :ip => '192.168.33.21', :id => '21' },
  { :hostname => 'master2',  :ip => '192.168.33.22', :id => '22' },
  { :hostname => 'node1',   :ip => '192.168.33.23', :id => '23' },
  { :hostname => 'node2',   :ip => '192.168.33.24', :id => '24' },
  { :hostname => 'node3',   :ip => '192.168.33.25', :id => '25' },
]

memory = 1500

$script = <<SCRIPT
sudo mv hosts /etc/hosts
chmod 0600 /home/vagrant/.ssh/id_rsa
usermod -a -G vagrant ubuntu
cp -Rvf /home/vagrant/.ssh /home/ubuntu
chown -Rvf ubuntu /home/ubuntu
apt-get -y update
apt-get -y install python-minimal python-apt
SCRIPT

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false
  nodes.each do |node|
    config.vm.define node[:hostname] do |nodeconfig|
      nodeconfig.vm.box = "ubuntu/xenial64"
      nodeconfig.vm.hostname = node[:hostname]
      nodeconfig.landrush.enabled = true
      nodeconfig.vm.network :private_network, ip: node[:ip]
      nodeconfig.vm.provider :virtualbox do |vb|
        vb.name = node[:hostname]+"."+domain
        vb.memory = memory
        vb.cpus = 2
        vb.customize ['modifyvm', :id, '--natdnshostresolver2', 'on']
        vb.customize ['modifyvm', :id, '--natdnsproxy2', 'on']
        vb.customize ['modifyvm', :id, '--macaddress2', "5CA2AB2E00"+node[:id]]
        vb.customize ['modifyvm', :id, '--natnet2', "192.168/16"]
      end
      nodeconfig.vm.provision "file", source: "hosts", destination: "hosts"
      nodeconfig.vm.provision "file", source: "~/.vagrant.d/insecure_private_key", destination: "/home/vagrant/.ssh/id_rsa"
    end
  end
end
```
After you have the vagrantfile, create the VMs as following.

```bash
$ cd $HOME/masteringkubernetes
$ vagrant up
```
Please note vagrant up will take some time depending upon the hardware you have got on your machine. It should take around 15-20 min I reckon.

## Preparing VMs for Kubernetes

You are going to installed ansible installed on your host OS (Laptop or PC) before you can bootstrap the VMs for kubernetes installation.

For MacOS perform following steps,

```bash
Install Xcode
sudo easy_install pip
sudo pip install ansible --quiet
```
For Linux perform following steps,

```bash
For RHEL based OS,
sudo yum install ansible

For Debian based OS,
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

After you have ansible installed, please create following files.

```bash

cd $HOME/masteringkubernetes

cat >ansible.cfg <<EOF
[defaults]
inventory = ./inventory
EOF

cat >inventory <<EOF
[all]
master0 ansible_connection=ssh ansible_ssh_user=vagrant
master1 ansible_connection=ssh ansible_ssh_user=vagrant
master2 ansible_connection=ssh ansible_ssh_user=vagrant
node1 ansible_connection=ssh ansible_ssh_user=vagrant
node2 ansible_connection=ssh ansible_ssh_user=vagrant
node3 ansible_connection=ssh ansible_ssh_user=vagrant

[masters]
master0 ansible_connection=ssh ansible_ssh_user=vagrant
master1 ansible_connection=ssh ansible_ssh_user=vagrant
master2 ansible_connection=ssh ansible_ssh_user=vagrant

[workers]
node1 ansible_connection=ssh ansible_ssh_user=vagrant
node2 ansible_connection=ssh ansible_ssh_user=vagrant
node3 ansible_connection=ssh ansible_ssh_user=vagrant
EOF

cat >pre_reqs.yaml <<EOF
---
- hosts: all
  become: yes
  tasks:
  - name: Updating OS.
    shell: "apt-get -y update && apt-get install -y apt-transport-https"
  - name: Adding K8 Repo.
    apt_repository:
      repo: 'deb http://apt.kubernetes.io/ kubernetes-xenial main'
      state: present
  - name: Adding Public Key for K8 Repo.
    apt_key:
      url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
      state: present
      validate_certs: no
  - name: Installing required packages.
    apt: name={{ item }} state=present allow_unauthenticated=yes
    with_items:
       - docker.io
       - kubelet
       - kubeadm
       - kubectl
       - ntp
EOF

cat >kernel_params.yaml <<EOF
---
- hosts: all
  become: yes
  tasks:
  - name: Add Kernel Modules.
    command: modprobe {{item}}
    with_items:
      - ip_vs
      - ip_vs_rr
      - ip_vs_wrr
      - ip_vs_sh
      - nf_conntrack_ipv4
  - name: Kernel Modules Boot time configuration.
    lineinfile: path=/etc/modules line='{{item}}' create=yes state=present
    with_items:
      - ip_vs
      - ip_vs_rr
      - ip_vs_wrr
      - ip_vs_sh
      - nf_conntrack_ipv4
  - name: Set up IP Forwarding.
    sysctl: name=net.ipv4.ip_forward value=1 state=present reload=yes sysctl_set=yes
  - name: Make Sure Services are enabled at boot.
    service: name={{item}} state=started enabled=yes
    with_items:
      - docker
      - ntp
      - kubelet
EOF
```

Now you are good to run ansible playbooks on our VMs.

You need to run both the ansible playbooks on all the nodes in the cluster so let's run,

```bash
$ ansible-playbook -i inventory pre-reqs.yaml
$ ansible-playbook -i inventory kernel-params.yaml
```

If the ansible playbooks ran successfully then you have configured the nodes for Kubernetes installation.

## Setting up SSH Config

Now lets setup SSH config for our VMs. SSH config is not a mandatory step but having SSH config setup will help your life easy.

Run the following command,

```bash

$vagrant ssh-config master0 >> $HOME/.ssh/config
$vagrant ssh-config master1 >> $HOME/.ssh/config
$vagrant ssh-config master2 >> $HOME/.ssh/config
$vagrant ssh-config node1 >> $HOME/.ssh/config
$vagrant ssh-config node2 >> $HOME/.ssh/config
$vagrant ssh-config node3 >> $HOME/.ssh/config

```
The above command will add SSH configuration for each VM into your $HOME/.ssh/config file so after that you can ssh into any machine by referring to its name. You do not need any DNS servers for VM's hostname to IP resolution after you setup SSH config as shown above.

## Passwordless SSH authentication

We need to copy over a few files and certificates throughout the installation so to make our life easy, lets setup passwordless ssh authetication between all the master nodes.

The first step is,

```bash
Generate SSH Key pair on master0 node,

$ssh master0
$sudo su - 
#ssh-keygen -t rsa <-- Accept the default for everything

$ssh master1
$sudo su - 
#ssh-keygen -t rsa <-- Accept the default for everything

$ssh master2
$sudo su - 
#ssh-keygen -t rsa <-- Accept the default for everything
```

One the SSH key pair is generated on all the master nodes, you need to copy the content of each respective node's public into a file called authorized_keys on all the master nodes.

```bash
$ssh master0
$sudo su - 
#cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys

$ssh master1
$sudo su - 
#cat $HOME/.ssh/id_rsa.pub <-- Copy the content of this file and paste on master0:/root/.ssh/authorized_keys

$ssh master2
$sudo su - 
#cat $HOME/.ssh/id_rsa.pub <-- Copy the content of this file and paste on master0:/root/.ssh/authorized_keys
```

Now master0 node's authorized_keys file have the public key for master0, master1 and master2 nodes so copy over that file to each master nodes.

```bash
$ssh master0
$sudo su - 
#scp $HOME/.ssh/authorized_keys master1:/root/.ssh
#scp $HOME/.ssh/authorized_keys master2:/root/.ssh
```

Lets verify now whether we can run any commands remotely on any of the master node and it should not ask for the password.

```bash
$ssh master0
$sudo su - 
#ssh master0 date
#ssh master1 date
#ssh master2 date
```
If it does not ask for password, then you have configured password less authentication on all the master nodes.


## ETCd clustering

You must have a storage backend for kubernetes so that kubernetes can store its metadata persitently. For that purpose, we are going to use etcd which is a famous and widely used key-value store used for kubernetes.

Ideally you must have a dedicated nodes for etcd cluster but for the demo purpose, we are going to use master nodes as etcd nodes as well.

So lets start building etcd cluster.

We are going to configure etcd in a secure fashion so we are going to need some SSL certificates generated. For that purpose, we will use cfssl and cfssljson utilities.

So lets download and install those utilties first.

```bash
connect to master0 
$ssh master0
$sudo su - 
#curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
#curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
#chmod +x /usr/local/bin/cfssl*

connect to master1 
$ssh master1
$sudo su - 
#curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
#curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
#chmod +x /usr/local/bin/cfssl*

connect to master2
$ssh master2
$sudo su - 
#curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
#curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
#chmod +x /usr/local/bin/cfssl*
```

On master0 node,

```bash

#mkdir -p /etc/kubernetes/pki/etcd
#cd /etc/kubernetes/pki/etcd

#cat >ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF

#cat >ca-csr.json <<EOF
{
    "CN": "etcd",
    "key": {
        "algo": "rsa",
        "size": 2048
    }
}
EOF

#cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

On master0:

#cat >client.json <<EOF
{
    "CN": "client",
    "key": {
        "algo": "ecdsa",
        "size": 256
    }
}
EOF

#cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client

#export PEER_NAME=$(hostname)
#export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+')

#cfssl print-defaults csr > config.json
#sed -i '0,/CN/{s/example\.net/'"$PEER_NAME"'/}' config.json
#sed -i 's/www\.example\.net/'"$PRIVATE_IP"'/' config.json
#sed -i 's/example\.net/'"$PEER_NAME"'/' config.json

#cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server config.json | cfssljson -bare server
#cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer config.json | cfssljson -bare peer
```

Above commands would have generated a few certificates under /etc/kubernetes/pki/etcd directory on master0 node.

Now lets copy over these certificates to master1 and master2 nodes.

```bash

$ssh master0
$sudo su - 
#cd /etc/kubernetes/pki/etcd
#scp ca.pem root@master1:/etc/kubernetes/pki/etcd/ca.pem
#scp ca-key.pem root@master1:/etc/kubernetes/pki/etcd/ca-key.pem
#scp client.pem root@master1:/etc/kubernetes/pki/etcd/client.pem
#scp client-key.pem root@master1:/etc/kubernetes/pki/etcd/client-key.pem
#scp ca-config.json root@master1:/etc/kubernetes/pki/etcd/ca-config.json

#scp ca.pem root@master2:/etc/kubernetes/pki/etcd/ca.pem
#scp ca-key.pem root@master2:/etc/kubernetes/pki/etcd/ca-key.pem
#scp client.pem root@master2:/etc/kubernetes/pki/etcd/client.pem
#scp client-key.pem root@master2:/etc/kubernetes/pki/etcd/client-key.pem
#scp ca-config.json root@master2:/etc/kubernetes/pki/etcd/ca-config.json

```
Now lets create etcd environment file on all master nodes.


```bash
$ssh master0
$sudo su - 
#export PEER_NAME=$(hostname)
#export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+') <-- Please rememeber so replace enp0s8 with your NIC ID.
#touch /etc/etcd.env
#echo "PEER_NAME=$PEER_NAME" >> /etc/etcd.env
#echo "PRIVATE_IP=$PRIVATE_IP" >> /etc/etcd.env

$ssh master1
$sudo su - 
#export PEER_NAME=$(hostname)
#export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+') <-- Please rememeber so replace enp0s8 with your NIC ID.
#touch /etc/etcd.env
#echo "PEER_NAME=$PEER_NAME" >> /etc/etcd.env
#echo "PRIVATE_IP=$PRIVATE_IP" >> /etc/etcd.env

$ssh master2
$sudo su - 
#export PEER_NAME=$(hostname)
#export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+') <-- Please rememeber so replace enp0s8 with your NIC ID.
#touch /etc/etcd.env
#echo "PEER_NAME=$PEER_NAME" >> /etc/etcd.env
#echo "PRIVATE_IP=$PRIVATE_IP" >> /etc/etcd.env
```
Now lets download etcd binary. Please note kubernetes is bit fussy with etcd version so download the supported version of etcd only.

```bash
$ssh master0
$sudo su - 
export ETCD_VERSION=v3.1.10
curl -sSL https://github.com/coreos/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz | tar -xzv --strip-components=1 -C /usr/local/bin/
rm -rf etcd-$ETCD_VERSION-linux-amd64*

$ssh master1
$sudo su - 
export ETCD_VERSION=v3.1.10
curl -sSL https://github.com/coreos/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz | tar -xzv --strip-components=1 -C /usr/local/bin/
rm -rf etcd-$ETCD_VERSION-linux-amd64*

$ssh master2
$sudo su - 
export ETCD_VERSION=v3.1.10
curl -sSL https://github.com/coreos/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz | tar -xzv --strip-components=1 -C /usr/local/bin/
rm -rf etcd-$ETCD_VERSION-linux-amd64*
```

After that lets create a service defination file which will be used to start etcd service on each node.

```bash
$ssh master0
$sudo su - 
#export PEER_NAME=$(hostname)
#export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+')
#cat >/etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos/etcd
Conflicts=etcd.service
Conflicts=etcd2.service

[Service]
EnvironmentFile=/etc/etcd.env
Type=notify
Restart=always
RestartSec=5s
LimitNOFILE=40000
TimeoutStartSec=0

ExecStart=/usr/local/bin/etcd --name ${PEER_NAME} \
    --data-dir /var/lib/etcd \
    --listen-client-urls https://${PRIVATE_IP}:2379 \
    --advertise-client-urls https://${PRIVATE_IP}:2379 \
    --listen-peer-urls https://${PRIVATE_IP}:2380 \
    --initial-advertise-peer-urls https://${PRIVATE_IP}:2380 \
    --cert-file=/etc/kubernetes/pki/etcd/server.pem \
    --key-file=/etc/kubernetes/pki/etcd/server-key.pem \
    --client-cert-auth \
    --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
    --peer-cert-file=/etc/kubernetes/pki/etcd/peer.pem \
    --peer-key-file=/etc/kubernetes/pki/etcd/peer-key.pem \
    --peer-client-cert-auth \
    --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
    --initial-cluster master0=https://master0:2380,master1=https://master1:2380,master2=https://master2:2380 \
    --initial-cluster-token my-etcd-token \
    --initial-cluster-state new

[Install]
WantedBy=multi-user.target
EOF

$ssh master1
$sudo su - 
#export PEER_NAME=$(hostname)
#export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+')
#cat >/etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos/etcd
Conflicts=etcd.service
Conflicts=etcd2.service

[Service]
EnvironmentFile=/etc/etcd.env
Type=notify
Restart=always
RestartSec=5s
LimitNOFILE=40000
TimeoutStartSec=0

ExecStart=/usr/local/bin/etcd --name ${PEER_NAME} \
    --data-dir /var/lib/etcd \
    --listen-client-urls https://${PRIVATE_IP}:2379 \
    --advertise-client-urls https://${PRIVATE_IP}:2379 \
    --listen-peer-urls https://${PRIVATE_IP}:2380 \
    --initial-advertise-peer-urls https://${PRIVATE_IP}:2380 \
    --cert-file=/etc/kubernetes/pki/etcd/server.pem \
    --key-file=/etc/kubernetes/pki/etcd/server-key.pem \
    --client-cert-auth \
    --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
    --peer-cert-file=/etc/kubernetes/pki/etcd/peer.pem \
    --peer-key-file=/etc/kubernetes/pki/etcd/peer-key.pem \
    --peer-client-cert-auth \
    --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
    --initial-cluster master0=https://master0:2380,master1=https://master1:2380,master2=https://master2:2380 \
    --initial-cluster-token my-etcd-token \
    --initial-cluster-state new

[Install]
WantedBy=multi-user.target
EOF

$ssh master2
$sudo su - 
#export PEER_NAME=$(hostname)
#export PRIVATE_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+')
#cat >/etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos/etcd
Conflicts=etcd.service
Conflicts=etcd2.service

[Service]
EnvironmentFile=/etc/etcd.env
Type=notify
Restart=always
RestartSec=5s
LimitNOFILE=40000
TimeoutStartSec=0

ExecStart=/usr/local/bin/etcd --name ${PEER_NAME} \
    --data-dir /var/lib/etcd \
    --listen-client-urls https://${PRIVATE_IP}:2379 \
    --advertise-client-urls https://${PRIVATE_IP}:2379 \
    --listen-peer-urls https://${PRIVATE_IP}:2380 \
    --initial-advertise-peer-urls https://${PRIVATE_IP}:2380 \
    --cert-file=/etc/kubernetes/pki/etcd/server.pem \
    --key-file=/etc/kubernetes/pki/etcd/server-key.pem \
    --client-cert-auth \
    --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
    --peer-cert-file=/etc/kubernetes/pki/etcd/peer.pem \
    --peer-key-file=/etc/kubernetes/pki/etcd/peer-key.pem \
    --peer-client-cert-auth \
    --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
    --initial-cluster master0=https://master0:2380,master1=https://master1:2380,master2=https://master2:2380 \
    --initial-cluster-token my-etcd-token \
    --initial-cluster-state new

[Install]
WantedBy=multi-user.target
EOF
```

After that on each master nodes,

```bash
#systemctl daemon-reload
#systemctl start etcd
#systemctl enable etcd
```

The last command should have started the etcd cluster so lets verify it.

```bash
$ssh master0
$sudo su - 
#export ETCDCTL_ENDPOINTS=https://master0:2379
#export ETCDCTL_CA_FILE=/etc/kubernetes/pki/etcd/ca.pem
#export ETCDCTL_CERT_FILE=/etc/kubernetes/pki/etcd/server.pem
#export ETCDCTL_KEY_FILE=/etc/kubernetes/pki/etcd/server-key.pem

#etcdctl member list

This should return three nodes which are running etcd cluster.
```

## Keepalived Configuration

On each master node,

```bash
#apt install keepalived
```

On master0 node,

```bash
#cat >/etc/keepalived/keepalived.conf <<EOF
 ! Configuration File for keepalived
 global_defs {
   router_id LVS_DEVEL
 }
    
 vrrp_script check_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 3
   weight -2
   fall 10
   rise 2
 }
    
 vrrp_instance VI_1 {
     state MASTER
     interface enp0s8
     virtual_router_id 51
     priority 101
     authentication {
         auth_type PASS
         auth_pass 4be37dc3b4c90194d1600c483e10ad1d
     }
     virtual_ipaddress {
         192.168.33.50
     }
     track_script {
         check_apiserver
     }
 }
EOF
```

On master1 node,

```bash
#cat >/etc/keepalived/keepalived.conf <<EOF
 ! Configuration File for keepalived
 global_defs {
   router_id LVS_DEVEL
 }
    
 vrrp_script check_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 3
   weight -2
   fall 10
   rise 2
 }
    
 vrrp_instance VI_1 {
     state BACKUP
     interface enp0s8
     virtual_router_id 51
     priority 100
     authentication {
         auth_type PASS
         auth_pass 4be37dc3b4c90194d1600c483e10ad1d
     }
     virtual_ipaddress {
         192.168.33.50
     }
     track_script {
         check_apiserver
     }
 }
EOF
```

On master2 node,

```bash
#cat >/etc/keepalived/keepalived.conf <<EOF
 ! Configuration File for keepalived
 global_defs {
   router_id LVS_DEVEL
 }
    
 vrrp_script check_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 3
   weight -2
   fall 10
   rise 2
 }
    
 vrrp_instance VI_1 {
     state BACKUP
     interface enp0s8
     virtual_router_id 51
     priority 100
     authentication {
         auth_type PASS
         auth_pass 4be37dc3b4c90194d1600c483e10ad1d
     }
     virtual_ipaddress {
         192.168.33.50
     }
     track_script {
         check_apiserver
     }
 }
EOF
```

On each master nodes,

```bash
#cat >/etc/keepalived/check_apiserver.sh <<EOF
 #!/bin/sh

 errorExit() {
     echo "*** $*" 1>&2
     exit 1
 }

 curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
 if ip addr | grep -q 192.168.33.50; then
     curl --silent --max-time 2 --insecure https://192.168.33.50:6443/ -o /dev/null || errorExit "Error GET https://192.168.33.50:6443/"
 fi
EOF

#chmod 755 /etc/keepalived/check_apiserver.sh
#systemctl restart keepalived

```

Now we have keepalived configured on each master nodes.
