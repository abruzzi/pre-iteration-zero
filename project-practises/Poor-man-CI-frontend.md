# 客户端程序的的持续交付

[上篇文章](http://icodeit.org/2016/01/a-poor-mans-cd-part1/)介绍了如何使用一些免费的服务来实现服务器端API的持续集成、持续交付环境的搭建。有了服务端，自然需要有消费者，在本文中我们将使用另外一个工具来实现纯前端的站点的部署。

其中包括：

-  持续集成（单元测试，集成测试等）
-  持续部署/持续交付
-  静态站点托管

除此之外，我们还会涉及到：

-  [自动化UI测试site_prism](https://github.com/natritmeyer/site_prism)
-  静态站点的发布脚本
-  aws的[命令行工具](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)

我们的应用最后看起来是这样子的。

![bookmarks app](/images/2016/01/bookmarks-app-resized.png)

## 技术选型

我们在本文中，将采取另外一套免费服务来完成环境的搭建

-  [ThoughtWorks](http://www.thoughtworks.com/)出品的[Snap CI](https://snap-ci.com/)作为持续集成/持续交付环境
-  [AWS的S3](https://console.aws.amazon.com/s3/home?region=us-west-2)作为应用发布的地方

`Snap CI`是一个非常易于使用的持续交付环境，由于很多关于持续集成，持续交付的概念和实践都跟`ThoughtWorks`有关，所以这个产品对于构建，流水线，部署等等的支持也都做的非常好。

`S3`是亚马逊的云存储平台，我们可以将静态资源完全托管在其上。`S3`的另一个好处是它可以将你的文件变成一个Web Site，比如你的目录中有`index.html`，这个文件就可以作为你的站点首页被其他人访问。这个对于我们这个前后端分离项目来说非常有用，我们的`css`，`js`，`font`文件，还有入口文件`index.html`都可以托管于其上。

## 实例

在本文的例子中，我们将定义3个`stage`。`Snap CI`将一次发布分为若干个`stage`，每个`stage`只做一件事情，如果一个`stage`失败了，后边的就不会接着执行。

我们的3个`stage`分别为：

1.  单元测试
2.  集成测试
3.  部署

### 准备工作

`bookmarks-frontend`是一个纯前端的应用，它会消费后端提供的API，但是其实它并不知道（也不应该知道）后端的API部署在什么地方：

```js

$(function() {
	var feeds = $.get(config.backend+'/api/feeds');
	var favorite = $.get(config.backend+'/api/fav-feeds/1');

	$.when(feeds, favorite).then(function(feeds, favorite) {
		//...
	});
});
```

由于我们在本地开发时，需要`backend`指向本地的服务器，而发布之后，则希望它指向[上一篇文章](http://icodeit.org/2016/01/a-poor-mans-cd-part1/)中提到的服务器，因此我们需要编写一点构建脚本来完成这件事儿：

```js
var backend = 'http://quiet-atoll-8237.herokuapp.com';

gulp.task('prepareConfig', function() {
    gulp.src(['assets/templates/config.js'])
    .pipe(replace(/#backend#/g, 'http://localhost:8100'))
    .pipe(gulp.dest('assets/script/'));
});

gulp.task('prepareRelease', function() {
    gulp.src(['assets/templates/config.js'])
    .pipe(replace(/#backend#/g, backend))
    .pipe(gulp.dest('assets/script/'));
});
```

我们定义了两个`gulp`的task，本地开发时，使用`prepareConfig`，要发布时，使用`prepareRelease`，然后定义一个简单的模板文件`config.js`：

```js
module.exports = {
	backend: '#backend#'
}
```

然后可以很简单的包装一下，方便本地开发和发布：

```js
gulp.task('dev', ['prepareConfig', 'browserify', 'concatcss']);
gulp.task('build', ['prepareConfig', 'script', 'css']);
gulp.task('release', ['prepareRelease', 'script', 'css']);
```

这样，我们在本地开发时，只需要简单的执行：

```sh
$ gulp
```

即可。而在发布阶段，只需要执行：

```sh
$ gulp release
```

### 单元测试

我们在`Snap CI`上将`github`上的代码库关联起来，然后添加一个名叫`unit-test`的`stage`，指定这个`stage`对应的命令为：

```sh
npm install
gulp
```

![Snap CI unit](/images/2016/01/snap-ci-unit-resized.png)

这样，每当我们有新的提交之后，`Snap CI`都会拿到新代码，并执行上述命令，如果执行成功，则本地构建成功。

### 集成测试

由于采取的是**前后端分离**的策略，我们的应用可以完全独立与后端进行开发，因此我们设置了一个`fake server`，具体细节可以参考[我之前的博客](http://icodeit.org/2015/06/whats-next-after-separate-frontend-and-backend/)，也可以看源码。不过这里我们要为集成测试编写一个脚本，并在`Snap CI`上执行。

```sh
#!/bin/bash

export PORT=8100
bundle install

# launch the application
echo "launch the application"
ruby app.rb 2>&1 &
PID=$!

# wait for it to start up
sleep 3

# run the rspec tests and record the status
rspec
RES=$?

# terminate after rspec
echo "terminate the application"
kill -9 $PID

# now we know whether the rspec success or not
exit $RES
```

这个脚本中，首先安装所有的`gems`，然后启动`fake server`并将这个server放置在后台运行，然后执行`rspec`。当`rspec`测试执行完成之后，我们终止服务进行，然后返回结果状态码。

这里使用了`capybara`和`poltergeist`来做UI测试，`capybara`会驱动`phantomjs`来在内存中运行浏览器，并执行定义好的`UI`测试，比如此处，我们的UI测试：

```rb
require 'spec_helper'

describe 'Feeds List Page' do
	let(:list_page) {FeedListPage.new}

	before do
		list_page.load
	end

	it 'user can see a banner and some feeds' do
		expect(list_page).to have_banner
		expect(list_page).to have_feeds
	end
	
	##...
end
```

![Snap CI logs](/images/2016/01/snap-ci-it-resized.png)

### 部署

首先需要在`S3`上创建一个`bucket`，命名为`bookmarks-frontend`。然后为其设置`static website hosting`，这时候`AWS`会assign一个新的域名给你，比如`http://bookmarks-frontend.s3-website-us-west-2.amazonaws.com/`。

然后你需要将这个`bucket`设置成`public`，这样其他人才可以访问你的`bucket`。

![AWS S3](/images/2016/01/aws-s3-public-resized.png)

有了这个之后，我们来编写一个小脚本，这个脚本可以将本地的文件上传至S3。

```sh
#!/bin/bash

# install gulp and its dependencies
npm install

# package stuff, and point the server to the right place
gulp release

# upload the whold folder
aws s3 cp public/ s3://bookmarks-frontend \
	--recursive \
	--region us-west-2 \
	--acl public-read
```

`aws`命令是`aws command line`提供的，另外我们需要在环境变量中设置AWS提供给你的token：

```sh
AWS_ACCESS_KEY_ID=xxxxxxxxxx
AWS_SECRET_ACCESS_KEY=xxxxxxxxxx
```

然后我们就可以将本地的`public`目录递归的上传到S3的对应目录了！

![snap ci pipeline](/images/2016/01/snap-ci-pipeline-resized.png)

完整的代码可以在[此处下载](https://github.com/abruzzi/bookmarks-frontend)。

## 总结

我们前端的持续交付也介绍完了。现在前后端应用完全独立，发布也互不影响。不论是服务器端新增加了API，还是添加了新数据，客户端的发布都不受影响；同样，修改样式，添加新的`JavaScript`也完全不会影响后端。更重要的是，所有的发布都是一键式的，开发者只需要一个`git push`就可以享受这些免费服务提供的自动构建，自动化测试以及自动部署的功能。
