#OpenStack Newton Deploy Note

##Environment
- Controller Node:
	- OS: Centos 7
	- External IP: 192.168.10.51
	- Management IP: 10.10.0.51
	- OPS Project: Keystone, Glance, Nova, Neutron (L2), Cinder, Horizon
- Compute Node 01:
	- OS: Centos 7
	- External IP: 192.168.10.x
	- Management IP: 10.10.0.x
	- OPS Project: Nova (compute)
 - Compute Node 02:
	- OS: Centos 7
	- External IP: 192.168.10.x
	- Management IP: 10.10.0.x
	- OPS Project: Nova (compute)
- Ceph Admin Node:
	- OS: Centos 7
	- Management IP: 10.10.0.10

## Deployment
###Controller Node
####Environment Setup
#####NTP
Install the packages:
```
yum -y install chrony
```
Edit the /etc/chrony.conf:
```
server 10.10.0.10 iburst

```
Start the NTP service:
```
systemctl enable chronyd.service
systemctl start chronyd.service
```
Verify:
```
chronyc sources
```

#####OpenStack Newton Packages
Update and Upgrade Repo
```
yum install -y centos-release-openstack-newton
yum update -y && yum upgrade -y
```
Install the OpenStack client:
```
yum install python-openstackclient
```

#####SQL Database
Install the packages:
```
yum install -y mariadb mariadb-server python2-PyMySQL
```
Edit `/etc/my.cnf.d/openstack.cnf`:
```
[mysqld]
bind-address = 10.10.0.51 		#Management IP of Controller Node
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```
Start the database service:
```
systemctl enable mariadb.service
systemctl start mariadb.service
```
Secure the database service:
```
mysql_secure_installation
```

#####Message queue
Install the package:
```
yum install rabbitmq-server
```
Start the message queue service:
```
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```
Add the openstack user:
```
rabbitmqctl add_user openstack openstack
```
Permit configuration, write, and read access for the openstack user:
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#####Memcached
Install the packages:
```
yum install memcached python-memcached
```
Edit the /etc/sysconfig/memcached:
```
10.10.0.51
```
Start the Memcached service:
```
systemctl enable memcached.service
systemctl start memcached.service
```

####Keystone
Create Database:
```
mysql -u root -p

CREATE DATABASE keystone;

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'openstack';

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'openstack';
```
Install the packages:
```
yum install openstack-keystone httpd mod_wsgi
```
Edit the /etc/keystone/keystone.conf:
```
[database]
...
connection = mysql+pymysql://keystone:openstack@controller/keystone

[token]
...
provider = fernet
```
Populate the database:
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
Initialize Fernet key repositories:
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
Bootstrap:
```
keystone-manage bootstrap --bootstrap-password openstack \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:35357/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
Edit the /etc/httpd/conf/httpd.conf:
```
ServerName controller
```
Create a link to the /usr/share/keystone/wsgi-keystone.conf file:
```
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```
Start the Apache HTTP service:
```
systemctl enable httpd.service
systemctl start httpd.service
```
Configure the administrative account:
```
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```
Create a domain, projects, users, and roles:
```
openstack project create --domain default \
  --description "Service Project" service

openstack project create --domain default \
  --description "Demo Project" demo

openstack user create --domain default \
  --password-prompt demo

openstack role create user

openstack role add --project demo --user demo user
```
Disable the temporary authentication token mechanism:
```
Edit /etc/keystone/keystone-paste.ini: Remove admin_token_auth from [pipeline:public_api], [pipeline:admin_api], [pipeline:api_v3]

unset OS_AUTH_URL OS_PASSWORD
```
Create admin client, create and edit admin-openrc:
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
Verify:
```
source admin-openrc
openstack token issue
```

####Glance
Create Database:
```
mysql -u root -p

CREATE DATABASE glance;

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'openstack';

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'openstack';
```
Create service credentials:
```
openstack user create --domain default --password-prompt glance

openstack role add --project service --user glance admin

openstack service create --name glance \
  --description "OpenStack Image" image

openstack endpoint create --region RegionOne \
  image public http://controller:9292

openstack endpoint create --region RegionOne \
  image internal http://controller:9292

openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```
Install the packages:
```
yum install -y openstack-glance
```
Edit the /etc/glance/glance-api.conf:
```
[database]
...
connection = mysql+pymysql://glance:openstack@controller/glance

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = openstack

