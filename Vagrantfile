# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'ipaddr'
require 'yaml'

inventory_template = %{
%nodes%

[mons]
%mons%

[osds]
%osds%

[rgws]
%rgws%

[all:vars]
%vars%

[mons:vars]
%mons_vars%

[osds:vars]
%osds_vars%
}

ceph_config = YAML.load_file('config.yml') rescue YAML.load_file('config.yml.example')

# generate IP Addresses
public_network = IPAddr.new(ceph_config['ceph']['vars']['osds']['public_network'])
cluster_network = IPAddr.new(ceph_config['ceph']['vars']['osds']['cluster_network'])
cluster_size = ceph_config["ceph"]["nodes"]
mon_count = ceph_config['ceph']['mons']
osd_per_node = ceph_config['ceph']['osd_per_node']

public_ips = cluster_size.times.to_a.map { |i| (public_network | 10+i).to_s }
cluster_ips = cluster_size.times.to_a.map { |i| (cluster_network | 10+i).to_s }
devices = ('/dev/sdb'..'/dev/sdz').to_a[0..osd_per_node-1]
ceph_config['ceph']['vars']['osds']['devices'] = devices

nodes_def = public_ips.each_with_index.map { |ip,i| "ceph#{i} ansible_ssh_host=#{ip}" }.join("\n")
mon_def = mon_count.times.to_a.map { |i| "ceph#{i}" }.join("\n")
osd_def = cluster_size.times.to_a.map { |i| "ceph#{i}" }.join("\n")
rgw_def = 'ceph0'
var_def = ceph_config['ceph']['vars']['all'].map{ |k,v| "#{k}=#{v}" }.join("\n")
monsvar_def = ceph_config['ceph']['vars']['mons'].map{ |k,v| "#{k}=#{v}" }.join("\n")
osdsvar_def = ceph_config['ceph']['vars']['osds'].map{ |k,v| "#{k}=#{v}" }.join("\n")

inventory = inventory_template.gsub('%nodes%', nodes_def)
inventory = inventory.gsub('%mons%', mon_def)
inventory = inventory.gsub('%osds%', osd_def)
inventory = inventory.gsub('%rgws%', rgw_def)
inventory = inventory.gsub('%vars%',  var_def)
inventory = inventory.gsub('%mons_vars%', monsvar_def)
inventory = inventory.gsub('%osds_vars%', osdsvar_def)

File.write 'inventory', inventory

Vagrant.configure(2) do |config|

  config.ssh.insert_key = false
  config.vm.box = "ubuntu/trusty64"

  cluster_size.times do |i|
    config.vm.define "ceph#{i}" do |node|
      node.vm.hostname = "ceph#{i}"
      node.vm.network :private_network, ip: public_ips[i]
      node.vm.network :private_network, ip: cluster_ips[i]
      node.vm.provider :virtualbox do |vb|
        vb.cpus = ceph_config['vm']['cpus']
        vb.memory = ceph_config['vm']['memory']
        osd_per_node.times do |j|
          vb.customize ['createhd', '--filename', "osd-#{i}-#{j}", '--size', "#{ceph_config['ceph']['osd_size']}"]
          vb.customize ['storageattach', :id, '--storagectl', 'SATAController', '--port', 3+j, '--device', 0, '--type', 'hdd', '--medium', "osd-#{i}-#{j}.vdi"]
        end
      end
    end
  end

end
