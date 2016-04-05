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

## Docker

### docker-machine

### docker-compose

