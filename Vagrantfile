# -*- mode: ruby -*-
# # vi: set ft=ruby :

boxes = {
  ubuntu: "ubuntu/xenial64",
  centos: "centos7.2",
}

distro = :centos # :ubuntu

Vagrant.configure(2) do |config|

  (1..3).each do |i|
    config.vm.define "k8s#{i}" do |s|
      s.ssh.forward_agent = true
      s.vm.box = boxes[distro]
      s.vm.hostname = "k8s#{i}"
      s.vm.provision :shell, path: "scripts/bootstrap_ansible_#{distro.to_s}.sh"
      n = 10 + i
      s.vm.network "private_network", ip: "172.42.42.#{n}", netmask: "255.255.255.0",
        auto_config: true,
        virtualbox__intnet: "k8s-net"
      s.vm.provider "virtualbox" do |v|
        v.name = "k8s#{i}"
        v.memory = 1536
        v.gui = false
      end
    end
  end

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end

end
