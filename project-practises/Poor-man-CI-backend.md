# 服务器端应用的持续交付

本文将使用一些免费的服务来为你的项目搭建`持续交付`平台，这些服务包括

-  持续集成环境
-  持续部署环境
-  服务端应用托管

以及一些可以用于本地开发使用的开源工具如：

-  [基于Node的构建monitor](https://github.com/abruzzi/node-build-monitor)
-  [Heroku的命令行工具](https://toolbelt.heroku.com/)
-  [Travis CI的命令行工具](https://github.com/travis-ci/travis.rb)

除此之外，我们在过程中编写的脚本还可以用以本地构建，如果你的团队中正好已经有CI工具/CD工具，将这些脚本集成进去也是一件非常容易的事情。

![heroku real time log](images/heroku-log-resized.png)

## 背景知识

### 软件的度量

传统的管理方法论，在软件开发这个领域来说基本上是不工作的。软件项目的不确定性使得人们畏惧，管理者希望通过一些数字，指标来让自己感到某种虚幻的“掌控感”。软件行数，测试覆盖率，代码故障率等数字的名声基本上已经很糟了，经常有人拿来讽刺那些追求虚幻掌控感的“领导”。

但是有一个数字，即使最顽固的“自由主义者”也会认为是有意义的，那就是周期时间（cycle time）。简而言之，就是一个需求从产生到最终上线所需要的时间。其中包括了需求分析，设计，编码，测试，部署，运维等活动，可能还会包含后续的监控。

其实不论是瀑布模型，还是迭代开发的方式，或者其他的方法论，周期时间的缩短都是至关重要的。而具体到周期内，单纯的开发时间变长或者测试时间变长都无关紧要。比如项目A的开发时间是测试时间的2倍，项目B则恰恰反过来，这并不能说A做的比B好，真正有意义的是A的周期时间是否比B更短。

单纯改善项目过程中的某一个阶段的时间，可能并不能达到预期的目的。局部优化并不一定会带来全局的优化。换言之，**通过某些策略来提高软件测试的效率未必能减少周期时间！**。

### 持续交付

传统情况下，企业要进行软件开发，从用户研究到产品上线，其中会花费数月，甚至数年（我的一位印度同事给我聊起过，他的上家公司做产品，从版本启动到版本上线需要整整两年时间！）。而且一旦软件需求发生变更，又有需要数月才能将变更发布上线。除了为变更提交代码外，还有很多额外的回归测试，发布计划，运维部门的进度等等。而市场机会千变万化，在特定的时间窗口中，企业的竞争者可能早已发布并占领了相当大的市场份额。

在软件工程领域，人们提出了持续交付（continuous delivery）的概念，它旨在减少周期时间，强调在任何时刻软件都处于可发布状态。采用这种实践，我们可以频繁，快速，安全的将需求的变化发布出来，交由真实世界的用户来使用，在为用户带来价值的同时，我们也可以快速，持续的得到反馈，并激励新的变化产生（新的商业创新，新的模式等）。

持续交付包含了自动化构建，自动化测试以及自动化部署等过程，持续改进开发流程中的问题，并促进开发人员，测试人员，运维人员之间的协作，团队可以在分钟级别将变更发布上线。

### 持续交付相关技术及实践

-  版本控制（配置管理）
-  持续集成CI
-  自动化测试
-  构建工具及构建脚本
-  部署流水线

团队通过版本控制来进行协作，所有的代码会在持续集成环境中编译，代码静态检查/分析，自动化测试（还可能产生报告等）。除此之外，CI还还需要有自动化验收测试，自动化回归测试等。

持续交付则更进一步，它将环境准备，持续集成，自动化部署等放在了一起。通过全自动（有些过程可以设置为手动，比如发布到产品环境）的方式，使得软件可以一键发布。如果上线后发现严重defect，还支持一键回滚的机制（其实就是将之前的一个稳定版本做一次发布，由于发布流程已经经过千锤百炼，所以发布本身就变得非常轻松，安全）

这篇文章中，我们会使用`git`+`github`作为版本控制工具，`travis-ci`作为持续集成环境，`gradle`作为构建工具，`Heroku`作为应用的部署环境。这些工具都是免费服务，如果你需要更高级的功能（比如更多的并发数，更大的数据库），则可以选择付费套餐。不过对于我们平时的大部分side project来说，免费服务已经足够。

## 实例

我在[《前后端分离了，然后呢？》](http://icodeit.org/2015/06/whats-next-after-separate-frontend-and-backend/)这篇文章中，提到了一个叫做`bookmarks`的应用，这个应用是一个前后端分离的非常彻底的应用。

我们这里会再次使用这个应用作为实例，并采用不同的两个免费服务（[travis-ci](https://travis-ci.org)和[snap-ci](https://snap-ci.com)）来完成`持续部署`环境的搭建。

### bookmarks服务器

`bookmarks-server`是一个基于`spring-boot`的纯粹的`API`，它可以被打包成一个`jar`包，然后通过命令行启动运行。在本文中，我们我们将会将这个server部署到[heroku](https://dashboard.heroku.com/)平台上。

首先需要定义一个`Procfile`，这个是我们应用的入口，`heroku`根据这个文件来明确以何种方式来启动我们的应用：

```
web: java -Dserver.port=$PORT -jar build/libs/bookmarks-server-0.1.0.jar --spring.profiles.active=staging
```

由于我们在本地使用的使用`mysql`，而`heroku`默认的是`postgres`数据库，因此需要在`application.yml`中额外配置

```yml
spring:
  profiles: staging

  datasource:
    driverClassName: org.postgresql.Driver
    url: ${JDBC_DATABASE_URL}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASS}

  jpa:
    database_platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: update
```

有了这些配置后，我们需要创建一个`heroku`应用：

```sh
$ heroku create
Created http://quiet-atoll-8237.herokuapp.com/ | git@heroku.com:quiet-atoll-8237.git
```

创建之后，我们可以在界面上对这个应用进行一些配置（当然，也可以通过命令行，具体参看`heroku help`）。为了支持数据库，需要为我们的应用添加一个`postgres`的AddOn。添加之后，`heroku`会为我们提供一个`postgres`的连接地址，格式大概是这样：

```
postgres://username:password@host:port/database
```

然后我们需要在`Heroku`的配置界面中配置一些环境变量：

![heroku env config](images/heroku-config-resized.png)

这样，当应用部署到`Heroku`上之后，我们的应用就可以读到这些配置了（注意`application.yml`中的环境变量`JDBC_DATABASE_URL`）。

#### 搭建持续集成环境

持续集成环境，这里我们选用最简单的`travis-ci`，它可以很容易的与`github`集成。

-  在项目X中定义一个`.travis.yml`的文件
-  将你的代码push到github上
-  绑定github帐号到`travis`
-  在`travis`中启用项目X

这个`.travis.yml`因项目而异，我们这里的项目是`spring-boot`，所以只需要指定`java`即可：

```yml
language: java
```

如果是`java`项目，并且项目中有`build.gradle`，`travis-ci`会自动执行`gradle check`任务。

#### 自动化部署

当CI运行成功之后，我们需要`travis-ci`帮我们将应用程序发布到`heroku`上，这时候需要做一些修改。最简单的方式是直接安装`travis-ci`的命令行工具到本地：

```sh
$ gem install travis -v 1.8.0 --no-rdoc --no-ri
```

然后通过`heroku`的`auth:token`命令获得`heroku`的token，在加密并写入`.travis.yml`：

```sh
$ heroku auth:token
00xxxxxxxxxxxxx55d11dbd0cxxxxxxxxxxfe067

$ travis encrypt 00xxxxxxxxxxxxx55d11dbd0cxxxxxxxxxxfe067 --add
```

当然可以合并为一条命令：

```sh
$ travis encrypt $(heroku auth:token) --add
```

将加密过的token存入`.travis.yml`文件。最后的结果大致如下：

```yml
language: java
deploy:
  provider: heroku
  api_key:
    secure: ...
  app: quiet-atoll-8237
```

注意此处的`app`，正是我们的App的名字。另外，还需要给`build.gradle`添加一个名叫`stage`的task，`travis`在deploy时需要这个`task`：

```groovy
task stage {
    dependsOn build
}
```

![travis deploy](images/travis-deploy-resized.png)

这样，我们只需要在本地的一个提交，一切都会自动化起来：

-  travis会执行`gradle check`
-  `gradle check`会编译并运行自动化测试
-  `travis`会部署应用到`heroku`上
-  `heroku`会自动重启服务

我们可以在本地进行简单的测试（注意此处我们的`staging`环境的URL）：

```sh
$ curl https://quiet-atoll-8237.herokuapp.com/api/feeds -s | jq .
[
  {
    "id": 1,
    "url": "http://icodeit.org/2016/01/how-to-summarize-privious-project/",
    "title": "如何持久化你的项目经历",
    "author": "icodit.org",
    "summary": "通常来说，下项目总是一件比较高兴的事（大部分团队还会一起吃个饭庆祝一下）。",
    "publishDate": "2016-01-07"
  },
  {
    "id": 2,
    "url": "http://icodeit.org/2015/11/get-started-with-reflux/",
    "title": "你为什么应该试一试Reflux？",
    "author": "icodit.org",
    "summary": "React在设计之初就只关注在View本身上，其余部分如数据的获取，事件处理等，全然不在考虑之内。",
    "publishDate": "2016-01-09"
  }
]
```

完整的[代码在这里](https://github.com/abruzzi/bookmarks-server)。

## 其他

### CI monitor

[node-build-monitor](https://github.com/abruzzi/node-build-monitor)是一个非常容易配置，使用的CI monitor，我们只需要进行简单地配置，就可以将`travis`的状态可视化出来

```json
{
  "monitor": {
    "interval": 2000,
    "numberOfBuilds": 12,
    "debug": true
  },
  "services": [
    {
      "name": "Travis",
      "configuration": {
        "slug": "abruzzi/bookmarks-server"
      }
    }
  ]
}
```

不过这个工具会在有网络异常时自动终止，我们可以通过一个简单的脚本来在它终止时自动重启：

```sh
#!/bin/bash

until node app/app.js
do
    echo "restarting..."
done
```

![CI Monitor](images/ci-monitor-resized.png)

## 小结

通过`travis`和`heroku`这样的免费服务，我们就可以轻松的将自己的项目做到持续集成+持续交付。我们后端的服务相对来说是比较容易的，但是涉及到一个前后端分离的架构，如何做到静态内容的托管，打包，部署，并和后端API集成起来，我会在下一篇文章中详细解释。
