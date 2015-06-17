# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  $provisioning_midonet_cluster = <<EOF
#
# Disable Network Manager
#
systemctl mask NetworkManager
systemctl stop NetworkManager
cat > /etc/sysconfig/network-scripts/ifcfg-enp0s8 << MIDO_EOF
DEVICE=enp0s8
BOOTPROTO=dhcp
NM_CONTROLLED=no
MIDO_EOF
cat > /etc/sysconfig/network-scripts/ifcfg-enp0s9 << MIDO_EOF
DEVICE=enp0s9
BOOTPROTO=dhcp
NM_CONTROLLED=no
MIDO_EOF
systemctl enable network.service
systemctl start network.service


#
# Repos
#
rpm -ivh http://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm
yum makecache fast

# packstack
yum install -y ntpdate
ntpdate pool.ntp.org
yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-juno/rdo-release-juno-1.noarch.rpm
yum install -y epel-release

systemctl stop firewalld
systemctl mask firewalld
yum install -y iptables-services
systemctl enable iptables

# Midonet
cat >> /etc/yum.repos.d/midonet.repo << EOF_MIDO
[midonet]
name=MidoNet
baseurl=http://bcn4.bcn.midokura.com:8081/artifactory/midonet/el7/master/nightly/all/noarch/
enabled=1
gpgcheck=1
gpgkey=http://bcn4.bcn.midokura.com:8081/artifactory/api/gpg/key/public

[midonet-openstack-integration]
name=MidoNet OpenStack Integration
baseurl=http://bcn4.bcn.midokura.com:8081/artifactory/midonet/el7/master/nightly/juno/noarch/
enabled=1
gpgcheck=1
gpgkey=http://bcn4.bcn.midokura.com:8081/artifactory/api/gpg/key/public

[midonet-misc]
name=MidoNet 3rd Party Tools and Libraries
baseurl=http://bcn4.bcn.midokura.com:8081/artifactory/midonet/el7/thirdparty/stable/all/x86_64/
enabled=1
gpgcheck=1
gpgkey=http://bcn4.bcn.midokura.com:8081/artifactory/api/gpg/key/public
EOF_MIDO

# Cassandra
cat >> /etc/yum.repos.d/cassandra.repo << EOF_MIDO
[datastax]
name= DataStax Repo for Apache Cassandra
baseurl=http://rpm.datastax.com/community
enabled=1
gpgcheck=0
EOF_MIDO


#
# Updating and installing dependencies
#

yum update -y

# Tools
yum install -y augeas crudini screen wget


#
# Packstack
#
#

wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
#pip install  oslo.utils oslo.config

IP=192.168.124.185
yum install -y openstack-packstack
packstack --install-hosts=$IP \
 --nagios-install=n \
 --os-swift-install=n \
 --os-ceilometer-install=n \
 --os-cinder-install=y \
 --os-glance-install=y \
 --os-heat-install=n \
 --os-horizon-install=y \
 --os-nova-install=y \
 --provision-demo=n

rc=$?; if [[ $rc != 0 ]]; then exit $rc; fi

#
# Disable unneeded remaining services
#
yum remove -y openstack-neutron-openvswitch
systemctl stop openvswitch
systemctl mask openvswitch
systemctl stop neutron-l3-agent
systemctl mask neutron-l3-agent


ADMIN_TOKEN=$(crudini --get /etc/keystone/keystone.conf DEFAULT admin_token)
ADMIN_PASSWORD=$(grep "OS_PASSWORD" /root/keystonerc_admin | sed -e 's/"//g' | cut -f2 -d'=')
NEUTRON_DBPASS=$(crudini --get /root/packstack-answers-* general CONFIG_NEUTRON_DB_PW)

cp /root/keystonerc_admin /home/vagrant/
chown vagrant:vagrant /root/keystonerc_admin /home/vagrant/keystonerc_admin


#
# Zookeeper
#

yum install -y java-1.7.0-openjdk-headless zookeeper

# Zookeeper expects the JRE to be found in the /usr/java/default/bin/ directory
# so if it is in a different location, you must create a symbolic link pointing
# to that location. To do so run the 2 following commands:

mkdir -p /usr/java/default/bin/
ln -s /usr/lib/jvm/jre-1.7.0-openjdk/bin/java /usr/java/default/bin/java

# Next we need to create the zookeeper data directory and assign permissions:

mkdir /var/lib/zookeeper/data
chmod 777 /var/lib/zookeeper/data

# Now we can edit the Zookeeper configuration file. We need to add the servers
# (in a prod installation you would have more than one zookeeper server in a
# cluster. For this example we are only using one. ). Edit the Zookeeper config
# file at /etc/zookeeper/zoo.cfg and add the following to the bottom of the
# file:

echo "server.1=$IP:2888:3888" >> /etc/zookeeper/zoo.cfg

# We need to set the Zookeeper ID on this server:

echo 1 > /var/lib/zookeeper/data/myid

systemctl enable zookeeper.service
systemctl start zookeeper.service

#
# Cassandra
#

yum install -y dsc20

sed -i -e "s/cluster_name:.*/cluster_name: 'midonet'/" /etc/cassandra/conf/cassandra.yaml

sed -i -e "s/listen_address:.*/listen_address: $IP/" /etc/cassandra/conf/cassandra.yaml

