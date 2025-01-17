### Kubernetes高可用集群

Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。

通过Kubernetes你可以：

- 快速部署应用
- 快速扩展应用
- 无缝对接新的应用功能
- 节省资源，优化硬件资源的使用



本工具使用ansible playbook初始化系统配置、安装k8s高可用集群，并可进行节点扩容、替换集群证书等。

版本说明：

| 名称       | 版本       |
| ---------- | ---------- |
| kubernetes | 1.15.3     |
| docker     | 19.03.1    |
| system     | CentOS 7.6 |



## 使用方法：

### 一、准备资源

1.1、准备机器资源，机器需要使用一块数据盘用来存放数据

1.2、配置master负载均衡，公有云可以直接使用云负载均衡产品，自建机房等需配置haproxy等（后期支持自动配置haproxy）

1.3、请按照inventory格式将以上准备资源填写

```
#本组内填写etcd服务器及主机名
[etcd]
172.17.15.233 hostname=etcd-01
172.17.15.234 hostname=etcd-02
172.17.15.235 hostname=etcd-03
172.17.15.236 hostname=etcd-04
172.17.15.237 hostname=etcd-05

#本组内填写master服务器及主机名
[master]
172.17.15.238 hostname=master-01
172.17.15.239 hostname=master-02
172.17.15.240 hostname=master-03
172.17.15.241 hostname=master-04
172.17.15.242 hostname=master-05

#本组机器不会进行系统初始化等操作，仅用做安装kubectl命令行
[kubectl]
172.17.15.238 hostname=master-01

#本组机器不会进行系统初始化等操作，只是apiserver证书签发时使用
[k8s_service]
10.64.0.1        #shoule be k8s servcie first ip
172.17.15.246    #shoule be k8s apiserver slb ip

[haproxy]
172.17.15.247 hostname=haproxy-01 type=MASTER priority=100
172.17.15.248 hostname=haproxy-02 type=BACKUP priority=90
[haproxy:vars]
vip=172.17.15.10

#本组内填写node服务器及主机名
[node]
172.17.15.243 hostname=node-01
172.17.15.244 hostname=node-02
172.17.15.245 hostname=node-03
```



###  二、修改相关配置

编辑group_vars/all.yml文件，填入自己的参数

| 配置项                   | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| disk                     | 指定机器数据盘盘符。本脚本会自动格式化并挂载磁盘             |
| data_dir                 | 指定机器数据盘挂在目录。本脚本会自动格式化并挂载磁盘         |
| gpgkey                   | 选择使用vpc内网软件源还是外网软件源                          |
| download_url             | k8s二进制文件下载地址，默认是官方下载地址，可能会比较慢或者下载失败，可自己先行下载配置文件服务器. |
| docker_version           | 可通过查看版本yum list docker-ce.x86_64 --showduplicates     |
| ssl_dir                  | 签发ssl证书保存路径，ansible控制端机器上的路径               |
| ssl_days                 | 签发ssl的有效期（单位：天）                                  |
| kube_dir                 | kubernetes相关配置文件存放目录，证书存在此路径下的ssl目录下  |
| kube_data_dir            | kubernetes数据存放目录                                       |
| apiserver_domain_name    | apiserver域名，签发证书和配置node节点连接master时会用到      |
| service_cluster_ip_range | 指定k8s集群service的网段                                     |
| pod_cluster_cidr         | 指定k8s集群pod的网段                                         |
| cluster_dns              | 指定集群dns服务IP                                            |
| harbor_url               | 镜像仓库地址(包含group)，例如：registry.k8sre.com/library    |
| pause_image              | 指定pause镜像名称及tag，与harbor_url拼接成完整镜像地址       |

- 注：以下程序默认数据目录

- etcd数据目录: ${kube_data_dir}/etcd

  docker数据目录: ${kube_data_dir}/docker

  kubelet数据目录: ${kube_data_dir}/kubelet

- 下载路径：

  ```
  ${download_url}/kubectl
  ```

  注：自行去https://github.com/kubernetes/kubernetes下载对应版本，将二进制文件解压至下载服务器对应目录

### 三、使用方法

#### 3.1、安装ansible

在控制端机器执行以下命令安装ansible

```
yum -y install python-devel python-pip
pip install ansible
```

#### 3.2、部署集群

以下步骤都可单独执行，除系统初始化外，其他都可重复执行。也可单独指定tag执行部分模块

1、系统初始化

```
ansible-playbook k8s.yml -i inventory -t init
```

2、签发证书

```
ansible-playbook k8s.yml -i inventory -t cert
```

3、安装etcd

```
ansible-playbook k8s.yml -i inventory -t install_etcd
```

4、安装master节点

```
ansible-playbook k8s.yml -i inventory -t install_master
```

5、安装node节点

```
ansible-playbook k8s.yml -i inventory -t install_node
```

6、全部安装

```
ansible-playbook k8s.yml -i inventory
```

如是公有云环境，则执行：

```
ansible-playbook k8s.yml -i inventory --skip-tags=install_haproxy,install_keepalived
```

⚠️：默认使用calico网络插件，可自行下载flannel yaml安装flannel网络插件

7、扩容mater节点

```
ansible-playbook k8s.yml -i inventory -t init -l master
ansible-playbook k8s.yml -i inventory -t cert,install_master 
```

8、扩容node节点

```
ansible-playbook k8s.yml -i inventory -t init -l node
ansible-playbook k8s.yml -i inventory -t cert,install_node
```

#### 3.3、替换集群证书

先删除ca、apiserver证书，然后执行以下步骤

```
ansible-playbook k8s.yml -i inventory -t cert
ansible-playbook k8s.yml -i inventory -t dis_certs,restart_master,restart_node,restart_etcd
```



### kubernetes HA架构

![k8s](kubernetes.png)



### 容器云平台架构

![business](business.jpeg)