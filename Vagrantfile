Vagrant.require_version ">= 1.8.0"
ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'
Vagrant.configure(2) do |config|
  
  config.vm.box = "ubuntu/focal64"
  config.vm.network "forwarded_port", guest: 3000, host: 3000

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
