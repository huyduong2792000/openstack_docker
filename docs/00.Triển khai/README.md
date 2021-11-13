# Hướng dẫn triển khai openstack 2 region sử dụng docker và kolla

## Mục lục

[1. Yêu cầu hệ thống](#system-require)

[2. Setup môi trường cài đặt](#set-up)

- [2.1. Cập nhật và nâng cấp các package ubuntu](#set-up_update-ubuntu)
- [2.2. Cài đặt python3 và các package](#set-up_python-venv)
- [2.3. Cài đặt ansible để deploy](#set-up_install-ansible)
- [2.4. Cài đặt và cấu hình kolla-ansible cho ubuntu](#set-up_install-kolla-ansible)

[3. Deploy region 1](#deploy-region1)

[4. Deploy region 2](#deploy-region2)
- [4.1. Trước khi deploy region 2 cần đảm bảo một số cấu hình và khai báo thêm regionTwo ở node 1](#deploy-region2_makesure-region1)
- [4.2. Tiến hành deploy region 2](#deploy-region2_deploy-region2)

[5. Kiểm tra kết quả](#check-result)

--------

<a name="system-require"></a>
## 1. Yêu cầu hệ thống 

**Host:**: 2 máy ảo

**Network:** Ít nhất 2 network interface

**Ram:** Tối thiểu 4GB ram/host

**Disk:** Tối thiểu 40G/host

![Triển khai](/images/kolla_openstack_region1_info.png)

![Triển khai](/images/kolla_openstack_region2info.png)


**IP Planning**

region1: enp0s8: 192.168.56.10, enp0s9: không set

region2: enp0s8: 192.168.56.20, enp0s9: không set

<a name="set-up"></a>
## 2. Setup môi trường cài đặt( thực hiện trên cả 2 region)

<a name="set-up_update-ubuntu"></a>
### 2.1  Cập nhật và nâng cấp các package ubuntu

``` sh
sudo apt update
sudo apt upgrade
```
<a name="set-up_python-venv"> </a>
### 2.2 Cài đặt python3 và các package để có thể tạo môi trường deploy
**Cài đặt python3-venv**

```sh
sudo apt install python3-dev python3-venv libffi-dev gcc libssl-dev git
```

**Tạo môi trường deploy ảo cho kolla-ansible**

``` sh
python3 -m venv $HOME/kolla-openstack
```
Tạo môi trường nhằm mục đích quản lý các package dễ dàng và tách biệt với các package ở môi trường ngoài

**Active môi trường vừa tạo**

``` sh
source $HOME/kolla-openstack/bin/activate
```

Sau khi active môi trường thì các gói cài đặt sẽ được mount vào folder lib của môi trường
Terminal sau khi active môi trường sẽ có dạng 

```sh
(kolla-openstack)$HOME
```

**Cập nhật công cụ quản lý package của python**

```sh
pip install -U pip
```
<a name="set-up_install-ansible"> </a>
### 2.3 Cài đặt ansible để deploy

**Cài đặt phiên bản <2.10 để phù hợp với ussuri của openstack**

```sh
pip install 'ansible<2.10'

```
**Tạo file cấu hình ansible**

```sh
vim $HOME/ansible.cfg
```

Nội dung file 
```
[defaults]
host_key_checking=False
pipelining=True
forks=100
```
<a name="set-up_install-kolla-ansible"></a>
### 2.4 Cài đặt và cấu hình kolla-ansible cho ubuntu

Kolla-ansible là một công cụ deploy của openstack được base trên ansible nên phải cài cả kolla-ansible và ansible

**Cài đặt kolla-ansible phiên bản 10.0.0(Phiên bản đầu tiên cho việc deploy openstack ussuri)

``` sh
pip install kolla-ansible==10.0.0
```

**Tạo direactory /etc/kolla để chứa các file cấu hình kolla-ansible**

```sh
sudo mkdir /etc/kolla
```

**Cấp quyền folder chứa file cấu hình**

```sh
sudo chown $USER:$USER /etc/kolla 
```

**Sao chép file cài đặt(globals.yml) và file chứa mật khẩu(passwords.yml) vào folder cấu hình vừa tạo ở trên**

```sh
cp $HOME/kolla-openstack/share/kollaansible/etc_examples/kolla/*\ /etc/kolla/
```

**Sao chép file inventory all in one(file chứa danh sách các server cần deploy) ra thư mục hiện tại**

```sh
cp $HOME/kolla-openstack/share/kolla-ansible/ansible/inventory/all-in-one .
```

**Thiết lập một số thông tin cài đặt đặc biệt trong file /etc/kolla/globals.yml**

Mở file này ra và đặt lại giá trị một số trường( nên sử dụng vim để mở file và tìm kiếm) 

```
config_strategy: "COPY_ALWAYS"
kolla_base_distro: "ubuntu"
kolla_install_type: "binary"
openstack_release: "ussuri"
kolla_internal_vip_address: "192.168.56.11"
kolla_internal_fqdn: "{{ kolla-openstack.kifarunix-demo.com }}"
kolla_external_vip_address: "{{ kolla_internal_vip_address }}"
kolla_external_fqdn: "{{ kolla_internal_fqdn }}"
network_interface: "enp0s8"
neutron_external_interface: "enp0s9"
neutron_plugin_agent: "openvswitch"
enable_haproxy: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
keystone_token_provider: 'fernet'
cinder_volume_group: "controller-vg"
nova_compute_virt_type: "qemu"
```

**Cập nhật lại các thay đổi vào biến môi trường**

```sh
source $HOME/kolla-openstack/bin/activate
```

**Tạo các mật khẩu quản trị, các mật khẩu này sẽ được lưu vào file  /etc/kolla/passwords.yml**

```sh
kolla-genpwd
```
<a name="deploy-region1"></a>
## 3. Deploy region 1( Chỉ thực hiện trên node 1)

**Khởi động cấu hình trước khi deploy**

```sh
kolla-ansible -i all-in-one bootstrap-servers
```
![Triển khai](/images/kolla_openstack_bootstrap_servers.png)

**Kiểm tra các cấu hình hệ thống xem đủ các điều kiện để deploy chưa**

```sh
kolla-ansible -i all-in-one prechecks
```
![Triển khai](/images/kolla_openstack_prechecks.png)

**Nếu như tất cả các điều kiện đều được thông qua thì bắt đầu deploy**

```sh
kolla-ansible -i all-in-one deploy
```
![Triển khai](/images/kolla_openstack_deploy.png)

**Sau khi deploy có thể kiểm tra trạng thái hoạt động của các service bằng lệnh**

```sh
docker ps
```
hoặc 
```sh
openstack service list
```

**Cài đặt command line administration tools của openstack**

```sh
pip install python-openstackclient python-neutronclient python-glanceclient
```

**Tạo OpenStack admin user credentials**

```sh
kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
```

**Tạo OpenStack network, images, nova keys demo**

```sh
kolla-openstack/share/kolla-ansible/init-runonce
```
<a name="deploy-region2"></a>
## 4. Deploy region 2( Chỉ thực hiện trên node 2)

<a name="deploy-region2_makesure-region1"> </a>
### 4.1 Trước khi deploy region 2 cần đảm bảo một số cấu hình và khai báo thêm regionTwo ở node 1

**Đảm bảo node1 đã cấu hình enable keystone và horizon trong /etc/kolla/globals.yml**

```
enable_keystone: "yes"
enable_horizon: "yes"
```

**Khai báo thêm region thứ hai có tên RegionTwo( Cũng trong file /etc/kolla/globals.yml)**

```
openstack_region_name: "RegionOne"
multiple_regions_names:
    - "{{ openstack_region_name }}"
    - "RegionTwo"
```

**reconfig lại node 1 sau khi config thêm RegionTwo**

```sh
kolla-ansible -i all-in-one  reconfigure
```

<a name="deploy-region2_deploy-region2"> </a>
### 4.2 Tiến hành deploy region 2( regionTwo)

**Sửa file /etc/kolla/globals.yml để region có thể liên kết được với region 1**

```
kolla_internal_fqdn_r1: 192.168.56.11

keystone_admin_url: "{{ admin_protocol }}://{{ kolla_internal_fqdn_r1 }}:{{ keystone_admin_port }}"
keystone_internal_url: "{{ internal_protocol }}://{{ kolla_internal_fqdn_r1 }}:{{ keystone_public_port }}"

openstack_auth:
    auth_url: "{{ admin_protocol }}://{{ kolla_internal_fqdn_r1 }}:{{ keystone_admin_port }}"
    username: "admin"
    password: "{{ keystone_admin_password }}"
    project_name: "admin"
    domain_name: "default"
```

**Khai báo đường dẫn ghi đè config**

Bước này nhằm tách biệt config của region với global config để dễ dàng maintain sau này

```
node_custom_config: "/etc/kolla/config"
```

**Tạo file global.conf trong folder /etc/kolla/config với nội dung**

```
[keystone_authtoken]
www_authenticate_uri = {{ keystone_internal_url }}
auth_url = {{ keystone_admin_url }}
```

**Tạo file nova.conf trong folder /etc/kolla/config/nova với nội dung**

```
[placement]
auth_url = {{ keystone_admin_url }}
```

**Tạo file heat.conf trong folder /etc/kolla/config/heat với nội dung**

```
[trustee]
www_authenticate_uri = {{ keystone_internal_url }}
auth_url = {{ keystone_internal_url }}

[ec2authtoken]
www_authenticate_uri = {{ keystone_internal_url }}

[clients_keystone]
www_authenticate_uri = {{ keystone_internal_url }}
```

**Tạo file ceilometer.conf trong folder /etc/kolla/config/ceilometer với nội dung**

```
[service_credentials]
auth_url = {{ keystone_internal_url }}
```

**Thay đổi tên region trong /etc/kolla/globals.yml của node2**

```
openstack_region_name: "RegionTwo"
```

**Khai báo cho node 2 không chạy service keystone và horizon**

```sh
enable_keystone: "no"
enable_horizon: "no"
```

<a name="check-result"> </a>
## 5. Kiểm tra kết quả

Vào browser nhập đường dẫn http://192.168.56.11/ sẽ thấy màn hình login

Lấy username và password trong file /etc/kolla/passwords.yml

Đăng nhập vào sẽ thấy có sẵn một image cirros demo tạo instance với image đó thành công là được

Một số hình ảnh kết quả triển khai:

![Triển khai](/images/kolla_openstack_dashboard.png)

![Triển khai](/images/kolla_openstack_system_info.png)

![Triển khai](/images/kolla_openstack_network_topology.png)
