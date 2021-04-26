---
layout: post
category: openstack
title: kolla-ansible Train部署及配置说明
tagline: by 噜噜噜
tags: 
  - xx
published: true
---

<!--more-->

### 一、环境

##### 1、节点规划

| **172.16.63.201** | **部署节点**            | centos7 |
| ----------------- | ----------------------- | ------- |
| **172.16.58.1**   | **controllercompute01** | centos7 |
| **172.16.58.2**   | **controllercompute02** | centos7 |
| **172.16.58.3**   | **controllercompute03** | centos7 |

##### 2、部署节点版本信息

| name          | 查询命令                      | 版本    |
| ------------- | ----------------------------- | ------- |
| ansible       | ansible --version             | 2.9.10  |
| kolla-ansible | pip list \|grep kolla-ansible | 9.2.0   |
| python        |                               | 2.7.5   |
| docker        | docker -v                     | 19.03.6 |

kolla版本与openstack版本对应关系：https://releases.openstack.org/teams/kolla.html

### 二、准备节点

##### 1、配置部署节点

1.1 基础环境配置

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g'  /etc/selinux/config
systemctl stop firewalld
systemctl stop NetworkManager
systemctl disable ntpd
systemctl disable firewalld
systemctl disable iptables
systemctl disable ip6tables
systemctl disable NetworkManager
```

1.2 配置yum源

```
rm -f /etc/yum.repos.d/*.repo
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/docker-ce.repo \
https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

1.3 配置dns

1.4 安装软件

```
yum install -y epel-release 
yum install -y python2-pip
mkdir ~/.pip
tee ~/.pip/pip.conf <<-EOF
[global]
index-url = https://pypi.douban.com/simple
EOF
pip install --upgrade pip
pip install ansible==2.9.10
yum install -y python-devel libffi-devel gcc openssl-devel git libselinux-python vim
```

1.5 配置/etc/hosts

```
172.16.63.201 deploy.iaas.com.cn
172.16.58.1     controllercompute01
172.16.58.2     controllercompute02
172.16.58.3     controllercompute03
```

1.6 配置免密

1.7 将/etc/hosts文件传到所有openstack节点

1.8 安装并配置好docker

```
yum install docker-ce   
mkdir -p /etc/systemd/system/docker.service.d
tee /etc/systemd/system/docker.service.d/kolla.conf <<-'EOF'
[Service]
MountFlags=shared
EOF

vim /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock  --insecure-registry deploy.iaas.com.cn:4000  --registry-mirror=http://f1361db2.m.daocloud.io

mkdir /etc/docker/
cat <<EOF>/etc/docker/daemon.json 
{
 "log-driver": "json-file",
 "log-opts": {
  "max-size": "10m",
  "max-file": "5"
 }
}
EOF

systemctl daemon-reload
systemctl restart docker
systemctl enable docker.service
```

1.9 配置docker本地仓库

将打包好的kolla镜像tar包上传并解压到本地/opt/registry/下

tar包地址：

链接：https://pan.baidu.com/s/1DKz46diPfqe64jGH_Q_tYA 
提取码：uj1m

```
mkdir /opt/registry/
tar -zxf  kolla-train.tar.gz -C /opt/registry/

docker run -d -p 4000:5000 -v /opt/registry:/var/lib/registry --restart=always \
 --name registry  registry:latest
 
 curl 172.16.63.201:4000/v2/_catalog    #验证是否成功
```

![](https://i.loli.net/2020/09/03/suqVWCX2mHcijTg.png)

1.10 安装kolla-ansible

```
pip install kolla-ansible==9.2.0
```

##### 2、配置openstack节点

2.1 基本配置<和部署节点一样>

2.2 开启虚拟化

```
modprobe kvm_intel
确认虚拟化打开lsmod |grep kvm
```

2.3 配置yum源、dns、pip

2.4 安装docker

2.5 安装其他软件

```
yum install -y python-devel libffi-devel gcc openssl-devel git libselinux-python vim
```

### 三、在部署节点上配置kolla

##### 1、编辑主机角色

```
mkdir -p /etc/kolla/inventory
cp /usr/share/kolla-ansible/ansible/inventory/multinode  /etc/kolla/inventory/deploy-multinode
```

vim /etc/kolla/inventory/deploy-multinode   **##一定要加上ansible_ssh_user=root**

```
[control]
controllercompute[01:03] ansible_ssh_user=root
[network]
controllercompute[01:03] ansible_ssh_user=root
[compute]
controllercompute[01:03] ansible_ssh_user=root
[monitoring]
controllercompute[01:03] ansible_ssh_user=root
[storage]
controllercompute[01:03] ansible_ssh_user=root
```

##### 2、设置服务密码

```
cp /usr/share/kolla-ansible/etc_examples/kolla/passwords.yml /etc/kolla/
kolla-genpwd #生成随机密码
【可选】修改openstack第一个admin用户的密码为你需要的，其他密码比如数据库的密码也在该文件上改
```

##### 3、配置globals.yml文件

```
cp /usr/share/kolla-ansible/etc_examples/kolla/globals.yml /etc/kolla/
```

这里使用第三方ceph为后端，网络设备使用OVS的openstack环境。

cat /etc/kolla/globals.yml  | grep -v ^$| grep -v ^#

---
```
config_strategy: "COPY_ALWAYS"
kolla_base_distro: "centos"
kolla_install_type: "source"   ##事实证明，源代码版本比二进制代码更可靠
openstack_release: "train"
kolla_internal_vip_address: "172.16.58.10"  ##VIP
docker_registry: "deploy.iaas.com.cn:4000"   ##仓库地址
docker_namespace: "bonc"    ##仓库namespace
network_interface: "bond_Oapistorag"
kolla_external_vip_interface: "{{ network_interface }}"
api_interface: "{{ network_interface }}"
storage_interface: "{{ network_interface }}"
tunnel_interface: "bond_Ovpc"
dns_interface: "{{ network_interface }}"
neutron_external_interface: "bond_Otrunk"
keepalived_virtual_router_id: "46"   ##虚拟路由id
enable_aodh: "yes"
enable_blazar: "yes"
enable_ceilometer: "yes"
enable_ceilometer_ipmi: "yes"
enable_central_logging: "yes"
enable_cinder: "yes"
enable_cinder_backend_iscsi: "no"
enable_cloudkitty: "yes"
enable_cyborg: "yes"
enable_etcd: "no"
enable_gnocchi: "yes"
enable_grafana: "yes"
enable_ironic: "yes"
enable_ironic_ipxe: "yes"
enable_iscsid: "no"
enable_kuryr: "yes"
enable_magnum: "yes"
enable_masakari: "yes"
enable_monasca: "no"
enable_neutron_vpnaas: "yes"
enable_neutron_dvr: "yes"
enable_neutron_qos: "yes"
enable_panko: "yes"
enable_prometheus: "yes"
glance_backend_ceph: "yes"
glance_backend_file: "no"
gnocchi_backend_storage: "ceph"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
ironic_dnsmasq_dhcp_range: "192.168.5.100,192.168.5.110"
ironic_inspector_kernel_cmdline_extras: ['ipa-lldp-timeout=90.0', 'ipa-collect-lldp=1']
```



**注意：**keepalived_virtual_router_id必须唯一，同一网段中不应有相同virtual_router_id的集群，通过tcpdump  -i bond_Oapi -n vrrp来检测是否被占用的ID号

Kolla现在支持许多OpenStack服务：https://github.com/openstack/kolla-ansible/blob/master/README.rst#openstack-services



### 四、配置文件说明及定制化配置

**##文件配置的参考：https://docs.openstack.org/kolla-ansible/train/**

##### 1、管理员引导

- **高级配置**：https://docs.openstack.org/kolla-ansible/train/admin/advanced-configuration.html

  - endpoint、TLS配置

  - 自定义参数配置 node_custom_config: "/etc/kolla/config" 

    - 方式1：/etc/kolla/config/configfile；

      如/etc/kolla/config/nova.conf

    - 方式2：/etc/kolla/config/service-name/configfile；

      如：/etc/kolla/config/nova/nova-scheduler.conf

    - 方式3：/etc/kolla/config/service-name/hostname/configfile;

      如：/etc/kolla/config/nova/controller-0002/nova.conf 

      ​       /etc/kolla/config/nova/controller-0003/nova.conf

  

  ​		举例：配置cpu和内存的超配

  ​		vim /etc/kolla/config/nova/myhost/nova.conf

  ```
  [DEFAULT]
  cpu_allocation_ratio = 16.0
  ram_allocation_ratio = 5.0
  ```

  - 外部elk环境

  - 不使用默认的端口：setting `<service>_port` in `globals.yml` file ；

    ​	例如database_port: 3307

  - 为容器挂载额外的卷

- **生产环境架构**：https://docs.openstack.org/kolla-ansible/train/admin/production-architecture-guide.html

  - 网卡接口配置： **##注意ansible无法识别带有-的名卡名称，可识别下划线_**

  - ip地址配置：从T版本开始，已经支持用ipv6代替ipv4;IPv6支持需要Ansible 2.8或更高版本

  - HA架构：

    - keepalived和haproxy来支持openstack服务的高可用

    - 核心服务：mariadb和rabbitMQ有自己的高可用架构；**【需要遵循CAP原则，提供仲裁功能】**

      **说明：仲裁需要多数票，因此部署其中的两个实例不会提供（默认情况下）任何高可用性，因为其中一个失败会导致另一个失败；建议的实例数为3，其中1个节点故障是可以接受的。为了扩展目的和更好的弹性，可以使用5个节点并接受2个故障**

##### 2、用户引导

2.1 安装依赖和pip、ansible

```
sudo yum install python-devel libffi-devel gcc openssl-devel libselinux-python
```

2.1.1 不使用虚拟环境

```
sudo easy_install pip
sudo pip install -U pip
sudo pip install 'ansible<2.10'  ##或者使用yum安装ansible
```

2.1.2 使用虚拟环境**<推荐>**

```
sudo yum install python-virtualenv
virtualenv /path/to/virtualenv
source /path/to/virtualenv/bin/activate
pip install -U pip
pip install 'ansible<2.10'  ##Kolla Ansible requires Ansible 2.6 to 2.9.
或者使用yum安装ansible
```

2.2 安装kolla-ansible

```
pip install kolla-ansible==x.x.x
```

2.3 配置ansible

  vim /etc/ansible/ansible.cfg

```
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

##### 3、服务配置<官网>

###### 3.1 计算服务

​	https://docs.openstack.org/kolla-ansible/train/reference/compute/index.html

3.1.1  Libvirt TLS 

3.1.2  Masakari 服务  <虚拟机高可用>

​	默认情况下，masakari-instancemonitor将使用qemu + tcp：//连接URI连接到libvirt守护程序，以获取基于KVM的虚拟机的事件

​	vim  /etc/kolla/config/masakari/masakari-monitors.conf

```
[libvirt]
connection_uri = "xen://{{ migration_interface_address | put_address_in_context('url') }}/system"
```

3.1.3  nova cells 

https://docs.openstack.org/kolla-ansible/train/reference/compute/nova-cells-guide.html

3.1.4  nova compute 

nova_compute_virt_type:     qemu, kvm, vmware,xenapi

https://docs.openstack.org/kolla-ansible/train/reference/compute/nova-guide.html

3.1.5 zun 容器服务

​	它旨在提供一个OpenStack API，用于在OpenStack上配置和管理容器化工作负载

```
enable_zun: "yes"
enable_kuryr: "yes"
enable_etcd: "yes"
docker_configure_for_zun: "yes"
```

###### 3.2  裸金属服务

​	https://docs.openstack.org/kolla-ansible/train/reference/bare-metal/ironic-guide.html

###### 3.3  存储服务

3.3.1  外部ceph

   Glance, Cinder, Nova, Gnocchi, Manila  这些服务都可能使用ceph存储

```
enable_ceph: "no"
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
gnocchi_backend_storage: "ceph"
enable_manila_backend_cephfs_native: "yes"
```



GLANCE：

**/etc/kolla/config/glance/glance-api.conf**

