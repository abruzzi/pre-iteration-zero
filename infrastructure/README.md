# 基础设施

## 传统项目

2012年之前，我在一家传统的软件公司工作。每当我们要新启动一个项目时，项目经费中总会有一项“硬件采购”。其中会描述我们需要多少台服务器（开发环境，测试环境，UAT，生产等），还会描述每个服务器的规格（网卡配置，内存，硬盘等）。当机器购买到之后，我们需要将其部署到机房，接上网线，然后由专门的运维工程师为其配置操作系统，网络，甚至数据库系统。

在开发过程中，如果某个环境由于某些原因崩溃了（比如加班到深夜的程序员睡眼惺忪的执行了`rm -rf /`），就只好等运维工程师重新安装。有时候，我们会想想何不把系统做成一个镜像呢？就像虚拟机那样，如果使用虚拟机，每当团队里有人要使用Linux时，我们可以拷贝一份镜像给他。<del>但是对于如何将物理机器上的操作系统变成镜像好像也没有成熟的方案，当大家都习惯了这种工作节奏之后，也就不会有人来多问（毕竟多一事儿不如少一事儿）。</del>

有了环境之后，开发人员开始写代码，并定期打包，然后部署到各个环境（有时候还要为64位机器和32位机器分别构建不同的软件包）。运维部门有些聪明的家伙会编写一些小脚本，通过`ssh`到不同机器来上传应用文件，重启`J2EE`容器等，这样他们只需要维护一组IP地址就可以了。

但是现实世界是复杂的，比如开发人员在`J2EE`应用中使用了`properties`文件来指定数据库的url和文件服务器的地址，这些配置信息在测试环境，staging环境都是不一样的，这样导致的一个问题是：要么我们为不同的环境打不同的包，要么将配置文件放在`war`包外边，然后让运维工程师在部署的时候，根据实际情况来修改这个`properties`，之后重启容器即可。

当环境慢慢变得多起来的时候，你就可以想像运维工程师的脸色有多难看。程序员最痛恨的就是重复劳动了，运维工程师也一样，谁会喜欢每周三晚上都工作到12点来部署另一个部门的人开发的`垃圾`程序呢？
