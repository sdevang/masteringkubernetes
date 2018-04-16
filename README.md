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

cat > pre_reqs.yaml <<EOF
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

cat > kernel_params.yaml <<EOF
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

