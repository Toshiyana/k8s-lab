Vagrant.configure("2") do |config|
  # 共通設定: Ubuntu 22.04
  config.vm.box = "ubuntu/jammy64"
  config.vm.boot_timeout = 600
  
  nodes = [
    { name: "lb",       ip: "192.168.56.10", mem: 512,  cpu: 1 },
    { name: "master-1", ip: "192.168.56.11", mem: 2048, cpu: 2 },
    { name: "master-2", ip: "192.168.56.12", mem: 2048, cpu: 2 },
    { name: "worker-1", ip: "192.168.56.21", mem: 2048, cpu: 2 }
  ]

  nodes.each do |node|
    config.vm.define node[:name] do |node_config|
      node_config.vm.hostname = node[:name]
      node_config.vm.network "private_network", ip: node[:ip]
      node_config.vm.provider "virtualbox" do |vb|
        vb.memory = node[:mem]
        vb.cpus = node[:cpu]
      end
      # OSの基本設定（スワップ無効化など）を自動実行
      node_config.vm.provision "shell", path: "setup.sh"
    end
  end
end