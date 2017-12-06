# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.box_version = "20171011.0.0"
  config.vm.box_check_update = false
  config.vm.network "forwarded_port", guest: 3000, host: 3000, host_ip: "127.0.0.1"

  # https://groups.google.com/forum/#!topic/vagrant-up/eZljy-bddoI
  config.vm.provider "virtualbox" do |vb|
    vb.customize [ "modifyvm", :id, "--uartmode1", "disconnected" ]
    vb.memory = 2048
  end

  config.vm.provision :shell, inline: <<-SHELL
sudo -u ubuntu -i -- <<EOF

# Enable truly non interactive apt-get installs
export DEBIAN_FRONTEND=noninteractive

echo -e "\ncd /vagrant" >> ~/.bashrc  # ubuntu


curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable"

sudo apt-get update
sudo apt-get install -yq docker-ce

sudo usermod -aG docker \\$(whoami)

wget -qO- https://cli-assets.heroku.com/install-ubuntu.sh | sh

# https://docs.docker.com/compose/install/#install-compose
sudo curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose


cd /vagrant

docker --version
docker-compose --version
heroku --version

echo -e "\n\n"

echo "now you need to 'vagrant ssh' in and 'heroku login'"

echo -e "\n\n"

# https://www.alexkras.com/how-to-copy-one-file-from-vagrant-virtual-machine-to-local-host/
# https://stackoverflow.com/questions/28471542/cant-ssh-to-vagrant-vms-using-the-insecure-private-key-vagrant-1-7-2
echo "you almost certainly want to get your local ssh keys into this vagrant box for convenience: (run in host)"
echo "scp -r -i .vagrant/machines/default/virtualbox/private_key -P 2222 ~/.ssh/id_rsa* ubuntu@127.0.0.1:/home/ubuntu/.ssh/"
echo "if this is a vagrant re-install, you might also need to 'sed -i '' '6d' ~/.ssh/known_hosts' with the appropriate line per the error you get from running the above cmd"

EOF
  SHELL
end