[paste_deploy]
...
flavor = keystone

[glance_store]
...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
Edit the /etc/glance/glance-registry.conf:
```
[database]
...
connection = mysql+pymysql://glance:openstack@controller/glance

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = openstack

[paste_deploy]
...
flavor = keystone
```
Populate the Image service database:
```
su -s /bin/sh -c "glance-manage db_sync" glance
```
Start Glance services:
```
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
```
Verify:
```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

openstack image list
```

####Nova
Create Database:
```
mysql -u root -p

CREATE DATABASE nova_api;

CREATE DATABASE nova;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'openstack';

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'openstack';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'openstack';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'openstack';
```
Create the service credentials:
```
openstack user create --domain default \
  --password-prompt nova

openstack role add --project service --user nova admin

openstack service create --name nova \
  --description "OpenStack Compute" compute

openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1/%\(tenant_id\)s
```
Install the packages:
```
yum install -y openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler
```
Edit /etc/nova/nova.conf:
```
[DEFAULT]
...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone
my_ip = 10.10.0.51
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
...
connection = mysql+pymysql://nova:openstack@controller/nova_api

[database]
...
connection = mysql+pymysql://nova:openstack@controller/nova

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = openstack

[vnc]
...
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
...
api_servers = http://controller:9292

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp
```
Populate the databases:
```
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage db sync" nova
```
Start Nova services:
```
systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```
Verify:
```
openstack compute service list
```