```
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
```

**将ceph.conf和ceph.client.glance.keyring 文件放在/etc/kolla/config/glance/下**



CINDER:

```
vim /etc/kolla/config/cinder/cinder-volume.conf
[DEFAULT]
enabled_backends=rbd-1

[rbd-1]
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=cinder
backend_host=rbd:volumes
rbd_pool=volumes
volume_backend_name=rbd-1
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_secret_uuid = {{ cinder_rbd_secret_uuid }}   ##后者在password.yml文件中查找cinder_rbd_secret_uuid
```

vim  /etc/kolla/config/cinder/cinder-backup.conf

```
[DEFAULT]
backup_ceph_conf=/etc/ceph/ceph.conf
backup_ceph_user=cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool=backups
backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
```

将ceph.conf文件拷贝到**/etc/kolla/config/cinder/**下

将ceph.conf文件同时拷贝到**/etc/kolla/config/cinder/cinder-volume** 和 **/etc/kolla/config/cinder/cinder-backup**下

将ceph.client.cinder-backup.keyring 和ceph.client.cinder.keyring 拷贝到**/etc/kolla/config/cinder/cinder-backup/**下   【cinder-backup需要两个密钥环来访问卷和备份池】

将ceph.client.cinder.keyring 拷贝到**/etc/kolla/config/cinder/cinder-volume** 下

