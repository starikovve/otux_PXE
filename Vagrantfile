# -*- mode: ruby -*-
# vi: set ft=ruby :
# export VAGRANT_EXPERIMENTAL="disks"

Vagrant.configure("2") do |config|

  config.vm.define "pxeserver" do |server|
    server.vm.box = 'ubuntu/jammy64'
    server.vm.disk :disk, size: "15GB", name: "extra_storage1"

    server.vm.hostname = 'pxeserver'
    server.vm.network :private_network,
                       ip: "10.0.0.20",
                       virtualbox__intnet: 'pxenet'

    server.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end

    server.vm.provision "shell",
      name: "Setup PXE server",
      path: "setup_pxe.sh"
  end

  config.vm.define "pxeclient" do |pxeclient|
    pxeclient.vm.box = 'ubuntu/jammy64'
    pxeclient.vm.hostname = 'pxeclient'
    pxeclient.vm.network :private_network, ip: "10.0.0.21",
                          virtualbox__intnet: 'pxenet'

    pxeclient.vm.provider :virtualbox do |vb|
      vb.memory = "2048"
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize [
          'modifyvm', :id,
          '--nic1', 'intnet',
          '--intnet1', 'pxenet',
          '--nic2', 'nat',
          '--boot1', 'net',
          '--boot2', 'none',
          '--boot3', 'none',
          '--boot4', 'none'
        ]
    end
  end

end