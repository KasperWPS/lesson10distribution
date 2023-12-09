# -*- mode: ruby -*- 
# vi: set ft=ruby : vsa
Vagrant.configure(2) do |config|
        config.vm.box = "centos/8"
        ip_repo= "10.111.177.150"
        ip_nginx = "10.111.177.160"
        #config.vm.box_version = "2004.01"
        config.vm.provider "virtualbox" do |v|
        v.memory = 256
        v.cpus = 1
    end
    config.vm.define "repo" do |repo|
        repo.vm.network "private_network", ip: ip_repo,  virtualbox__intnet: "net1"
        repo.vm.hostname = "ropository"
        repo.vm.provision "shell", path: "scripts/repo.sh"
    end
    config.vm.define "websrv" do |websrv|
        websrv.vm.network "private_network", ip: ip_nginx,  virtualbox__intnet: "net1"
        websrv.vm.hostname = "websrv"
        websrv.vm.provision "shell", path: "scripts/websrv.sh", args: [ip_repo]
    end
end
