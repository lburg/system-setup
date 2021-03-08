Vagrant.configure("2") do |config|

    # Pull official archlinux box
    config.vm.box = "archlinux/archlinux"

    # Enable synced folders to sync the playbook to the virtual machine using rsync
    config.vm.synced_folder "ansible", "/vagrant", type: "rsync", rsync__auto: "true"

    # Configure the vagrant provisioner to use a local ansible connection
    config.vm.provision "ansible_local" do |ansible|
        ansible.verbose = "v"
        ansible.limit = "all"
        ansible.playbook = "/vagrant/site.yml"
        ansible.inventory_path = "/vagrant/inventory/hosts.ini"
    end

    # Customize the VM configs
    config.vm.provider "virtualbox" do |vb|
        # Add an optical drive to install guest additions
        vb.customize ["storageattach", :id,
                      "--storagectl", "IDE Controller",
                      "--port", "0", "--device", "1",
                      "--type", "dvddrive",
                      "--medium", "emptydrive"]
        # increase virtual video ram
        vb.customize ["modifyvm", :id, "--vram", "128"]
    end
end
