# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_NO_PARALLEL'] = 'yes'

Vagrant.configure(2) do |config|

  config.vm.provision "shell", path: "bootstrap.sh"
  
  LoadCount = 2

  # Load Balancer Nodes
  (1..LoadCount).each do |i|
    config.vm.define "k8s-lb#{i}" do |lb|
      lb.vm.box = "generic/ubuntu2004"
      lb.vm.hostname = "k8s-lb#{i}.example.com"
      lb.vm.network "private_network", ip: "172.16.16.14#{i}"
      lb.vm.provider "virtualbox" do |v|
        v.name = "k8s-lb#{i}"
        v.memory = 1024
        v.cpus = 1
      end
    end
  end

  MasterCount = 2

  # Kubernetes Master Nodes
  (1..MasterCount).each do |i|
    config.vm.define "k8s-master#{i}" do |masternode|
      masternode.vm.box = "generic/ubuntu2004"
      masternode.vm.hostname = "k8s-master#{i}.example.com"
      masternode.vm.network "private_network", ip: "172.16.16.15#{i}"
      masternode.vm.provider "virtualbox" do |v|
        v.name = "k8s-master#{i}"
        v.memory = 2048
        v.cpus = 2
      end
    end
  end

  NodeCount = 2

  # Kubernetes Worker Nodes
  (1..NodeCount).each do |i|
    config.vm.define "k8s-worker#{i}" do |workernode|
      workernode.vm.box = "generic/ubuntu2004"
      workernode.vm.hostname = "k8s-worker#{i}.example.com"
      workernode.vm.network "private_network", ip: "172.16.16.16#{i}"
      workernode.vm.provider "virtualbox" do |v|
        v.name = "k8s-worker#{i}"
        v.memory = 1024
        v.cpus = 1
      end
    end
  end

end
