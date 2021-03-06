
Vagrant.configure("2") do |config|

  # Sync time with the local host
  config.vm.provider 'virtualbox' do |vb|
   vb.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 1000 ]
  end

  $num_instances = 3


  (1..$num_instances).each do |i|

    config.vm.define "k8s-node#{i}" do |node|
    # Xenial is a bit old but works best at the moment. Kubectl repo seems to be
    # available for Xenial only currently
    node.vm.box = "ubuntu/xenial64"
    node.vm.hostname = "k8s-node#{i}"
    ip = "10.85.20.#{i+245}"
    node.vm.network "public_network", bridge: "en0: Ethernet", ip: ip
    #if "#{i}" == "1"
    #  node.vm.network "forwarded_port", guest: 30002, host: 30002, protocol: "tcp"
    #end
    # Change the share to match your environment
    #node.vm.synced_folder "/home/jonas/ENV/VAGRANT/share", "/home/vagrant/share"

    node.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      # K8s requires at least 2 CPUs
      vb.cpus = 2
      vb.name = "k8s-node#{i}"
    end

    node.vm.provision "shell" do |s|
      s.inline = <<-SHELL

        echo "-------------------------------------------------------------------------- Update hosts file"
cat >> /etc/hosts <<EOF
10.85.20.246 k8s-node1
10.85.20.247 k8s-node2
10.85.20.248 k8s-node3
EOF

cat /etc/hosts

echo "-------------------------------------------------------------------------- Update DNS settings"
echo "nameserver 8.8.8.8">/etc/resolv.conf
cat /etc/resolv.conf

echo "-------------------------------------------------------------------------- Disable swap"
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

echo "-------------------------------------------------------------------------- Adding required apt keys and Kubernetes repos"

wget -qO- https://packages.cloud.google.com/apt/doc/apt-key.gpg > /home/vagrant/apt-key.gpg
apt-key add /home/vagrant/apt-key.gpg
echo "apt_preserve_sources_list: true" >> /etc/cloud/cloud.cfg

apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
apt-get update

echo "-------------------------------------------------------------------------- Install SSH keys"
wget -qO- https://raw.githubusercontent.com/jonas-werner/pubkeys/master/nopass.pub >> /home/vagrant/.ssh/authorized_keys

echo "-------------------------------------------------------------------------- Installing Kubeadm and Docker"
apt-get install kubeadm -y

### Install packages to allow apt to use a repository over HTTPS
apt-get update && apt-get install apt-transport-https ca-certificates curl software-properties-common -y

### Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

### Add Docker apt repository.
add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

## Install Docker CE.
apt-get update && apt-get install docker-ce=18.06.2~ce~3-0~ubuntu -y

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker
# Enable packet forwarding
sysctl net.bridge.bridge-nf-call-iptables=1


###### MASTER NODE START ######
if [[ #{i} -eq 1 ]];then
  echo "-------------------------------------------------------------------------- Configuring Master node"

  systemctl enable kubelet
  systemctl start kubelet

  kubeadm init --apiserver-advertise-address=10.85.20.246 --pod-network-cidr=10.244.0.0/16 | tee /home/vagrant/k8s-install.log
  cat /home/vagrant/k8s-install.log | grep "kubeadm join" > /vagrant/cluster-join.sh
  echo "--discovery-token-unsafe-skip-ca-verification" >> /vagrant/cluster-join.sh

  mkdir -p /home/vagrant/.kube
  cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  chown vagrant:vagrant /home/vagrant/.kube/config

  echo "-------------------------------------------------------------------------- Installing Flannel network"
  runuser -l vagrant -c "kubectl apply -f /vagrant/kube-flannel.yml"

  echo "-------------------------------------------------------------------------- Installing and configuring LoadBalancer"
  #runuser -l vagrant -c "kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml"
  #runuser -l vagrant -c "kubectl apply -f /vagrant/metal-lb-config.yaml"
fi
###### MASTER NODE END ######

###### WORKER NODE START ######
if [[ #{i} -gt 1 ]];then
  echo "-------------------------------------------------------------------------- Configuring Worker node"

  cp /vagrant/cluster-join.sh /home/vagrant/cluster-join.sh
  chmod 755 /home/vagrant/cluster-join.sh
  /vagrant/cluster-join.sh

fi
###### WORKER NODE END ######


      SHELL
      end
    end
  end
end
