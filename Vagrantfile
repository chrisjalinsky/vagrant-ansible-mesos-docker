VAGRANTFILE_API_VERSION = "2"

base_dir = File.expand_path(File.dirname(__FILE__))

# Cluster, VM and network settings
NETWORK_SUBNET = "10.0.10"
NUM_MASTERS = 1
NUM_SLAVES = 2
MASTER_IPS_START = 10
SLAVE_IPS_START = 100
MASTER_MEMORY = 1024
SLAVE_MEMORY = 1024
MASTER_CPU = 1
SLAVE_CPU = 1
CLUSTER = {}

# Build the cluster
(1..NUM_MASTERS).each do |i|
  CLUSTER["mesos-master#{i}"] = {:ip => "#{NETWORK_SUBNET}.#{MASTER_IPS_START + i}",  :cpus => "#{MASTER_CPU}", :mem => "#{MASTER_MEMORY}"}
end

(1..NUM_SLAVES).each do |i|
  CLUSTER["mesos-slave#{i}"] = {:ip => "#{NETWORK_SUBNET}.#{SLAVE_IPS_START + i}",  :cpus => "#{SLAVE_CPU}", :mem => "#{SLAVE_MEMORY}"}
end

ansible_provision = proc do |ansible|
  # Note: Can't do ranges like ans[0-2] in groups because
  # these aren't supported by Vagrant, see
  # https://github.com/mitchellh/vagrant/issues/3539
  #  ansible.groups = {
  #    "local" => ["localhost"],
  #    "mesos-masters"  => (1..NUM_SLAVES).map { |i| CLUSTER["mesos-master#{i}"][:ip] },
  #    "mesos-slaves" => (1..NUM_SLAVES).map { |i| CLUSTER["mesos-slave#{i}"][:ip] },
  #    "vagrant:children" => ["mesos-masters","mesos-slaves"]
  #  }
  ansible.verbose = "v"
  ansible.inventory_path = base_dir + "/inventory/vagrant"
  ansible.playbook = base_dir + "/cluster.yml"
  ansible.limit = "#{info[:ip]}" # Ansible hosts are identified by ip
end


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :machine
    config.cache.enable :apt
  end

  CLUSTER.each do |hostname, info|

    config.vm.define hostname do |cfg|

      cfg.vm.provider :virtualbox do |vb, override|
        override.vm.box = "trusty64"
        override.vm.box_url = "https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box"
        override.vm.network :private_network, ip: "#{info[:ip]}"
        override.vm.hostname = hostname

        vb.name = 'vagrant-mesos-' + hostname
        vb.customize ["modifyvm", :id, "--memory", info[:mem], "--cpus", info[:cpus], "--hwvirtex", "on" ]
      end

      # provision nodes with ansible
      cfg.vm.provision :ansible do |ansible|
        ansible.verbose = "v"

        ansible.inventory_path = base_dir + "/inventory/vagrant"
        ansible.playbook = base_dir + "/cluster.yml"
        ansible.limit = "#{info[:ip]}" # Ansible hosts are identified by ip
      end

    end

  end

end