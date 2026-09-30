# Vagrantfile — стенд(PXE-сервер для автоустановки Ubuntu 24.04)
#
# Фактически ДЗ выполнено на виртуалках Proxmox:
#   pxeserver = VMID 510, 192.168.1.155 (vmbr0) + 10.0.0.20 (PXE-сеть vmbr-net14)
#   pxeclient = VMID 511, только PXE-сеть vmbr-net14, boot order = network
Vagrant.configure("2") do |config|
  # ---- PXE-сервер (DHCP + TFTP + HTTP + seed-файлы) ----
  config.vm.define "pxeserver" do |server|
    server.vm.box = "bento/ubuntu-24.04"
    server.vm.host_name = "pxeserver"
    server.vm.network "forwarded_port", guest: 80, host: 8080
    # Внутренняя PXE-сеть
    server.vm.network :private_network,
      ip: "10.0.0.20",
      virtualbox__intnet: "pxenet"
    # Служебный интерфейс для Ansible (доступ с хоста)
    server.vm.network :private_network, ip: "192.168.50.10", adapter: 3
    server.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end

    server.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/provision.yml"
      ansible.inventory_path = "ansible/hosts"
      ansible.host_key_checking = "false"
      ansible.limit = "all"
    end
  end

  # ---- PXE-клиент (грузится по сети, без диска с ОС) ----
  config.vm.define "pxeclient" do |pxeclient|
    pxeclient.vm.box = "bento/ubuntu-24.04"
    pxeclient.vm.host_name = "pxeclient"
    pxeclient.vm.network :private_network, ip: "10.0.0.21"
    pxeclient.vm.provider :virtualbox do |vb|
      vb.memory = "4096"
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize [
        "modifyvm", :id,
        "--nic1", "intnet",
        "--intnet1", "pxenet",
        "--nic2", "nat",
        "--boot1", "net",
        "--boot2", "none",
        "--boot3", "none",
        "--boot4", "none"
      ]
    end
  end
end
