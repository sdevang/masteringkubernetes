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
Create a file called Vagrantfile with following content inside the directory you just created.

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
