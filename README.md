OPENSTACK
=================================================================
#Cài đặt OpenStack ICEHOUSE trên 3 Node

Nguồn: http://docs.openstack.org/icehouse/install-guide/install/apt/content/index.html

##Cài đặt OpenStack ICEHOUSE trên 3 node 

* KeyStone
* Glance
* Nova
* Neutron
* Horizon

### 1. Chuẩn bị


|       HOST     |          NICS          |
| ---------------|------------------------| 
|    Controller  | eth0: 10.10.10.61/24   |
|                | eth1: 172.16.69.61/24  | 


|       HOST     |          NICS          |
| ---------------|------------------------| 
|    Compute     | eth0: 10.10.10.62/24   |
|                | eth1: 172.16.69.62/24  |
|                | eth2: 10.10.20.62/24   |


|       HOST     |          NICS          |
| ---------------|------------------------| 
|    Network     | eth0: 10.10.10.63/24   |
|                | eth1: 172.16.69.63/24  |
|                | eth2: 10.10.20.63/24   |

!["image"](http://i.imgur.com/c02p9u8.png "topo")

* Cấu hình máy

Controller: 

* HDD: 20 GB
* CPU: 02 (Có tích vào các chế độ ảo hóa)
* RAM: 2GB 
* NIC: 02 NICs (eth0 - chế độ vmnet3 ), (eth1 - chế độ brige)
* Hệ điều hành Ubuntu Server 12.04 (LTS)

Compute:

* HDD: 20 GB
* CPU: 02 (Có tích vào các chế độ ảo hóa)
* RAM: 2GB 
* NIC: 02 NICs (eth0 - chế độ vmnet3 ), (eth1 - chế độ brige), (eth2 - chế độ vmnet4)
* Hệ điều hành Ubuntu Server 12.04 (LTS)

Network:

* HDD: 20 GB 
* CPU: 02 (Có tích vào các chế độ ảo hóa)
* RAM: 512 MB 
* NIC: 02 NICs (eth0 - chế độ vmnet3 ), (eth1 - chế độ brige), (eth2 - chế độ vmnet4)
* Hệ điều hành Ubuntu Server 12.04 (LTS)

### 2.  Cấu hình cơ bản

* Đặt tên hostname: 

controller
compute
network

* Cấu hình file etc/hosts tại 3 node

```
10.10.10.61       controller
10.10.10.62       compute
10.10.10.63       network
```

* Network Time Protocol (NTP)

Cài đặt ntp trên 3 node
```
# apt-get install ntp
```
Để cho phép Network và Compute node đồng bộ thời gian với Controller node, sửa file /etc/ntp.conf

Comment các dòng (thêm dấu #)
```
server 0.ubuntu.pool.ntp.org
server 1.ubuntu.pool.ntp.org
server 2.ubuntu.pool.ntp.org
server 3.ubuntu.pool.ntp.org
```
Sửa dòng server ntp.ubuntu.com thành: server controller (hoặc ip của controller)

Lưu thay đổi và khởi động lại dịch vụ NTP
```
#service ntp restart
```
* Passsword

Tạo password ngẫu nhiên 
```
$ openssl rand -hex 10
```
Các loại password

!["image"](http://i.imgur.com/ezJtviV.png "Password")

*   Database

_Cài đặt trên controller node_

```
#apt-get install python-mysqldb mysql-server
```
Sửa file /etc/mysql/my.cnf:
```
bind-address = 10.10.10.61
```
Lưu thay đổi và khởi động lại dịch vụ MySQL

```
#service mysql restart
```

Xóa anonymous user, test database, đặt mật khẩu root cho MySQL

```
#mysql_install_db
#mysql_secure_installation
```

_Cài đặt trên 2 node còn lại_


```
#apt-get install python-mysqldb
```

*   OpenStack packages

Cài đặt trên 3 node

Cài đặt  Ubuntu Cloud Archive cho Icehouse
```
#apt-get update
#apt-get install python-software-properties
#apt-get update
#add-apt-repository cloud-archive:icehouse
```
Update và Upgrade hệ điều hành
```
# apt-get update
# apt-get dist-upgrade
```
*   Messaging server ( RabbitMQ)

Cài đặt trên controller node

```
#apt-get install rabbitmq-server
```

Đổi mật khẩu guest user
```
#rabbitmqctl change_password guest RABBIT_PASS
```

###3. Identity Service-Keystone
####3.1 Cài đặt keystone

Cài đặt dịch vụ keystone ở Controller:
```
#apt-get install keystone
```

Sửa dòng cấu hình kết nối database trong file /etc/keystone/keystone.conf

```
#nano /etc/keystone/keystone.conf
```
```
...
[database]
# The SQLAlchemy connection string used to connect to the database
connection = mysql://keystone:KEYSTONE_DBPASS@controller/keystone
...
```

Xóa database mặc định của Ubuntu cho keystone
```
#rm /var/lib/keystone/keystone.db
```
Tạo database trong MySQL cho keystone
```
#mysql -u root -p
```
<Nhập mật khẩu root của mysql>

```
mysql> CREATE DATABASE keystone;
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
mysql> FLUSH PRIVILEGES;
mysql> exit
```

Tạo các bảng trong database của keystone
```
#su -s /bin/sh -c “keystone-manage db_sync” keystone
```

Cấu hình token để bảo mật kết nối giữa dịch vụ Identity và các dịch vụ khác.

Sửa file /etc/keystone/keystone.conf và cập nhật dòng:
```
#nano /etc/keystone/keystone.conf
```
```
[DEFAULT]
# A "shared secret" between keystone and other openstack services
admin_token = ADMIN_TOKEN
...
```
Cấu hình thư mục chứa log cho keystone. Sửa file /etc/keystdongfkeystone.conf và cập nhật dòng:
```
[DEFAULT]
...
log_dir = /var/log/keystone
```
Khởi động lại dịch vụ keystone
```
#service keystone restart
```
Chạy lệnh sau để thanh lọc token hết hạn mỗi giờ và ghi đầu ra vào /var/log/keystone/keystone-tokenflush.log

```
# (crontab -l -u keystone 2>&1 | grep -q token_flush) || \
echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' >> /var/spool/cron/crontabs/keystone
```
####3.2 Cài đặt các user, tenant và role

*   Thiết đặt các biến môi trường để chứng thực việc sử dụng lệnh keyston(keystone’s CLI)

```
#export OS_SERVICE_TOKEN=ADMIN_TOKEN
#export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
```

*   Tạo user admin
```
#keystone user-create --name=admin --pass=ADMIN_PASS --email=ADMIN_EMAIL
```
*   Tạo admin role
```
#keystone role-create --name=admin
```
*   Tạo admin tenant
```
keystone tenant-create --name=admin --description="Admin Tenant"
```
*   Liên kết admin user, admin role và admin tenant
```
#keystone user-role-add --name=admin --tenant=admin --role=admin
```
*   Liên kết admin user, _member_ role và admin tenant
```
#keystone user-role-add --user=admin --role=_member_ --tenant=admin
```
*   Tạo demo user
```
#keystone user-create --name=demo --pass=DEMO_PASS --email=demo@example.com
```

*   Tạo demo tenant
```
#keystone tennant-create --name=demo --description="Demo Tenant"
```
*   Liên kết demo user, _member_ role và demo tenant
```
#keystone user-role-add --user=demo --role=_member_ --tenant=demo
```

*   Các dịch vụ OpenStack yêu cầu một username, tenant và role để truy cập đến các dịch vụ OpenStack. Trong cài đặt cơ bản, các dịch vụ OpenStack chia sẽ một tenant duy nhất tên là service. Tạo service tenant

```
#keystone tenant-create --name=service --description="Service Tenant"
```

####3.3 Xác định dịch vụ và API endpoint

*   Để dịch vụ Identity có thể kiểm tra các dịch vụ OpenStack đã được cài đặt và địa chỉ truy cập của chúng trên mạng, bạn phải đăng ký cho mỗi dịch vụ. Để đăng ký một dịch vụ, chạy các lệnh sau:

```
    keystone service-create: định nghĩa dịch vụ

    keystone endpoint-create: liên kết API endpoint với dịch vụ
```
*   API endpoint: một tiến trình chạy ẩn (daemon), một công việc (worker), hoặc một dịch vụ mà một client  giao tiếp để truy cập vào một API. API endpoint có thể cung cấp bất kỳ số liệu về một dịch vụ, như chứng thực, dữ liệu sử dụng, số liệu hiệu suất…. Một API endpoint trong OpenStack bao gồm các URLs cho public API, internal API và admin API.

Đăng ký dịch vụ Identity
```
#keystone service-create --name=keystone --type=identity \
  --description="OpenStack Identity"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| id          | 15c11a23667e427e91bc31335b45f4bd |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
```
ID của dịch vụ sẻ được sinh ngẫu nhiên và duy nhất.

Tạo API endpoint cho dịch vụ Identity bằng cách sử dụng Identity ID. Lưu ý răng dịch vụ Identity sử dụng một cổng khác cho admin API.
```
#keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
  --publicurl=http://controller:5000/v2.0 \
  --internalurl=http://controller:5000/v2.0 \
  --adminurl=http://controller:35357/v2.0
+-------------+-----------------------------------+
|   Property  |             Value                 |
+-------------+-----------------------------------+
| adminurl    | http://controller:35357/v2.0      |
| id          | 11f9c625a3b94a3f8e66bf4e5de2679f  |
| internalurl | http://controller:5000/v2.0       |
| publicurl   | http://controller:5000/v2.0       |
| region      | regionOne                         |
| service_id  | 15c11a23667e427e91bc31335b45f4bd  |
+-------------+-----------------------------------+
```

```
$ unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
```
```
$ keystone --os-username=admin --os-password=ADMIN_PASS \
  --os-auth-url=http://controller:35357/v2.0 token-get
```
```
$ keystone --os-username=admin --os-password=ADMIN_PASS \
  --os-tenant-name=admin --os-auth-url=http://controller:35357/v2.0 \
  token-get
```
Tạo file admin-openrc.sh chứa các biến môi trường nhằm chứng thực cho việc sử dụng CLI sau này

vi admin-openrc.sh

```
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://controller:35357/v2.0
```

Nhập biến môi trường bằng lệnh:

```
$ source admin-openrc.sh
```
```
$ keystone user-list
+----------------------------------+-------+---------+-------------------+
|                id                |  name | enabled |    email          |
+----------------------------------+-------+---------+-------------------+
| afea5bde3be9413dbd60e479fddf9228 | admin |   True  | admin@example.com |
| 32aca1f9a47540c29d6988091f76c934 |  demo |   True  | demo@example.com  |
+----------------------------------+-------+---------+-------------------+
```
```
$ keystone user-role-list --user admin --tenant admin
+----------------------------------+----------+----------------------------------+----------------------------------+
|                id                |   name   |             user_id              |            tenant_id             |
+----------------------------------+----------+----------------------------------+----------------------------------+
| 9fe2ff9ee4384b1894a90878d3e92bab | _member_ | afea5bde3be9413dbd60e479fddf9228 | e519b772cb43474582fa303da62559e5 |
| 5d3b60b66f1f438b80eaae41a77b5951 |  admin   | afea5bde3be9413dbd60e479fddf9228 | e519b772cb43474582fa303da62559e5 |
+----------------------------------+----------+----------------------------------+----------------------------------+
```
###4. Image Service-Glance

####4.1 Cài đặt dịch vụ glance

Cài đặt dịch vụ Image trên Controller node
```
#apt-get install glance python-glanceclient
```

Sửa cấu hình kết nối database ở các file /etc/glance/glance-api.conf và /etc/glance/glance-registry.conf
```
...
[database]
connection = mysql://glance:GLANCE_DBPASS@controller/glance
```
Sửa file /etc/glance/glance-api.conf

```
[DEFAULT]
...
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = RABBIT_PASS
```

Mặc định, Ubuntu tạo một SQLite database cho glance. Xóa glance.sqlite file
```
#rm /var/lib/glance/glance.sqlite
```
Tạo database trong MySQL cho glance
```
#mysql -u root -p
```
<Nhập mật khẩu root của MySQL>
```
mysql> CREATE DATABASE glance;
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
IDENTIFIED BY 'GLANCE_DBPASS';
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
IDENTIFIED BY 'GLANCE_DBPASS';
mysql> FLUSH RTIVILEGES
```

Tạo các database tables cho dịch vụ Image
```
#su -s /bin/sh -c "glance-manage db_sync" glance
```
Tạo một glance user trong keystone để dịch vụ Image có thể chứng thực với dịch vụ Identity. Sử dụng service tenant và admin role
```
#keystone user-create --name=glance --pass=GLANCE_PASS \
   --email=glance@example.com
#keystone user-role-add --user=glance --tenant=service --role=admin
```
Cấu hình dịch vụ Image sử dụng dịch vụ Identity để chứng thực. Sửa các file /etc/glance/glance-api.conf và /etc/glance/glance-registry.conf

- Sửa các dòng dưới phần [keystone_authtoken]:
```
[keystone_authtoken]
auth_uri = http://controller:5000
auth_host = controller
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = GLANCE_PASS
```

- Sửa flavor dưới phần [paste_deploy]:
```
[paste_deploy]
...
flavor = keystone
```
Đăng ký và tạo endpoint cho dịch vụ Image
```
#keystone service-create --name=glance --type=image \
  --description="OpenStack Image Service"
#keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ image / {print $2}') \
  --publicurl=http://controller:9292 \
  --internalurl=http://controller:9292 \
  --adminurl=http://controller:9292
```

Khởi động lại dịch vụ glance để cập nhật cấu hình mới
```
#service glance-registry restart
#service glance-api restart
```
####4.2 Upload file ảnh

Tải CirrOS image
```
#mkdir images
#cd images
#wget http://cdn.download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img
```
Upload lên dịch vụ Image
```
#glance image-create --name=IMAGELABEL --disk-format=FILEFORMAT \
  --container-format=CONTAINERFORMAT --is-public=ACCESSVALUE < IMAGEFILE
```
Ví dụ:
```
#source admin-openrc.sh
#glance image-create --name "cirros-0.3.2-x86_64" --disk-format qcow2 \
 --container-format bare --is-public True --progress < cirros-0.3.2-x86_64-disk.img
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 64d7c1cd2b6f60c92c14662941cb7913     |
| container_format | bare                                 |
| created_at       | 2014-04-08T18:59:18                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | acafc7c0-40aa-4026-9673-b879898e1fc2 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.2-x86_64                  |
| owner            | efa984b0a914450e9a47788ad330699d     |
| protected        | False                                |
| size             | 13167616                             |
| status           | active                               |
| updated_at       | 2014-01-08T18:59:18                  |
+------------------+--------------------------------------+
```
Kiểm tra file ảnh CirrOS vừa upload
```
#glance image-list
+--------------------------------------+---------------------+-------------+------------------+----------+--------+
| ID                                   | Name                | Disk Format | Container Format | Size     | Status |
+--------------------------------------+---------------------+-------------+------------------+----------+--------+
| acafc7c0-40aa-4026-9673-b879898e1fc2 | cirros-0.3.2-x86_64 | qcow2       | bare             | 13167616 | active |
+--------------------------------------+---------------------+-------------+------------------+----------+--------+
```
###5. Compute Service-Nova
####5.1 Cài đặt dịch vụ Compute ở Controller node

Cài đặt các gói của dịch vụ Compute
```
#apt-get install nova-api nova-cert nova-conductor nova-consoleauth \
  nova-novncproxy nova-scheduler python-novaclient
```
Cấu hình kết nối database trong file /etc/nova/nova.conf
Thêm phần [database] vào cuối file:
```
[database]
connection = mysql://nova:NOVA_DBPASS@controller/nova
```
Cấu hình RabbitMQ trong file /etc/nova/nova.conf
```
[DEFAULT]
...
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = RABBIT_PASS
```
Cấu hình vnc trong file /etc/nova/nova.conf
```
[DEFAULT]
....
my_ip = 10.10.10.61
vncserver_listen = 10.10.10.61
vncserver_proxyclient_address = 10.10.10.61
```
Xóa SQLite database mặc định của Ubuntu tạo cho nova
```
#rm /var/lib/nova/nova.sqlite
```
Tạo database cho nova trong MySQL
```
#mysql -u root -p
```
<Nhập mật khẩu root của MySQl>
```
mysql> CREATE DATABASE nova;
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY 'NOVA_DBPASS';
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
IDENTIFIED BY 'NOVA_DBPASS';
mysql> FLUSH PRIVILEGES;
```
Tạo database tables cho nova
```
#su -s /bin/sh -c "nova-manage db sync" nova
```
Tạo một nova user trong keystone để dịch vụ Compute có thể chứng thực với dịch vụ Identity. Sử dụng service tenant và admin role
```
#keystone user-create --name=nova --pass=NOVA_PASS --email=nova@example.com
#keystone user-role-add --user=nova --tenant=service --role=admin
```
Cấu hình Compute sử dụng các chứng chỉ của dịch vụ Identity. Cấu hình file /etc/nova/nova.conf
```
[DEFAULT]
....
auth_strategy = keystone
```
Thêm phần [keystone_authtoken] vào cuối file
```
[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_host = controller
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = NOVA_PASS
```

Đăng ký và tạo endpoint cho dịch vụ Compute
```
#keystone service-create --name=nova --type=compute \
  --description="OpenStack Compute"
#keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ compute / {print $2}') \
  --publicurl=http://controller:8774/v2/%\(tenant_id\)s \
  --internalurl=http://controller:8774/v2/%\(tenant_id\)s \
  --adminurl=http://controller:8774/v2/%\(tenant_id\)s
```

Khởi động lại dịch vụ Compute
```
#service nova-api restart
#service nova-cert restart
#service nova-consoleauth restart
#service nova-scheduler restart
#service nova-conductor restart
#service nova-novncproxy restart
```
Để xác minh cấu hình xem danh sách các image
```
$ nova image-list
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| acafc7c0-40aa-4026-9673-b879898e1fc2 | cirros-0.3.2-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```

####5.1 Cài đặt dịch vụ Compute ở Compute node

```
#apt-get install nova-compute-kvm python-guestfs

# dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-$(uname -
r)

```
Tạo file : /etc/kernel/postinst.d/statoverride
```
#!/bin/sh
version="$1"
# passing the kernel version is required
[ -z "${version}" ] && exit 0
dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-${version}

```
```
# chmod +x /etc/kernel/postinst.d/statoverride
```


