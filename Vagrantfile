# -*- mode: ruby -*-
# vi: set ft=ruby :
#

require 'json'
require 'ipaddress'

settings = YAML.load_file 'provisioning/vm-config.yml'

# numero di nodi che compone il cluster galera, deve essere almeno di 3 elementi
$db_num = settings['db_num'] ? settings['db_num'] : 3

# numero di load balancer verso il cluster db - forzato a 2 in HA A/P
$dbb_num = 2

# numero di nodi che compone il cluster galera, deve essere almeno di 3 elementi
$wp_num = settings['wp_num'] ? settings['wp_num'] : 3

# numero di load balancer verso i nodi fe - forzato a 2 in HA A/P
$wpb_num = 2

# rete su cui assegnare ip alle macchine
$vm_net = settings['vm_net'] ? settings['vm_net'] : '192.168.122.0/24'

# cpu per macchina
$vm_cpu = settings['vm_cpu'] ? settings['vm_cpu'] : 1

# ram per macchina
$vm_ram = settings['vm_ram'] ? settings['vm_ram'] : 256

Vagrant.configure("2") do |config|

  $dbs_ip = {}
  (1..$db_num).each do |i|
          ip = IPAddress($vm_net)
          ip[3] = ip[3] + 200 + i
          $dbs_ip["db#{i}"] = ip.address
  end

  $dbbs_ip = {}
  (1..$dbb_num).each do |i|
          ip = IPAddress($vm_net)
          ip[3] = ip[3] + 150 + i
          $dbbs_ip["dbb#{i}"] = ip.address
  end

  ip = IPAddress($vm_net)
  ip[3] = ip[3] + 200
  $db_vip = ip.address

  ip = IPAddress($vm_net)
  ip[3] = ip[3] + 100
  $minikube_ip = ip.address

  (1..1).each do |i|
  	config.vm.define "minikube" do |node| #VM name
  		node.vm.hostname = "minikube"
  		node.vm.box = "debian/buster64"
  		node.vm.provider 'libvirt' do |v|
  			v.memory = 2048
  			v.cpus = 2
  		end
  		node.vm.network 'private_network', ip: $minikube_ip
  	end
  end

  (1..$dbb_num).each do |i|
          config.vm.define "dbb#{i}" do |node|
                  node.vm.provider 'libvirt' do |v|
                          v.memory = $vm_ram
                          v.cpus = $vm_cpu
                  end
                  node.vm.box = 'debian/buster64'
                  node.vm.hostname = "dbb#{i}"
                  node.vm.network 'private_network', ip: $dbbs_ip[node.vm.hostname] 
          end
  end


  (1..$db_num).each do |i|
          config.vm.define "db#{i}" do |node|
                  node.vm.provider 'libvirt' do |v|
                          v.memory = $vm_ram
                          v.cpus = $vm_cpu
                  end
                  node.vm.box = 'debian/buster64'
                  node.vm.hostname = "db#{i}"
                  node.vm.network 'private_network', ip: $dbs_ip[node.vm.hostname] 

            	  if i == $db_num
            	          config.vm.provision 'ansible' do |ansible|
            	                  ansible.playbook = 'provisioning/playbook.yml'  
            	                  ansible.limit = 'all'
            	                  ansible.groups = {
            	                          'dbs' => ["db[1:#{$wp_num}]"],
            	                          'dbbs' => ["dbb[1:#{$dbb_num}]"],
					  'minikube' => ["minikube"],
            	                          'all:vars' => { 
            	                                  'db_num' => "#{$db_num}",
            	                                  'wp_num' => "#{$wp_num}",
            	                                  'dbs_ip' => "#{$dbs_ip.to_json}",
            	                                  'db_vip' => "#{$db_vip}",
						  'minikube_ip' => "#{$minikube_ip}",
            	                                  'dbbs_ip' => "#{$dbbs_ip.to_json}"
            	                          }
            	                  }
            	          end
            	  end
          end
  end
end
