# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile

require 'yaml'
# Contenu YAML directement dans le Vagrantfile
yaml_parameters = <<-YAML
---
# cluster_name is used to group the nodes in a folder within VirtualBox:
cluster_name: k8s-cluster
# network et autres paramètres sont inchangés
network:
  control_ip: 192.168.99.10
  dns_servers:
    - 8.8.8.8
    - 1.1.1.1
  pod_cidr: 172.16.1.0/16
  service_cidr: 172.17.1.0/18
nodes:
  control:
    cpu: 2
    memory: 2048
  workers:
    count: 5  # Changer à 5 pour avoir 5 nœuds workers
    cpu: 1
    memory: 2048
software:
  box: eazytrainingfr/ubuntu
  calico: 3.26.0
  dashboard: 2.7.0
  kubernetes: 1.32.0-*
  os: xUbuntu_24.04
YAML

# Parser le contenu YAML en Ruby
settings = YAML.load(yaml_parameters)

IP_SECTIONS = settings["network"]["control_ip"].match(/^([0-9.]+\.)([^.]+)$/)
# First 3 octets including the trailing dot:
IP_NW = IP_SECTIONS.captures[0]
# Last octet excluding all dots:
IP_START = Integer(IP_SECTIONS.captures[1])
NUM_WORKER_NODES = settings["nodes"]["workers"]["count"]

Vagrant.configure("2") do |config|
  config.vm.provision "shell", env: { "IP_NW" => IP_NW, "IP_START" => IP_START, "NUM_WORKER_NODES" => NUM_WORKER_NODES }, inline: <<-SHELL
      apt-get update -y
      echo "$IP_NW$((IP_START)) controlplane" >> /etc/hosts
      for i in `seq 1 ${NUM_WORKER_NODES}`; do
        echo "$IP_NW$((IP_START+i)) node0${i}" >> /etc/hosts
      done
  SHELL

  if `uname -m`.strip == "aarch64"
    config.vm.box = settings["software"]["box"] + "-arm64"
  else
    config.vm.box = settings["software"]["box"]
  end
  config.vm.box_check_update = true

  # 3 nœuds master
  (1..3).each do |i|
    config.vm.define "master#{i}" do |controlplane|
      controlplane.vm.hostname = "master#{i}"
      controlplane.vm.network "private_network", ip: IP_NW + "#{IP_START + i - 1}"
      if settings["shared_folders"]
        settings["shared_folders"].each do |shared_folder|
          controlplane.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
        end
      end
      controlplane.vm.provider "virtualbox" do |vb|
        vb.cpus = settings["nodes"]["control"]["cpu"]
        vb.memory = settings["nodes"]["control"]["memory"]
        if settings["cluster_name"] and settings["cluster_name"] != ""
          vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
        end
      end
      controlplane.vm.provision "shell",
        env: {
          "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
          "ENVIRONMENT" => settings["environment"],
          "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
          "KUBERNETES_VERSION_SHORT" => settings["software"]["kubernetes"][0..3],
          "OS" => settings["software"]["os"],
          "CALICO_VERSION" => settings["software"]["calico"],
          "CONTROL_IP" => settings["network"]["control_ip"],
          "POD_CIDR" => settings["network"]["pod_cidr"],
          "SERVICE_CIDR" => settings["network"]["service_cidr"],
          "ENABLE_ZSH" => ENV["ENABLE_ZSH"]      
        },
        path: "install_kubernetes.sh",
        args: "controlplane"
    end
  end

  # 5 nœuds worker
  (1..NUM_WORKER_NODES).each do |i|  # Changer de NUM_WORKER_NODES à 5
    config.vm.define "worker#{i}" do |node|
      node.vm.hostname = "worker#{i}"
      node.vm.network "private_network", ip: IP_NW + "#{IP_START + 3 + i - 1}"  # Commence après les nœuds master
      if settings["shared_folders"]
        settings["shared_folders"].each do |shared_folder|
          node.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
        end
      end
      node.vm.provider "virtualbox" do |vb|
        vb.cpus = settings["nodes"]["workers"]["cpu"]
        vb.memory = settings["nodes"]["workers"]["memory"]
        if settings["cluster_name"] and settings["cluster_name"] != ""
          vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
        end
      end
      node.vm.provision "shell",
        env: {
          "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
          "ENVIRONMENT" => settings["environment"],
          "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
          "KUBERNETES_VERSION_SHORT" => settings["software"]["kubernetes"][0..3],
          "OS" => settings["software"]["os"],
          "ENABLE_ZSH" => ENV["ENABLE_ZSH"]      
        },
        path: "install_kubernetes.sh",
        args: "node"
    end
  end
end
