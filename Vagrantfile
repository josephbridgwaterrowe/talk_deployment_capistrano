# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.ssh.insert_key = false
  config.ssh.private_key_path = '~/.vagrant.d/insecure_private_key'
  config.vm.box = 'ubuntu/trusty64'

  config.vm.define 'web01' do |web|
    web.vm.network "private_network", ip: "192.168.33.10"
  end

  config.vm.define 'app01' do |app|
    app.vm.network 'private_network', :ip => '192.168.33.11'
  end

  config.vm.define 'app02' do |app|
    app.vm.network 'private_network', :ip => '192.168.33.12'
  end

  config.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
  end
end
