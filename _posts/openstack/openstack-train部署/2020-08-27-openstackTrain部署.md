---
layout: post
category: openstack
title:  Train部署
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
kolla_install_type: "source"
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



**##文件配置的参考：https://docs.openstack.org/kolla-ansible/train/**

- **高级配置**：https://docs.openstack.org/kolla-ansible/train/admin/advanced-configuration.html

  - endpoint、TLS配置
  - 自定义参数配置 node_custom_config: "/etc/kolla/config" 
    - 方式1：/etc/kolla/config/configfile；如/etc/kolla/config/nova.conf
    - 方式2：/etc/kolla/config/service-name/configfile；如：/etc/kolla/config/nova/nova-scheduler.conf
    - 方式3：/etc/kolla/config/service-name/hostname/configfile；如：/etc/kolla/config/nova/controller-0002/nova.conf 和 /etc/kolla/config/nova/controller-0003/nova.conf

  

  ​		举例：配置cpu和内存的超配

  ​		vim /etc/kolla/config/nova/myhost/nova.conf

  ```
  [DEFAULT]
  cpu_allocation_ratio = 16.0
  ram_allocation_ratio = 5.0
  ```

  - 外部elk环境
  - 不使用默认的端口：setting `<service>_port` in `globals.yml` file ；例如database_port: 3307
  - 为容器挂载额外的卷

- **生产环境架构**：https://docs.openstack.org/kolla-ansible/train/admin/production-architecture-guide.html

  - 网卡接口配置： **##注意ansible无法识别带有-的名卡名称，可识别下划线_**

  - ip地址配置：从T版本开始，已经支持用ipv6代替ipv4;IPv6支持需要Ansible 2.8或更高版本

  - HA架构：

    - keepalived和haproxy来支持openstack服务的高可用

    - 核心服务：mariadb和rabbitMQ有自己的高可用架构；**【需要遵循CAP原则，提供仲裁功能】**

      **说明：仲裁需要多数票，因此部署其中的两个实例不会提供（默认情况下）任何高可用性，因为其中一个失败会导致另一个失败；建议的实例数为3，其中1个节点故障是可以接受的。为了扩展目的和更好的弹性，可以使用5个节点并接受2个故障**

      

- 





### 四、部署