####Neutron (L2)
Create Database:
```
mysql -u root -p

CREATE DATABASE neutron;

GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'openstack';

GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'openstack';
```
Create Neutron credentials:
```
openstack user create --domain default --password-prompt neutron

openstack role add --project service --user neutron admin

openstack service create --name neutron \
  --description "OpenStack Networking" network

openstack endpoint create --region RegionOne \
  network public http://controller:9696

openstack endpoint create --region RegionOne \
  network internal http://controller:9696

openstack endpoint create --region RegionOne \
  network admin http://controller:9696
```
Install packages:
```
yum install -y openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```
Edit /etc/neutron/neutron.conf:
```
[DEFAULT]
...
core_plugin = ml2
service_plugins =
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True


[database]
...
connection = mysql+pymysql://neutron:openstack@controller/neutron

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = openstack

[nova]
...
auth_url = http://controller:35357
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password =  openstack

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```
Edit /etc/neutron/plugins/ml2/ml2_conf.ini:
```
[ml2]
...
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
...
flat_networks = provider

[securitygroup]
...
enable_ipset = True
```
Edit /etc/neutron/plugins/ml2/linuxbridge_agent.ini:
```
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = False

[securitygroup]
...
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
Edit /etc/neutron/dhcp_agent.ini:
```
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```
Edit /etc/neutron/metadata_agent.ini:
```
nova_metadata_ip = controller
metadata_proxy_shared_secret =  openstack
```
Edit /etc/nova/nova.conf:
```
[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password =  openstack
service_metadata_proxy = True
metadata_proxy_shared_secret =  openstack
```

Create symbolic link:
```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```
Populate the database:
```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```
Restart Nova:
```
systemctl restart openstack-nova-api.service
```
Start Neutron:
```
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service

systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
``` 
Verify:
```
neutron ext-list

openstack network agent list
```

####Horizon
Install packages:
```
yum install -y openstack-dashboard
```
Edit /etc/openstack-dashboard/local_settings:
```
OPENSTACK_HOST = "controller"

ALLOWED_HOSTS = ['*', ]

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "Asia/Ho_Chi_Minh"
```
Restart Apache:
```
systemctl restart httpd.service memcached.service
```

###Compute Node
####Environment Setup
#####NTP
Install the packages:
```
yum install -y chrony
```
Edit the /etc/chrony.conf:
```
server 10.10.0.10 iburst

```
Start the NTP service:
```
systemctl enable chronyd.service
systemctl start chronyd.service
```
Verify:
```
chronyc sources
```
####Nova
Install the packages:
```
yum install -y openstack-nova-compute
```
Edit /etc/nova/nova.conf:
```
[DEFAULT]
...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone
my_ip = 10.10.0.x
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
...
connection = mysql+pymysql://nova:openstack@controller/nova_api

[database]
...
connection = mysql+pymysql://nova:openstack@controller/nova

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = openstack

[vnc]
...
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.10.51:6080/vnc_auto.html

[glance]
...
api_servers = http://controller:9292

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp
```
Check vitualization support:
```
egrep -c '(vmx|svm)' /proc/cpuinfo
```
If value = 0:
```
[libvirt]
...
virt_type = qemu
```
Start Nova services:
```
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

####Neutron
Install packages:
```
yum install openstack-neutron-linuxbridge ebtables ipset
```
Edit /etc/neutron/neutron.conf:
```
[DEFAULT]
...
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password =  openstack

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```
Edit /etc/neutron/plugins/ml2/linuxbridge_agent.ini:
```
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = False

[securitygroup]
...
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

Edit /etc/nova/nova.conf:
```
[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password =  openstack
```
Restart Nova:
```
systemctl restart openstack-nova-compute.service
```
Start Neutron:
```
systemctl enable neutron-linuxbridge-agent.service
systemctl start neutron-linuxbridge-agent.service
```

###Install Cinder + Integrate Ceph
####Install Cinder (API+Volume) on Controller Node
Create Database:
```
mysql -u root -p

CREATE DATABASE cinder;

GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'openstack';

GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'openstack';
```
Create Cinder credentials:
```
openstack user create --domain default --password-prompt cinder

openstack role add --project service --user cinder admin

openstack service create --name cinder \
  --description "OpenStack Block Storage" volume

openstack service create --name cinderv2 \
  --description "OpenStack Block Storage" volumev2

openstack endpoint create --region RegionOne \
  volume public http://controller:8776/v1/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  volume internal http://controller:8776/v1/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  volume admin http://controller:8776/v1/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  volumev2 public http://controller:8776/v2/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  volumev2 internal http://controller:8776/v2/%\(tenant_id\)s

openstack endpoint create --region RegionOne \
  volumev2 admin http://controller:8776/v2/%\(tenant_id\)s
```
Install packages:
```
yum install openstack-cinder targetcli python-keystone
```
Edit /etc/cinder/cinder.conf:
```
[DEFAULT]
...
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone
my_ip = 10.10.0.51
glance_api_servers = http://controller:9292
[database]
...
connection = mysql+pymysql://cinder:openstack@controller/cinder

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password =  openstack

[oslo_concurrency]
...
lock_path = /var/lib/cinder/tmp
```
Populate database:
```
su -s /bin/sh -c "cinder-manage db sync" cinder
```
Edit /etc/nova/nova.conf:
```
[cinder]
os_region_name = RegionOne
```
Restart Nova:
```
systemctl restart openstack-nova-api.service
```
Start Cinder:
```
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service /
openstack-cinder-volume.service target.service

systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service /
openstack-cinder-volume.service target.service
```

####Install Ceph on OpenStack Node
Install Ceph over Ceph Admin Node:
```
ceph-deploy install {openstack-node}
```
Copy config file:
```
ssh {openstack-node} sudo tee /etc/ceph/ceph.conf </etc/ceph/ceph.conf
```
####Create Ceph Cinder user authentication
On Ceph Admin Node:
```
ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool={pool-name}'

ceph auth get-or-create client.cinder | ssh cinder-volume-server} sudo tee /etc/ceph/ceph.client.cinder.keyring

ssh {cinder-volume-server} sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring

ceph auth get-or-create client.cinder | ssh {your-nova-compute-server} sudo tee /etc/ceph/ceph.client.cinder.keyring
```
Edit /etc/ceph/ceph.conf:
```
[client.cinder]
keyring = /etc/ceph/ceph.client.cinder.keyring
```
Verify on OPS Node:
```
ceph -n client.cinder --keyring=/etc/ceph/ceph.client.cinder.keyring health
```

####Define Virsh Secret for Nova Compute
On Ceph Admin Node:
```
ceph auth get-key client.cinder | ssh {your-compute-node} tee client.cinder.key
```
On Nova Compute Node:
```
uuidgen
{uuid-value}

cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>{uuid-value}</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF

sudo virsh secret-define --file secret.xml

sudo virsh secret-set-value --secret {value} --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml
```

####Config Cinder + Nova
Edit /etc/cinder/cinder.conf:
```
[DEFAULT]
...
enabled_backends = rbd

[rbd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = {uuid-value}
report_discard_supported = true
```
