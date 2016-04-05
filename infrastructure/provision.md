# 自动化环境搭建

## 自动化

我们来梳理一下上面这个场景里的问题：

-  开发自己不做部署
-  环境的安装是手工的
-  应用的配置信息需要手工修改

除了艺术品之外，在工业社会里，手工就意味着低效，容易犯错，且不可持续。一切可以自动化的，都应该被自动化起来。

一个软件系统往往会包含很多的组件（消息队列服务，应用服务器，数据库服务器，负载均衡器，反向代理，文件服务器等），而且每一套环境（开发环境，测试环境，UAT，Staging，生产）还有自己独立的组件。

因此一个操作系统要被配置成系统的某个组件还需要做很多工作：以Java为例，我们需要安装特定版本的JDK，设置`CLASSPATH`环境变量，修改操作系统的内核参数，创建特定用户（数据库用户等），修改一些目录的权限等等。这些操作如果交给人工来完成，必然会出现各种错误（想想这个过程要被在不同的环境中重复多遍，出错的机率会大大增加）。

事实上业界已经有了很多帮助开发/运维工程师进行环境安装的工具，比如

-  Chef
-  Puppet
-  Ansible

前两者我已经在[《轻量级Web应用开发》](http://book.douban.com/subject/26585461/)有过介绍，这里我们以`Ansible`为例来描述。

## Vagrant

`Vagrant`提供对虚拟机的封装，使用它可以很容易的通过配置的方式来定义一个`虚拟机`。

使用`Vagrant`，你只需要定义一个文本文件`Vagrantfile`即可。`Vagrant`自带的命令行工具`vagrant`会尝试加载这个文件，并按照其中的配置来启动虚拟机。`Vagrantfile`按照`ruby`的语法编写，不过不用担心，你无需在其中定义函数或者类，只需要做一些配置即可。

下面是一个简单的虚拟机定义：

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "precise64"
  config.vm.network "private_network", :ip => "192.168.2.100"
end
```

我们指定了虚拟机使用`precise64`（precise是一个ubuntu的发行版，64表示它是一个64位系统的镜像）这样一个镜像，并且给这个虚拟机分配一个私有的IP地址，这样我们就可以在宿主环境中通过这个IP来访问该虚拟机了。

定义了`Vagrantfile`之后，使用`vagrant`工具的子命令就可以启动虚拟机了

```sh
$ vagrant up
```

你可以在`VirtualBox`的界面里看到正在运行的虚拟机（Vagrant在底层使用了VirtualBox的虚拟机，而不是自行开发另外一套）：

![virtual box](images/virtualbox.png)

启动之后，你可以通过

```sh
$ vagrant ssh
```

来登录到虚拟机中。

如果你不知道使用哪个镜像，不知道如何配置`Vagrantfile`，可以使用这个命令来从头开始：

```sh
$ vagrant init hashicorp/precise64
$ vagrant up
```

`vagrant`命令会自动下载镜像，并设置环境，然后启动虚拟机。

在工程实践里，`Vagrantfile`会checkin到代码库中，这样团队里的其他人也可以很容易的在本地重新搭建相同的环境。另外，我推荐你将`box`的版本尽量和生产环境一致（比如都使用ubuntu的precise64位），这样可以尽早发现一些环境相关的问题。

### 初始化环境

`Vagrant`还提供了丰富的机制来初始化环境。你可以使用简单的`Shell`脚本，或者全功能的`Ansible`，`Chef`等来初始化环境。

设想你需要在虚拟机环境就绪后，在`vagrant`用户的home目录下创建一个叫`workers`的目录，安装一个叫`wget`的软件包，然后下载一个网络上的文件到`workers`目录。

要完成这样的动作，我们可以在当前目录（和`Vagrantfile`放在一起）创建一个`setup.sh`的脚本：

```sh
#!/usr/bin/env bash

mkdir -p ~/workers
sudo apt-get update
sudo apt-get install wget
wget http://host:port/resource.zip -O ~/workers/resource.zip
```

然后在`Vagrantfile`中加入：

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "precise64"
  config.vm.provision :shell, path: "setup.sh"
end
```

当`Vagrant`在初始化虚拟机的时候，会执行`setup.sh`，这样我们就得到了一个经过设置的环境。设置环境可能会是一个非常复杂的过程，比如安装web服务器，定义缓存目录，安装监控服务器的客户端，为某些应用程序创建专用用户，修改权限等等，如果用shell来写，会比较复杂。

`Vagrant`支持很多的`provision`的工具，比如`Ansible`来完成这种复杂的操作。

## Ansible

Ansible是一个自动化配置工具，相对于`Chef`，`Puppet`，它的安装和配置更加简单（无需在被配置的服务器安装额外的Agent程序）。它通过`ssh`将一些Ansible模块部署到远程机器上，然后执行。

使用`Ansible`可以同时配置，更新多个机器。目前很多企业都会使用各种各样的云产品，比如AWS的EC2，阿里云等，通过`Ansible`可以很容易的将这些环境配置变成自动化。在企业内部的私有云（从一台服务器划分出来的众多虚拟机）中，也可以使用`Ansible`来减少配置环境的时间，提高效率。

在一个冬日的下午，我和一个新手程序员结对在服务器上修改`tomcat`服务器的一些日志的配置，折腾了很久之后，我放弃了。我心想，要不删了`webapps`这个目录重新部署一下看看吧，可能是缓存问题也说不定。不过，头昏脑涨的我并没有发现敲入的命令是`rm -rf /usr/share/tomcat7`。新手程序员问我，`rm -rf`是什么意思？我一边用力的敲下回车键，一遍警告这个新手：“rm -rf是一个非常危险的操作，它表示要强力删除整个……”。等我发现我删除的是`tomcat7`的时候已经太晚了，我们的QA环境彻底挂了，所有人都被block住了（还好不是在其他人给客户showcase的时候）。

另一个工程师，我们姑且称之为`运维工程师`吧，花费了好几个小时来重新安装`tomcat`，以及其中的各种`jvm`参数。

我们在随后的几周里，引入了`Ansible`，这样即使头昏脑涨的程序员无意识的敲入了愚蠢而致命的命令也无所谓，我们只需要2分钟就可以配置好一个`tomcat`服务器，崭新的。

### 惯例

`Ansible`中的一些关键概念：

1.  role 定义一个角色，比如nginx就可以是一个角色，要完成nginx的安装需要很多小的步骤，这些步骤都包含在nginx这个role中
1.  inventory 定义一组环境，比如Web服务器需要三台做负载均衡，数据库由两台服务器组成等，这些都可以通过inventory文件来描述，inventory文件被称为清单文件
1.  playbook 定义在哪些inventory应用哪些role

`Ansible`中有一些惯例，遵循这些惯例有助于你快速读懂其他人写的`playbook`/`role`。

```yml
production                # 生产环境的清单文件
staging                   # staging环境的清单文件
qa                        # 测试环境的清单文件

site.yml                  # 主playbook
webservers.yml            # web服务器的playbook
dbservers.yml             # 数据库服务器的playbook

roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      # 具体任务定义
        handlers/         #
            main.yml      # 回调任务
        templates/        #  
            nginx.conf.j2 # 模板文件
        files/            #
            app.conf      # 需要拷贝到被配置环境中的文件 
        vars/             #
            main.yml      # 变量定义
        defaults/         #
            main.yml      # 低优先级变量定义
        meta/             #
            main.yml      # 元数据，用以表述作者信息，定义依赖等

    webtier/              # 另外一个role，结构和`common`一致
    monitoring/           # 用于监控的role，结构同上
```

比如一个简单的`inventory`文件看起来是这样的：

```yml
[webservers]
10.29.2.1
10.29.2.2
10.29.2.3

[dbservers]
10.29.2.4
10.29.2.5
```

没错，它就是一个简单的ini文件。如果你需要添加新的机器，只需要将域名/IP地址添加到对应的小节即可。

`Ansible`使用yml作为配置，我们来看一个playbook的例子:

```yml
-   name: webservers
    hosts: webservers
    roles:
        - roles/nginx
    user: robot
    sudo: true

-   name: dbservers
    hosts: dbservers
    roles:
        - roles/mongodb 
    user: robot
    sudo: true
    environment:
        http_proxy: http://user:pass@proxy.host:8080
        https_proxy: http://user:pass@proxy.host:8080
```

这个playbook定义了两个环境的配置信息：`webservers`和`dbservers`。`webservers`中的所有主机会被应用`nginx`角色（安装和配置nginx，并启动nginx服务），而`dbservers`中的主机会被应用`mongodb`的角色。

在`dbservers`中，我们还加入了`environment`节，其中定义了可以用在安装过程中的一些环境变量设置。

定义好之后，你可以通过下列命令来执行这个`playbook`：
```sh
ansible-playbook -i qa playbook.yml
```

### 命令

`Ansible`内置了很多常用的命令来简化配置的工作，比如安装软件包，拷贝文件，使用模板，创建用户，创建目录等。

#### 安装软件包

```yml
- name: install package
  apt: name=nginx state=present
```

`apt`命令可以用于安装一个软件包，它相当于在主机上执行`apt-get install nginx -y`。如果你需要安装多个软件包，可以采用`with_items`子命令：


```yml
- name: install packages
  apt: name={{ item }} state=present
  with_items: 
  	-  nginx
  	-  python
  	-  git
```

#### 创建目录

```yml
- name: create directory
  file: path=/home/vagrant/workers state=directory
```

#### 拷贝文件

```yml
- name: copy file to workers folder
  copy: src=resource.zip dest=/home/vagrant/resource.zip
```

如果你要使用的命令，正好`Ansible`没有内置，你还可以使用`command`命令来执行：

```yml
- name: universal command
  command: ls -alt /home/vagrant
```

如果要完成`Vagrant`小节中的例子，我们的配置看起来就是这样的：

```yml
- name: create directory
  file: path=/home/vagrant/workers state=directory

- name: install package
  apt: name=wget state=present

- name: download file
  command: wget http://host:port/resource.zip -O ~/workers/resource.zip
```

当然，根据预定，我们会把变量放在`vars`目录的`main.yml`中，比如上面`apt`例子中的多个包的安装，我们会在`vars/main.yml`中定义变量

```yml
packages:
  	-  nginx
  	-  python
  	-  git  
```

然后在`tasks/main.yml`中引用：

```yml
- name: install packages
  apt: name={{ item }} state=present
  with_items: '{{ packages }}'
```

### 独立使用

通常我们会在一个Linux环境安装`Ansible`，这个环境专门做环境初始化的工作。如果你使用ubuntu环境，可以通过预编译的二进制包来安装：

```sh
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

当然，你也可以通过源码来安装：

```sh
$ git clone git://github.com/ansible/ansible.git --recursive
$ cd ./ansible
$ source ./hacking/env-setup
$ sudo make install #安装到系统路径，其他用户也可以使用
```

安装完成之后，你会得到`ansible`命令和`ansible-playbook`命令。

在执行`ansible`命令之前，我们需要定义`ansible`对应的远程机器列表配置。你需要在`/etc/ansible/hosts`文件中添加所有需要配置的机器IP地址或者域名（如果没有这个目录和文件，可以直接创建）。

文件内容就是每个地址一行的形式：

```
10.29.2.1
10.29.2.2
10.29.2.3
```

`ansible`命令可以用来向inventory执行shell命令，比如：

```sh
$ ansible all -m ping -u robot -b --become-user root
```

上面的命令会向主机`all`（`/etc/ansible/hosts`文件中指定的所有地址），执行一个`Ansible`模块`ping`（-m ping），使用用户`robot`（-u robot），并且以另一个用户身份`root`（-b --become-user root）。

如果没有设置过`ssh`的私钥，还需要指定`--ask-pass`选项，否则`ansible`会尝试用ssh登陆，然后失败。

```sh
$ ansible all -m ping -u robot -b --become-user root --ask-pass
```

除了使用内置模块之外，你开可以使用其他任何shell命令：

```sh
$ ansible all -a "/bin/echo hello" -u robot --ask-pass
```

`Ansible`会直接在远程机器上执行对应的shell命令。

### 在Vagrant中使用

`Vagrant`可以很容易的和`Ansible`集成在一起，只需要指定`config.vm.provision`为`ansible`即可：

```ruby
Vagrant.configure("2") do |config|
  config.vm.provision :ansible do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
```

当然，你可以定义多个虚拟机，然后并发的来自动化配置，比如像这样：

```ruby
(1..5).each do |machine_id|
  config.vm.define "machine#{machine_id}" do |machine|
    machine.vm.hostname = "machine#{machine_id}"
    machine.vm.network "private_network", ip: "192.168.2.#{20+machine_id}"

    if machine_id == N
      machine.vm.provision :ansible do |ansible|
        ansible.limit = "all"
        ansible.playbook = "playbook.yml"
      end
    end
  end
end
```

## Docker

### docker-machine

### docker-compose

