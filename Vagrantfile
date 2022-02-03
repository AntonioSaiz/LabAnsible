$install_docker_script = <<SCRIPT
echo Installing Docker...
curl -sSL https://get.docker.com/ | sh
usermod -aG docker vagrant
SCRIPT

$master_script = <<SCRIPT
echo Swarm Init...
docker swarm init --listen-addr 10.100.199.200:2377 --advertise-addr 10.100.199.200:2377
docker swarm join-token --quiet labworker > /mnt/nfs/labworker_token
curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
docker stack deploy -c portainer-agent-stack.yml portainer
SCRIPT

$labworker_script = <<SCRIPT
echo Swarm Join...
docker swarm join --token $(cat /mnt/nfs/labworker_token) 10.100.199.200:2377
SCRIPT

$master_nfs = <<SCRIPT
apt-get update
apt install -y nfs-kernel-server
mkdir /mnt/nfs/
chown nobody:nogroup /mnt/nfs
echo "/mnt/nfs 10.100.199.201(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports
echo "/mnt/nfs 10.100.199.202(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports
systemctl restart nfs-kernel-server
SCRIPT

$labworker_nfs = <<SCRIPT
sudo apt update
sudo apt install -y nfs-common
mkdir /mnt/nfs/
echo "10.100.199.200:/mnt/nfs/           /mnt/nfs/   nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" >> /etc/fstab
mount -a
SCRIPT


Vagrant.configure('2') do |config|

  vm_box = 'ubuntu/focal64'

  config.vm.boot_timeout = 300
  config.vm.define :master, primary: true  do |master|
    master.vm.box = vm_box
    master.vm.box_check_update = true
    master.vm.disk :disk, size: "100GB", primary: true
    master.vm.network :private_network, ip: "10.100.199.200"
    master.vm.network :forwarded_port, guest: 8080, host: 8080
    master.vm.network :forwarded_port, guest: 9443, host: 9443
    master.vm.network :forwarded_port, guest: 9000, host: 9000
    master.vm.hostname = "master"
    master.vm.provision "shell", inline: $master_nfs, privileged: true
    master.vm.provision "shell", inline: $install_docker_script, privileged: true
    master.vm.provision "shell", inline: $master_script, privileged: true
    master.vm.provider "virtualbox" do |vb|
      vb.name = "master"
      vb.memory = "2048"
    end
  end

  (1..2).each do |i|
    config.vm.define "labworker0#{i}" do |labworker|
      labworker.vm.box = vm_box
      labworker.vm.box_check_update = true
      labworker.vm.disk :disk, size: "50GB", primary: true
      labworker.vm.network :private_network, ip: "10.100.199.20#{i}"
      labworker.vm.hostname = "labworker0#{i}"
      labworker.vm.provision "shell", inline: $labworker_nfs, privileged: true
      labworker.vm.provision "shell", inline: $install_docker_script, privileged: true
      labworker.vm.provision "shell", inline: $labworker_script, privileged: true
      labworker.vm.provider "virtualbox" do |vb|
        vb.name = "labworker0#{i}"
        vb.memory = "2048"
      end
    end
  end

end