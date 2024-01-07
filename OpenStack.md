- OpenStack gồm 2 dịch vụ:
  - OpenStack controller:
    - ens32: 10.10.10.10
    - ens33: 10.10.20.10
    - ens34: 192.168.17.129
  - OpenStack compute:
    - ens32: 10.10.10.11
    - ens33: 10.10.20.11
    - ens35: 192.168.17.130
## Config OpenStack
### Cài đặt Ansible
```
apt install python3-pip
pip install ansible
mkdir -p /etc/ansible
txt="[defaults]\nhost_key_checking=False\npipelining=True\nforks=100"
echo -e $txt >> /etc/ansible/ansible.cfg
```
### Hardening cho các node cài đặt OpenStack
```
mkdir -p /root/inventory
vim /root/inventory/hardening
[all:vars]
ansible_connection=ssh
ansible_ssh_port=22
ansible_ssh_user=root
ansible_ssh_pass=hadt

[all]
10.10.10.10 hostname=controller
10.10.10.11 hostname=compute
```
### Cấu hình hostname cho các host
```
ansible -i /root/inventory/hardening -m shell -a "hostnamectl set-hostname {{ hostname }}" all
```
### Cấu hình selinux
```
ansible -i /root/inventory/hardening -m raw -a "sed -i 's/^SELINUX=./*/SELINUX=disabled' /etc/default/grub && sudo update grub" all
```
### Cấu hình tắt firewall 
```
ansible -i /root/inventory/hardening -m service -a "name=ufw state=stopped enabled=no" all
```
### Cài một số gói cần thiết
```
ansible -i /root/inventory/hardening -m shell -a "apt-get install -y software-properties-common" all
ansible -i /root/inventory/hardening -m shell -a "apt-get update -y" all
ansible -i /root/inventory/hardening -m shell -a "apt-get install -y git wget gcc python3-dev python3-pip" all
```
### Cài đặt kolla-ansible
```
pip3 install "kolla-ansible==9.3.2"
mkdir -p /etc/kolla
cp /usr/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
mkdir -p /root/inventory
cp /usr/local/share/kolla-ansible/ansible/inventory/multinode /root/inventory
```
## Triển khai Cloud OpenStack
- Chọn node controller là note triển khai. Trên note triển khai cần cài đặt kolla-ansible. Các node cài đặt OpenStack đều cần thực hiện cấu hình và cài các gói cần thiết trước bước triển khai
### Triển khai inventory
- Mở file `file /root/inventory/multinode` thêm vào đầu nội dung sau:
```
[all:vars]
ansible_connection=ssh
ansible_ssh_port=22
ansible_ssh_user=root
ansible_ssh_pass=hadt
```
- Khai báo địa chỉ IP của node controller vào các group control, compute, network, deployment, monitoring, storage
```
[control]
10.10.10.10

[network]
10.10.10.10

[compute]
10.10.10.11

[monitoring]
10.10.10.10

[storage]
10.10.10.10

[deployment]
10.10.10.10
```
### Khai báo cấu hình kolla
- Mở file `/etc/kolla/globals.yml`
```
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "train"
enable_chrony: "yes"
enable_haproxy: "no"
kolla_internal_vip_address: "10.10.10.10"
network_interface: "ens32"
neutron_external_interface: "ens33"
enable_neutron_provider_networks: "yes"
nova_compute_virt_type: "qemu"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_cinder_backup: "no"
enable_horizon_mistral: "{{ enable_mistral | bool }}"
enable_mistral: "yes"
```
### Thiết lập LVM cho cinder volume
```
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb
```
### Cài đặt
- Tạo password cho các dịch vụ
```
kolla-genpwd
```
- Thực hiện bootstrap server
```
kolla-ansible -i /root/inventory/multinode bootstrap-servers
```
- Triển khai OpenStack
```
kolla-ansible -i /root/inventory/multinode deploy
```
- Thiết lập môi trường OpenStack
```
kolla-ansible -i /root/inventory/multinode post-deploy
```
### Cài đặt Openstack client
- Thiết lập môi trường python
```
cd /root
pip install virtualenv
virtualenv openstack
```
- Vào môi trường vừa tạo
```
source /root/openstack/bin/activate
```
- Cài đặt gói cần thiết
```
pip install python-openstackclient python-glanceclient python-neutronclient
```
- Kiểm tra hoạt động của OpenStack
```
source /etc/kolla/admin-openrc.sh
openstack compute service list
```
- Kết quả như sau là OpenStack đã cài đặt thành công
![image](https://github.com/DinhHa1011/OpenStack/assets/119484840/0d256845-5f09-4087-ae87-37d4e14cc025)
## Thiết lập sau khi cài đặt
- Thực import biến môi trường để tương tác với hệ thống
```
source /etc/kolla/admin-openrc.sh
```
- Kiểm tra hoạt động của OpenStack bằng lệnh `openstack token issue` sẽ ra kết quả
![image](https://github.com/DinhHa1011/OpenStack/assets/119484840/aa2d2705-ce71-4913-869f-b2db30935ac0)
- Chỉnh sửa file /usr/share/kolla-ansible/init-runonce để thiết lập external network cho VM trong hệ thống, sử dụng dải IP thuộc eth1 như đã khai báo trong cấu hình /etc/kolla/globals.yml trước khi cài đặt
```
EXT_NET_CIDR=${EXT_NET_CIDR:-'10.10.20.0/24'}
EXT_NET_RANGE=${EXT_NET_RANGE:-'start=10.10.20.110,end=10.10.20.119'}
EXT_NET_GATEWAY=${EXT_NET_GATEWAY:-'10.10.20.1'}
```
- Cũng bỏ tham số `--no-dhcp` trong file `/usr/share/kolla-ansible/init-runonce`
```
openstack subnet create --no-dhcp \
    --allocation-pool ${EXT_NET_RANGE} --network public1 \
    --subnet-range ${EXT_NET_CIDR} --gateway ${EXT_NET_GATEWAY} public1-subnet
```
- Chạy script để khởi tạo network, flavor, tải image. Quá trình chạy script cần nhập passphase:
```
bash /usr/share/kolla-ansible/init-runonce
```
- Tạo VM
```
openstack server create \
    --image cirros \
    --flavor m1.tiny \
    --key-name mykey \
    --network demo-net \
    demo1
```
## Một số lưu ý
- Mật khẩu tài khoản admin để đăng nhập và OpenStack trong file `/etc/kolla/admin-openrc.sh`
- Link đăng nhập: `http://10.10.10.10`
- Mật khẩu của các project khác trong OpenStack ở file `/etc/kolla/passwords.yml`
