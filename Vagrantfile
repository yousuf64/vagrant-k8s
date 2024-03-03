# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/bionic64"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.provider "hyperv" do |hv|
    hv.cpus = 2
    hv.memory = "2048"
  end
  
  config.vm.provision "shell", inline: <<-SHELL
    # Disable swap
    sudo sed -i.bak '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

    # Load overlay, br_netfilter modules
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
      overlay
      br_netfilter
EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter

    # Network settings
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1
EOF

    sudo sysctl --system
  SHELL

  # Install CRI (CRI-O)  
  config.vm.provision "shell", inline: <<-SHELL
    export OS=xUbuntu_18.04
    export CRIO_VERSION=1.28
    
    sudo echo 'deb http://deb.debian.org/debian buster-backports main' > /etc/apt/sources.list.d/backports.list
    sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 0E98404D386FA1D9
    sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 6ED0E7B82643E131
    sudo apt update
    sudo apt install -y -t buster-backports libseccomp2 || sudo apt update -y -t buster-backports libseccomp2 # install or update (if exists) libseccomp2 from buster-backports

    sudo apt-get install ca-certificates -y
    
    sudo su <<EOF
      echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > \
        /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list

      echo "deb [signed-by=/usr/share/keyrings/libcontainers-crio-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$CRIO_VERSION/$OS/ /" > \
        /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$CRIO_VERSION.list

      mkdir -p /usr/share/keyrings
      curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | gpg --yes --dearmor -o /usr/share/keyrings/libcontainers-archive-keyring.gpg
      curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$CRIO_VERSION/$OS/Release.key | gpg --yes --dearmor -o /usr/share/keyrings/libcontainers-crio-archive-keyring.gpg
      apt-get update
      apt-get install cri-o cri-o-runc -y
EOF

    sudo systemctl daemon-reload
    sudo systemctl enable crio
    sudo systemctl start crio
  SHELL

  # DISABLE SWAP BEFORE KUBELET

  # Install kubelet, kubeadm, kubectl
  config.vm.provision "shell", inline: <<-SHELL
    sudo apt-get update
    sudo apt-get install -y apt-transport-https
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
  SHELL

  # Bootstrap cluster
  config.vm.provision "shell", inline: <<-SHELL
    export CIDR=10.85.0.0/16
    sudo kubeadm init --pod-network-cidr=$CIDR --cri-socket=unix:///var/run/crio/crio.sock
  SHELL
  
  # Setup KUBECONFIG
  config.vm.provision "shell", inline: <<-SHELL
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    echo "alias k=kubectl" > ~/.bash_aliases
    source ~/.bashrc
  SHELL

  # Untaint node
  config.vm.provision "shell", inline: <<-SHELL
    k taint nodes ubuntu-18 node-role.kubernetes.io/control-plane:NoSchedule-
  SHELL

  # Verify cluster using Sonobuoy
  # config.vm.provision "shell", inline: <<-SHELL
    # mkdir ~/sonobuoy-pkg
    # wget https://github.com/vmware-tanzu/sonobuoy/releases/download/v0.56.16/sonobuoy_0.56.16_linux_amd64.tar.gz -P ~/sonobuoy-pkg/

    # mkdir ~/sonobuoy
    # tar -xvf ~/sonobuoy-pkg/sonobuoy_0.56.16_linux_amd64.tar.gz -C ~/sonobuoy
    # ~/sonobuoy/sonobuoy run --wait
  # SHELL
end
