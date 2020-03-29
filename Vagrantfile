class VagrantPlugins::ProviderVirtualBox::Action::SetName
    alias_method :original_call, :call
    def call(env)
        ui = env[:ui]
        driver = env[:machine].provider.driver

        second_disk_name = "sdb"
        vm_name = env[:machine].name
        data_dir = env[:machine].data_dir
        vm_uuid = driver.instance_eval { @uuid }
        ui.info "vm_uuid: '#{vm_uuid}'"

        def unquoted(s)
          s.gsub(/\A"(.*)"\Z/,'\1')
        end

        def is_disk(driver, disk_uuid)
          begin
            driver.execute("showmediuminfo", 'disk', disk_uuid)
            true
          rescue
            false
          end
        end

        def identify_disks(driver, vm_uuid)
          vminfo = get_vminfo(driver, vm_uuid)
          disks = []
          disk_keys = vminfo.keys.select { |k| k =~ /-ImageUUID-/ }
          disk_keys.each do |key|
            disk_uuid = vminfo[key]
            if is_disk(driver, disk_uuid)
              disk_name = key.gsub(/-ImageUUID-/,'-')
              disk_file = vminfo[disk_name]
              disks << {
                uuid: disk_uuid,
                name: disk_name,
                file: disk_file
              }
            end
          end
          disks
        end

        def get_vminfo(driver, vm_uuid)
          vminfo = {}
          driver.execute('showvminfo', vm_uuid, '--machinereadable', retryable: true).split("\n").each do |line|
            parts = line.partition('=')
            key = unquoted(parts.first)
            value = unquoted(parts.last)
            vminfo[key] = value
          end
          vminfo
        end

        def get_disk_size(driver, disk)
          size = nil
          driver.execute("showmediuminfo", disk[:file]).each_line do |line|
            if line =~ /Capacity:\s+([0-9]+)\s+MB/
              size = $1.to_i
            end
          end
          size
        end

        disks = identify_disks(driver, vm_uuid)
        target_disk = disks.first    # TODO Shouldn't assume that the first disk is the one we want to check
        target_disk_size = get_disk_size(driver, target_disk)
        target_controller_name = target_disk[:name].gsub(/-([0-9]+)-([0-9]+)/,'')
        target_controller_device = target_disk[:name].split("-", -2)[1]
        target_controller_port = target_disk[:name].split("-").last.to_i
        if target_controller_name.match("IDE")
          target_controller_type = 'ide'
        elsif target_controller_name.match("SAS")
          target_controller_type = 'sas'
        elsif target_controller_name.match("SATA")
          target_controller_type = 'sata'
        else
          target_controller_type = 'unknown'
        end

        ui.info "target_disk_uuid: '#{target_disk[:uuid]}'"
        ui.info "target_disk_name: '#{target_disk[:name]}'"
        ui.info "target_disk_size: '#{target_disk_size}'"
        ui.info "target_controller_name: '#{target_controller_name}'"
        ui.info "target_controller_device: '#{target_controller_device}'"
        ui.info "target_controller_port: '#{target_controller_port}'"
        ui.info "target_controller_type: '#{target_controller_type}'"

        ui.info "Create '#{data_dir}/#{vm_name}_#{second_disk_name}\.vmdk'..."
        driver.execute("createhd",
            "--filename", "#{data_dir}" + "/" + "#{vm_name}_#{second_disk_name}" + ".vmdk",
            "--size", "#{target_disk_size}",
            "--format", "VMDK"
        )

        target_controller_port += 1

        ui.info "Attach '#{data_dir}/#{vm_name}_#{second_disk_name}\.vmdk' to '#{target_controller_name}' port '#{target_controller_port}'..."
        driver.execute(
            "storageattach", vm_uuid,
            "--storagectl", "#{target_controller_name}",
            "--port", "#{target_controller_port}",
            "--device", "#{target_controller_device}",
            "--type", "hdd",
            "--medium", "#{data_dir}" + "/" + "#{vm_name}_#{second_disk_name}" + ".vmdk"
        )

        original_call(env)
    end
end

Vagrant.configure("2") do |config|
  config.vm.box = "aairey/proxmoxve"
  config.vm.box_version = "0.1.0"
  (1..1).each do |i|
    config.vm.define "host#{i}" do |node|
      node.vm.hostname = "host#{i}"
      node.vm.network "private_network", ip: "192.0.2.#{10+i}"
      config.vm.provider :virtualbox do |vb|
        vb.name = "host#{i}"
        vb.cpus = 1
        vb.memory = "2048"
#        vb.customize ["storagectl", :id, "--add", "sas", "--name", "SAS Controller" , "--portcount", 2, "--hostiocache", "on"]
#        vb.customize ['createhd', '--filename', "host_#{i}b.vdi", '--size', 8192]
#        vb.customize ['storageattach', :id, '--storagectl', "SAS Controller", '--port', 1, '--type', 'hdd', '--medium', "host_#{i}a.vdi" ]
#        vb.customize ['storageattach', :id, '--storagectl', "SAS Controller", '--port', 0, '--type', 'hdd', '--medium', "host_#{i}b.vdi" ]
      end
    end
  end
end
