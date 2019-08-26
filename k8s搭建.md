## 基于centos7  Vagrant +virtualbox 安装k8s集群

#### 环境及版本

- centos: CentOS Linux release 7.6.1810 (Core)
- Virtualbox：6.0.4
- Vagrant：2.2.3
- Docker：1.13.1
- Etcd：3.3.1
- Kube-apiserver：
- Kube-controller-manager:
- Kube-scheduler:
- Flannel:
- Kubelet:

#### 安装

###### 一.安装vagrant virtualbox (略)

[vagrant](https://www.vagrantup.com/)

[virtualbox](https://www.virtualbox.org/wiki/Downloads)



###### 二.安装centos 以及 集群节点的创建

- 创建工作目录，我们新建一个目录当做项目目录的基准位置

  ```
  mkdir vagrant_root
  cd vagrant_root
  ```

- 初始化操作系统

  初次安装时会生成vagrantfile

  `例：vagrant init centos/7`

  可以配置 vagrantfile 来生成所需要的节点 如下:

  ```
  x # -*- mode: ruby -*-# vi: set ft=ruby :MASTER_COUNT = 3Vagrant.configure("2") do |config|    config.ssh.insert_key = false     config.ssh.private_key_path = ["/home/humanbrain/vagrantbox_root/ssh_key","~/.vagrant.d/insecure_private_key"]    config.vm.provision "file", source: "/home/humanbrain/vagrantbox_root/ssh-key.pub", destination: "~/.ssh/authorized_keys"    config.vm.provision "shell", inline: <<-EOC        sudo sed -i "s/\#PasswordAuthentication yes/PasswordAuthentication yes/g" /etc/ssh/sshd_config        sudo sed -i "s/\#PermitRootLogin yes/PermitRootLogin yes/g" /etc/ssh/sshd_config        sudo service sshd restart        sudo mkdir -p /root/.ssh/        sudo cp -r /home/vagrant/.ssh/authorized_keys /root/.ssh/authorized_keys  EOC      (1..MASTER_COUNT).each do |i|        config.vm.define "kmaster#{i}" do |node|                node.vm.network :private_network, ip: "192.168.98.#{30+i}"                node.vm.hostname = "kmaster#{i}"                #node.vm.provision :shell, inline: "cat /vagrant/ssh-key.pub >> .ssh/authorized_keys"        node.vm.provider "virtualbox" do |vb|                  vb.memory = 4096                  vb.cpus = 3                end        end      end          config.vm.define "knode1" do |node|                node.vm.network :private_network, ip: "192.168.98.20"                node.vm.hostname = "knode1"                node.vm.provider "virtualbox" do |vb|                  vb.memory = 6096                  vb.cpus = 3                 # vb.pci :bus => '0x06', :slot => '0x00', :function => '0x0'                 # vb.kvm_hidden = "true"        end        node.vm.provision "shell", inline: <<-SHELL        sudo yum install -y pciutils        rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm                SHELL	        end  config.vm.provision "shell", inline: <<-SHELL    yum install -y net-tools wget    sudo sudo sed -i "s/obsoletes=1/obsoletes=0/g" /etc/yum.conf  SHELLend
  ```

  修改完毕 vagrantfile 记得重启虚拟机：

  `vagrant reload`

  执行命令创建虚拟节点：

  `vagrant up`

  

###### 三.安装组件

​	

| 角色               | 安装组件名称                                                 |
| ------------------ | ------------------------------------------------------------ |
| master（管理节点） | Docker： Etcd：Kube-apiserver： Kube-controller-manager: Kube-scheduler: Flannel: Kubelet |
| node1 （计算节点） | Docker： Etcd：Kube-apiserver： Kube-proxy: Flannel: Kubelet |
| node2 （计算节点） | Docker： Etcd：Kube-apiserver： Kube-proxy: Flannel: Kubelet |



- 安装etcd

  ```
  yum install etcd #默认安装到 /usr/bin
  
  etcd --version #查看版本 验证是否成功
  
  systemd enable etcd #设置开机启动
  
  
  ```

  

- 安装docker 

   

  ```
  Docker 要求 CentOS 系统的内核版本高于 3.10 ，查看本页面的前提条件来验证你的CentOS 版本是否支持 Docker 。
  
  uname -r #查看你当前的内核版本
  
  yum update # 更新yum源
  
  yum install docker #直接从yum源安装最新版本
  
  docker version #查看版本
  ```

  使用docker ps 查看时候会出现：Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

  重启docker即可

  `systemctl restart docker `

  同样设置开机启动

  

-  安装kubelet

  1.我们需要在所有节点上安装 **kubelet**、**kubeadm** 和 **kubectl**，它们作用分别如下：

  - **kubeadm**：用来初始化集群（**Cluster**）
  - **kubelet**：运行在集群中的所有节点上，负责启动 **pod** 和 容器。
  - **kubectl**：这个是 **Kubernetes** 命令行工具。通过 **kubectl** 可以部署和管理应用，查看各种资源，创建、删除和更新各种组件。

  

  2.配置镜像

  ```
  cat <<EOF > /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
  EOF
  ```

  3.安装：

  `yum install -y kubectl kubelet kubeadm`

  4.设置开机启动：

  `systemctl enable kubelet`

  5.启动：

  `systemctl start kubelet`

  

  **修改sysctl配置：**

  ```
  对于 RHEL/CentOS 7 系统，可以会由于 iptables 被绕过导致网络请求被错误的路由。所以还需执行如下命令保证 sysctl 配置中 net.bridge.bridge-nf-call-iptables 被设为1。
  
  
  
  （1）使用 vi 命令编辑相关文件：
  	vi /etc/sysctl.conf
  （2）在文件中添加如下内容后，保存退出。
  	net.bridge.bridge-nf-call-ip6tables = 1
  	net.bridge.bridge-nf-call-iptables = 1
  	net.ipv4.ip_forward = 1
  （3）最后执行如下命令即可：
  	sysctl --system
  ```

  **关闭swap**

  1.执行命令将其关闭：

  `swapoff -a`

  2.编辑/etc/fstab 文件，机器重启后swap不会自动打开

  ```
  vi /etc/fstab
  
  
  将 /dev/mapper/centos-swap swap swap default 0 0 这一行前面加个 # 号将其注释掉。
  
  ```

  

###### 四.Master节点的安装配

​	1.初始化Master

- 我们在 **Master** 上执行如下命令进行初始化：

  **注意**：**--pod-network-cidr=10.244.0.0/16** 是 **k8s** 的网络插件所需要用到的配置信息，用来给 **node** 分配子网段。然后我们这边用到的网络插件是 **flannel**，就是这么配。

  `kubeadm init --pod-network-cidr=10.244.0.0/16`

  

- 初始化的时候 **kubeadm** 会做一系列的校验，以检测你的服务器是否符合 **kubernetes** 的安装条件，检测结果分为 **[WARNING]** 和 **[ERROR]** 两种。其中 **[ERROR]** 部分要予以解决。

  ![1566799498649](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1566799498649.png)

- 上图我主要有两个error：

  - docker 未启动：执行systemctl start docker.server   加入开机启动：
  - swap需要关闭 执行swapoff- a即可

- 所有 **error** 解决后，再执行最开始的 **init** 命令后 **kubeadm** 就开始安装了。但通常这时还是会报错，这是因为国内 **gcr.io**无法访问（谷歌自己的容器镜像仓库），造成镜像下载不下来。

  ![1566799843454](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\1566799843454.png)

- 我们可以通过国内厂商提供的 **kubernetes** 的镜像服务来下载，比如第一个 **k8s.gcr.io/kube-apiserver:v1.14.1** 镜像，可以执行如下命令从阿里云下载：

  `docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.14.1`

- 镜像下载下来以后再通过 **docker tag** 命令将其改成kudeadm安装时候需要的镜像名称。

  ```
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.14.1 k8s.gcr.io/kube-apiserver:v1.14.1
  ```

  

- 其它缺失的镜像也依照上面步骤进行操作。

  ```
  查看所需的镜像及版本：
  
  kubeadm config images list
  
  
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:v1.15.3
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:v1.15.3
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:v1.15.3
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:v1.15.3
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:3.3.10
  docker pull coredns/coredns:1.3.1
  
  
  
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:v1.15.3 k8s.gcr.io/kube-proxy:v1.15.3
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:v1.15.3 k8s.gcr.io/kube-scheduler:v1.15.3
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:v1.15.3 k8s.gcr.io/kube-apiserver:v1.15.3
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:v1.15.3 k8s.gcr.io/kube-controller-manager:v1.15.3
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:3.3.10  k8s.gcr.io/etcd:3.3.10
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1  k8s.gcr.io/pause:3.1
  docker tag docker.io/coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
  
  ```

  

- 镜像全部下载完毕后，再执行最开始的 **init** 命令后 **kubeadm** 就能成功安装了。最后一行，**kubeadm** 会提示我们，其他节点需要加入集群的话，只需要执行这条命令就行了，同时里面包含了加入集群所需要的 **token**（这个要记下来）。

  

​     2.配置kubectl

​	 	嗷嗷

​	