注意：####将文件命名为ceph.client *很重要 ####



NOVA:

```
 ls /etc/kolla/config/nova
 ceph.client.cinder.keyring      ceph.client.nova.keyring    ceph.conf
```

**/etc/kolla/config/nova/nova-compute.conf**

```
[libvirt]
images_rbd_pool=vms
images_type=rbd
images_rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=nova
```



GNOCCHI:

vim /etc/kolla/config/gnocchi.conf

```
[storage]
driver = ceph
ceph_username = gnocchi
ceph_keyring = /etc/ceph/ceph.client.gnocchi.keyring
ceph_conffile = /etc/ceph/ceph.conf
```

```
$ ls /etc/kolla/config/gnocchi
ceph.client.gnocchi.keyring    ceph.conf    gnocchi.conf
```



MANILA:

```
enable_manila_backend_cephfs_native=true
```

1. Configure CephFS backend, setting `enable_manila_backend_cephfs_native`
2. Create Ceph configuration file in `/etc/ceph/ceph.conf`
3. Create Ceph keyring file in `/etc/ceph/ceph.client.<username>.keyring`
4. Setup Manila in the usual way

https://docs.openstack.org/kolla-ansible/train/reference/storage/manila-guide.html



3.3.2  cinder

​	LVM/NFS

