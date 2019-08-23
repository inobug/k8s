## 基于centos7  Vagrant +virtualbox 安装k8s集群

#### 环境及版本

- centos: CentOS Linux release 7.6.1810 (Core)
- Virtualbox：6.0.4
- Vagrant：2.2.3
- Docker：
- Etcd：
- Kube-apiserver：
- Kube-controller-manager:
- Kube-scheduler:
- Flannel:
- Kubelet:

#### 安装

###### 1.安装vagrant virtualbox (略)

[vagrant](https://www.vagrantup.com/)

[virtualbox](https://www.virtualbox.org/wiki/Downloads)



###### 2.安装centos 以及 集群节点的创建

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
  
  # -*- mode: ruby -*-
  # vi: set ft=ruby :
  
  MASTER_COUNT = 3
  
  Vagrant.configure("2") do |config|
    
    config.ssh.insert_key = false 
    
    config.ssh.private_key_path = ["/home/humanbrain/vagrantbox_root/ssh_key","~/.vagrant.d/insecure_private_key"]
    
    config.vm.provision "file", source: "/home/humanbrain/vagrantbox_root/ssh-key.pub", destination: "~/.ssh/authorized_keys"
    
    config.vm.provision "shell", inline: <<-EOC
      
      sudo sed -i "s/\#PasswordAuthentication yes/PasswordAuthentication yes/g" /etc/ssh/sshd_config
      
      sudo sed -i "s/\#PermitRootLogin yes/PermitRootLogin yes/g" /etc/ssh/sshd_config
      
      sudo service sshd restart
      
      sudo mkdir -p /root/.ssh/
      
      sudo cp -r /home/vagrant/.ssh/authorized_keys /root/.ssh/authorized_keys
    EOC
  
        (1..MASTER_COUNT).each do |i|
          config.vm.define "kmaster#{i}" do |node|
                  node.vm.network :private_network, ip: "192.168.98.#{30+i}"
                  node.vm.hostname = "kmaster#{i}"
                  #node.vm.provision :shell, inline: "cat /vagrant/ssh-key.pub >> .ssh/authorized_keys"
  		node.vm.provider "virtualbox" do |vb|
                    vb.memory = 4096
                    vb.cpus = 3
                  end
          end
        end
  
    
          config.vm.define "knode1" do |node|
                  node.vm.network :private_network, ip: "192.168.98.20"
                  node.vm.hostname = "knode1"
                  node.vm.provider "virtualbox" do |vb|
                    vb.memory = 6096
                    vb.cpus = 3
                   # vb.pci :bus => '0x06', :slot => '0x00', :function => '0x0'
                   # vb.kvm_hidden = "true"
  		end
     		node.vm.provision "shell", inline: <<-SHELL
  		sudo yum install -y pciutils
  		rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
  		
  		SHELL
          end
  
  
    config.vm.provision "shell", inline: <<-SHELL
      yum install -y net-tools wget
      sudo sudo sed -i "s/obsoletes=1/obsoletes=0/g" /etc/yum.conf
    SHELL
  end
  
  ```

  修改完毕 vagrantfile 记得重启虚拟机：

  `vagrant reload`

  