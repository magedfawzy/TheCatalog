#######################################################################
# Objective: Automate K8S Lab Creation
# Desc: K8S|Calico|Helm|Dash|Prometheus|Grafana|NetData|Gremlin|Litmus
# Created By: maged.fawzy@ericsson.com
# Date: 2020-09-08
#######################################################################
Vagrant.configure("2") do |config|
  config.vm.provider :virtualbox do |v|
    v.memory = 8024
    v.cpus = 4
    v.customize ["modifyvm", :id, "--ioapic", "on"]
    v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]  
  end

  config.vm.provision :shell, privileged: true, inline: $install_common_tools
  config.vm.synced_folder ".", "/vagrant"

  config.vm.define :master do |master|
    master.vm.box = "generic/ubuntu1804" # walidsaad/RancherOS_1.5.7      
    master.vm.hostname = "master"
    master.vm.network :private_network, ip: "172.16.0.21"    
    master.vm.network "forwarded_port", guest: 8001, host: 8001  #used for kube proxy to connect all K8S services

    master.vm.provision :shell, privileged: false, inline: $provision_master_node
    master.vm.provision :shell, privileged: false, inline: $provision_master_toolkit
    master.vm.provision :shell, privileged: false, inline: $provision_master_networking
            
  end

  %w{worker1 worker2}.each_with_index do |name, i|
    config.vm.define name do |worker|
      worker.vm.box = "generic/ubuntu1804"
      worker.vm.hostname = name
      worker.vm.network :private_network, ip: "172.16.0.#{i + 11}"
      worker.vm.provision :shell, privileged: false, inline: $provision_worker_node
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end


#####  On All Nodes #############################################################
$install_common_tools = <<-SHELL
# bridged traffic to iptables is enabled for kube-router.
cat >> /etc/ufw/sysctl.conf <<EOF
net/bridge/bridge-nf-call-ip6tables = 1
net/bridge/bridge-nf-call-iptables = 1
net/bridge/bridge-nf-call-arptables = 1
EOF

# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Install kubeadm, kubectl and kubelet
export DEBIAN_FRONTEND=noninteractive
apt-get -qq install ebtables ethtool
apt-get -qq update
apt-get -qq install -y docker.io apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -qq update
apt-get -qq install -y kubelet kubeadm kubectl

#enable docker
sudo systemctl enable docker.service
sudo systemctl start docker.service

####  Install Helm #####
sudo curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
sudo chmod 700 get_helm.sh
sudo ./get_helm.sh

#### Create Volumes ####
mkdir -p /mnt/data/pv0 /mnt/data/pv1 /mnt/data/pv2 /mnt/data/pv3
chmod 777 /mnt/data/*

SHELL

#### On Master Only ####################################################################
$provision_master_node = <<-SCRIPT
OUTPUT_FILE=/vagrant/join.sh
rm -rf $OUTPUT_FILE
rm -rf /vagrant/config

# Start cluster
sudo kubeadm init --apiserver-advertise-address=172.16.0.21 --pod-network-cidr=10.96.0.0/16 | grep -Ei "kubeadm join|discovery-token-ca-cert-hash" > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo cp -i /etc/kubernetes/admin.conf /vagrant/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=172.16.0.21"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Configure Calico
kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SCRIPT

#################################################################################
$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL

#################################################################################
$provision_master_toolkit = <<-SHELL

OUTPUT_FILE=/vagrant/credentials.txt

##### Install K8S Dashboard UI #####
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
#Access via : http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

echo "Dashboard UI Token" >> ${OUTPUT_FILE}
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}') >> ${OUTPUT_FILE}
echo "==================" >> ${OUTPUT_FILE}

##### Install NetData UI #####
sudo git clone https://github.com/netdata/helmchart.git netdata-helmchart

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/pv0"
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/pv1"
EOF

sudo helm install netdata --set parent.database.volumesize=1Gi ./netdata-helmchart

SHELL

#######

$provision_master_networking = <<-SHELL

#by default k8s uses 127.0.0.1 and it gets trapped inside cluster, to expose below
kubectl proxy --address=0.0.0.0 & 

SHELL

#######

$provision_worker_node = <<-SHELL

sudo /vagrant/join.sh

mkdir -p $HOME/.kube
sudo cp -i /vagrant/config $HOME/.kube/config   #enables kubectl to query apiserver at master
sudo chown $(id -u):$(id -g) $HOME/.kube/config

ip=$(ifconfig eth1 | grep 'inet ' | awk -F" " '{print $2}')
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip='$ip'"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

sudo systemctl daemon-reload
sudo systemctl restart kubelet


SHELL