https://docs.openstack.org/kolla-ansible/train/reference/storage/cinder-guide.html

3.3.3  swift

https://docs.openstack.org/kolla-ansible/train/reference/storage/swift-guide.html

3.3.4 manila <共享文件系统服务>

https://docs.openstack.org/kolla-ansible/train/reference/storage/manila-guide.html



###### 3.4 网络服务

https://docs.openstack.org/kolla-ansible/train/reference/networking/index.html

3.4.1 designate 

3.4.2 DPDK

3.4.3 neutron 

3.4.4 octavia

3.4.5 Opendaylight

3.4.6 SRIOV



###### 3.5 通用服务

https://docs.openstack.org/kolla-ansible/train/reference/shared-services/index.html

3.5.1 glance

3.5.2 horizon

3.5.3 keystone

3.5.4  外部mariadb

https://docs.openstack.org/kolla-ansible/train/reference/databases/external-mariadb-guide.html

3.5.5  rabbitmq

https://docs.openstack.org/kolla-ansible/train/reference/message-queues/rabbitmq.html





###### 3.6 日志和监控

https://docs.openstack.org/kolla-ansible/train/reference/logging-and-monitoring/index.html



###### 3.7 容器服务

https://docs.openstack.org/kolla-ansible/train/reference/containers/index.html

