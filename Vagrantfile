
def controller_exists?(hostname,controllername)
  machineinfo = `VBoxManage showvminfo #{hostname} --machinereadable 2>&1`.split("\n")
  machineinfo
    .select { |line| line.match(/storagecontrollername[^\-]*#{controllername}/) }
    .count > 0
end

Vagrant.configure('2') do |outer_config|
  cluster_disk = ::File.join(::File.dirname(__FILE__), 'cluster_shared.vdi')

  outer_config.vm.provider :virtualbox do |v|
    unless ::File.exists?(cluster_disk)
      v.customize [
        'createhd', '--filename', cluster_disk, '--size', '1024', '--variant', 'Fixed'
      ]
    end
  end

  outer_config.vm.define 'iscsihost' do |config|
    config.vm.box = 'opscode-centos-7.0'
      config.vm.network 'private_network', ip: '33.33.33.20'
      config.vm.hostname = 'iscsihost.opscode.piab'
      config.vm.provider 'virtualbox' do |v|
      v.customize [
        'modifyvm', :id,
        '--name', 'iscsihost.opscode.piab',
        '--memory', '512',
        '--cpus', '1',
        '--natdnshostresolver1', 'on',
        '--usb', 'off',
        '--usbehci', 'off'
      ]
      unless controller_exists?(config.vm.hostname, 'SAS')
        v.customize [
          'storagectl', :id,
          '--name', 'SAS',
          '--add', 'sas',
          '--bootable', 'off'
        ]
      end
      v.customize [
        'storageattach', :id,
        '--storagectl', 'SAS',
        '--port', '0',
        '--device', '0',
        '--nonrotational', 'on',
        '--type', 'hdd',
        '--medium', cluster_disk
      ]
    end
  end

  outer_config.vm.define 'backend0' do |config|
    config.vm.box = 'opscode-centos-7.0'
    config.vm.box_url = 'http://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_centos-7.0_chef-provisionerless.box'
    config.vm.network 'private_network', ip: '33.33.33.21'
    config.vm.hostname = 'backend0.opscode.piab'
    config.vm.provider 'virtualbox' do |v|
      v.customize [
        'modifyvm', :id,
        '--name', 'backend0.opscode.piab',
        '--memory', '512',
        '--cpus', '1',
        '--natdnshostresolver1', 'on',
        '--usb', 'off',
        '--usbehci', 'off'
      ]
    end
  end

  outer_config.vm.define 'backend1' do |config|
    config.vm.box = 'opscode-centos-7.0'
      config.vm.network 'private_network', ip: '33.33.33.22'
      config.vm.hostname = 'backend1.opscode.piab'
      config.vm.provider 'virtualbox' do |v|
      v.customize [
        'modifyvm', :id,
        '--name', 'backend1.opscode.piab',
        '--memory', '512',
        '--cpus', '1',
        '--natdnshostresolver1', 'on',
        '--usb', 'off',
        '--usbehci', 'off'
      ]
    end
  end
end