sed -i -e "s/seeds:.*/seeds: \"$IP\"/" /etc/cassandra/conf/cassandra.yaml
sed -i -e "s/rpc_address:.*/rpc_address: $IP/" /etc/cassandra/conf/cassandra.yaml

rm -rf /var/lib/cassandra/data/system

systemctl enable cassandra.service
systemctl start cassandra.service


#
# Midolman
#

yum install -y midolman
systemctl enable midolman.service
systemctl start midolman.service

#
# Midonet client
#
yum install -y python-midonetclient

#
# Midonet-cluster
#
#

echo "zookeeper.use_new_stack: true" | mn-conf set -t default
echo "cluster.rest_api.http_port: 8081" | mn-conf set -t default

yum install -y midonet-cluster
systemctl enable midonet-cluster.service
systemctl start midonet-cluster.service

systemctl restart midolman

#
# Midonet-cli
#
#

yum install -y python-midonetclient
cat > ~/.midonetrc << MIDO_EOF
[cli]
api_url = http://$IP:8081/midonet-api
username = admin
password = $ADMIN_PASSWORD
project_id = admin
MIDO_EOF

cp /root/.midonetrc /home/vagrant/

# Create a tunnel zone
TUNNEL_ZONE=$(midonet-cli -e tunnel-zone create name gre type gre)
HOST_UUID=$(midonet-cli -e list host | awk '{print $2;}')
midonet-cli -e tunnel-zone $TUNNEL_ZONE add member host $HOST_UUID address $IP


#
# Keystone Integration
#

source ~/keystonerc_admin
keystone service-create --name midonet --type midonet --description "Midonet API Service"

keystone user-create --name midonet --pass midonet --tenant admin
keystone user-role-add --user midonet --role admin --tenant admin


#
# Neutron integration
#
yum install -y openstack-neutron python-neutron-plugin-midonet

crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin midonet.neutron.plugin.MidonetPluginV2

mkdir /etc/neutron/plugins/midonet
cat > /etc/neutron/plugins/midonet/midonet.ini << MIDO_EOF
[DATABASE]
sql_connection = mysql://neutron:$NEUTRON_DBPASS@$IP/neutron
sql_max_retries = 100
[MIDONET]
# MidoNet API URL
midonet_uri = http://$IP:8081/midonet-api
# MidoNet administrative user in Keystone
username = midonet
password = midonet
# MidoNet administrative user's tenant
project_id = admin
auth_url = http://$IP:5000/v2.0
MIDO_EOF

rm -f /etc/neutron/plugin.ini
ln -s /etc/neutron/plugins/midonet/midonet.ini /etc/neutron/plugin.ini

# Comment out the service_plugins definitions
crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins neutron.services.firewall.fwaas_plugin.FirewallPlugin
sed -i -e 's/^router_scheduler_driver/#router_scheduler_driver/' /etc/neutron/neutron.conf

# dhcp agent config
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.MidonetInterfaceDriver
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver midonet.neutron.agent.midonet_driver.DhcpNoOpDriver
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata True
crudini --set /etc/neutron/dhcp_agent.ini MIDONET midonet_uri http://$IP:8081/midonet-api
crudini --set /etc/neutron/dhcp_agent.ini MIDONET username midonet
crudini --set /etc/neutron/dhcp_agent.ini MIDONET password midonet


systemctl restart neutron-server
systemctl restart neutron-dhcp-agent
systemctl restart neutron-metadata-agent

#
# Create internal network for usage on the instances
# external network
neutron net-create ext-net --shared --router:external=True
neutron subnet-create ext-net --name ext-subnet --allocation-pool start=200.200.200.2,end=200.200.200.254 --disable-dhcp --gateway 200.200.200.1 200.200.200.0/24

# Fake uplink
# We are going to create the following topology to allow the VMs reach external
# networks

ip link add type veth
ip link set dev veth0 up
ip link set dev veth1 up
brctl addbr uplinkbridge
brctl addif uplinkbridge veth0
ip addr add 172.19.0.1/30 dev uplinkbridge
ip link set dev uplinkbridge up
sysctl -w net.ipv4.ip_forward=1
ip route add 200.200.200.0/24 via 172.19.0.2

router=$(midonet-cli -e router list | awk '{print $2}')
port=$(midonet-cli -e router $router add port address 172.19.0.2 net 172.19.0.0/30)
midonet-cli -e router $router add route src 0.0.0.0/0 dst 0.0.0.0/0 type normal port router $router port $port gw 172.19.0.1
host=$(midonet-cli -e host list | awk '{print $2}')
midonet-cli -e host $host add binding port router $router port $port interface veth1

EOF
  config.vm.box = "centos7"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.define :midonet_cluster do |midonet_cluster|
      midonet_cluster.vm.hostname = "nova-docker.local"
      midonet_cluster.vm.network :private_network, ip: "192.168.124.185"
      midonet_cluster.vm.network :private_network, ip: "192.168.124.186"
      midonet_cluster.vm.provision "shell",
    inline: $provisioning_midonet_cluster
      midonet_cluster.vm.provider :virtualbox do |vb|
          vb.memory = 8192
          vb.cpus = 2
      end
  end
end