Kuryr



###### 3.8 容器的资源限制

https://docs.openstack.org/kolla-ansible/train/reference/deployment-config/resource-constraints.html

### 五、部署ceph

使用 ceph-deploy部署ceph。注意此处要部署N版及以上

部署完之后：

**创建池**

```
ceph osd pool create images 64 64    ##注意要是2的平方
ceph osd pool create backups 64 64
ceph osd pool create vms 64 64
ceph osd pool create volumes  64 64
```



```
[cephbonc@ceph-nautilus-deploy ceph]$ ceph osd lspools
15 volumes
16 vms
17 images
18 backups
```

**创建认证文件**

```
ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images' mgr 'profile rbd pool=images' > ceph.client.glance.keyring

ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' mgr 'profile rbd pool=volumes, profile rbd pool=vms' > ceph.client.cinder.keyring



ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=backups' mgr 'profile rbd pool=backups' > ceph.client.cinder-backup.keyring

ceph auth get-or-create client.nova mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rx pool=images' > ceph.client.nova.keyring
```

```
[root@ceph_nautilus_node_1 ceph]# ll
total 24
-rw------- 1 root root 151 Sep 11 17:49 ceph.client.admin.keyring
-rw-r--r-- 1 root root  71 Sep 15 20:13 ceph.client.cinder-backup.keyring
-rw-r--r-- 1 root root  64 Sep 15 20:13 ceph.client.cinder.keyring
-rw-r--r-- 1 root root  72 Sep 15 20:11 ceph.client.glance.keyring
-rw-r--r-- 1 root root  62 Sep 16 11:09 ceph.client.nova.keyring
-rw-r--r-- 1 root root 431 Sep 11 17:54 ceph.conf
-rw-r--r-- 1 root root  92 Aug 11 04:44 rbdmap
-rw------- 1 root root   0 Sep 11 16:39 tmpQZjcR8
```

将认证文件和ceph配置文件拷贝到kolla部署节点上对应的文件中

### 六、部署

##### 1、修改一些tasks

修改docker官方yum源为阿里云yum源，另外配置docker镜像加速，指定使用阿里云镜像加速

```
sed -i 's/^docker_yum_url/#&/' /usr/share/kolla-ansible/ansible/roles/baremetal/defaults/main.yml
sed -i 's/^docker_custom_config/#&/' /usr/share/kolla-ansible/ansible/roles/baremetal/defaults/main.yml

cat >> /usr/share/kolla-ansible/ansible/roles/baremetal/defaults/main.yml <<EOF
docker_yum_url: "https://mirrors.aliyun.com/docker-ce/linux/{{ ansible_distribution | lower }}"
docker_custom_config: {"registry-mirrors": ["https://uyah70su.mirror.aliyuncs.com"]}
EOF
```

##### 2、部署

```
kolla-ansible -i /etc/kolla/inventory/deploy-multinode bootstrap-servers

#部署检查
kolla-ansible -i /etc/kolla/inventory/deploy-multinode prechecks


#执行部署
kolla-ansible -i /etc/kolla/inventory/deploy-multinode deploy

#生成认证文件
kolla-ansible post-deploy

scp /etc/kolla/admin-openrc.sh  ##到全部的openstack控制节点上
```

##### 3、安装openstack客户端

```
#启用openstack存储库
yum install -y centos-release-openstack-train

#安装openstack客户端
yum install -y python-openstackclient python-glanceclient python-neutronclient

#启用selinux,安装openstack-selinux软件包以自动管理OpenStack服务的安全策略
yum install -y openstack-selinux

#报错处理
pip uninstall urllib3
yum install -y python2-urllib3
```

