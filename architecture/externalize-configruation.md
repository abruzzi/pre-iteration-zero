## 配置外化

应用开发中，开发者会将诸如数据库配置信息，NFS服务器的地址，消息队列的大小等等信息保存到配置文件中。比如Java Web中的`application.properties`文件，Rails中的`database.yml`等。这样我们可以在不同的环境中方便切换，只需要修改几行配置信息，应用的代码则完全不用修改。

下面是一个`Rails`应用的数据库配置文件：

```yml
test:
  adapter: sqlite3
  database: db/test.sqlite3
  pool: 5
  timeout: 5000

production:
  adapter: mysql2
  database: test-db
  pool: 5
  username: root
  password: s3cr3t
  socket: /tmp/mysql.sock
```

这个`yml`定义了`test`环境，数据库使用`sqlite3`，数据库文件为`db/development.sqlite3`。而在`production`环境，数据库采用`mysql`。

在运行时，只需要指定环境变量，即可切换数据库：

```sh
$ RAILS_ENV=test rails server
```

### 实例

我们以一个简单的Java应用来演示如何将应用程序的配置信息。在这个应用程序中，我们需要数据库配置可以在运行是改变，而不是将配置内置在应用程序中。

在实际场景中，应用程序可能在部署时才知道要连接的数据库地址是什么，而且数据库的名称，数据库连接池的大小等信息都可能因环境而变化。

在这个应用程序中，我们将在应用程序中连接`mongodb`数据库，从数据库中读取一个集合的内容，然后打印出整个集合。我们会使用`soring`和`spring-data-mongodb`来完成简化应用的编写。

我们使用`gradle`的`init`命令来生成一个典型的Java应用程序：

```sh
$ mkdir -p spring-mongo-demo
$ cd spring-mongo-demo
$ gradle init --type=java-library
```

另外我们为应用程序添加依赖：

```gradle
apply plugin: 'java'
apply plugin: 'idea'

buildscript {
    repositories {
        jcenter()
    }
}

repositories {
    jcenter()
}

dependencies {
    compile 'org.slf4j:slf4j-api:1.7.13'
    compile 'org.springframework:spring-context:4.2.4.RELEASE'
    compile 'org.springframework.data:spring-data-mongodb:1.8.4.RELEASE'
}
```

这样，只需要执行

```sh
$ gradle build
```

就可以下载所有依赖库了。

#### 配置文件

我们首先来为应用程序创建这样的包接口：

```
src
├── main
│   ├── java
│   │   └── com
│   │       └── thoughtworks
│   │           └── mongo
│   │               ├── config
│   │               ├── model
│   │               └── repo
│   └── resources
└── test
    └── java
```

要做到在运行时可改变配置，我们首先需要保证配置和代码分离。这个步骤很容易实现，只需要将配置定义在配置文件中，然后应用在启动时读取该配置（application.properties）即可：

```
mongo.host=localhost
mongo.database=test
```

根据惯例，`application.properties`会放在`resources`目录下。

在我们这个应用中，数据库中又一个`people`的集合，对应的我们需要又一个`Person`的实体，和一个用来访问数据库集合的`PersonRepository`。

```java
package com.thoughtworks.mongo.model;

import org.springframework.data.annotation.Id;

public class Person {
    @Id
    private String id;

    private String name;
    private int age;

    public Person() {
    }

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    //getter & setter

    @Override
    public String toString() {
        return "{name="+name+", age="+age+"}";
    }
}
```

`spring-data-mongo`提供了一个`MongoRepository`接口，我们的应用只需要继承这个接口，就可以免费获得很多有用的数据库访问功能。

```java
package com.thoughtworks.mongo.repo;

import com.thoughtworks.mongo.model.Person;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface PersonRepository extends MongoRepository<Person, String> {
}
```

借助`spring`强大的注入器，我们很容易在应用中使用这个接口，而不用关心背后它是如何被实例化的：

```java
public class Application {

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        PersonRepository personRepository = context.getBean(PersonRepository.class);

        List<Person> all = personRepository.findAll();
        System.err.println(all);
    }

}
```

我们首先创建一个基于注解的Context，具体的应用配置我们放在了`AppConfig`类中，有了这个Context，我们可以从中获取`PersonRepository`的实例，并使用它的`findAll`方法来获取所有的人员列表。

对于我们的应用来说，所有的配置信息都放在`AppConfig`中。我们来看看这个类：

```java
package com.thoughtworks.mongo.config;

import com.mongodb.MongoClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.*;
import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;
import org.springframework.core.env.Environment;
import org.springframework.data.mongodb.MongoDbFactory;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.SimpleMongoDbFactory;

@Configuration
@ComponentScan(value = "com.thoughtworks.mongo.*")
@PropertySource(value = "classpath:application.properties")
public class AppConfig {
    @Autowired
    private Environment environment;

    @Bean
    public MongoDbFactory mongoDbFactory() throws Exception {
        String mongoHost = environment.getProperty("mongo.host");
        String mongoDatabase = environment.getProperty("mongo.database");

        return new SimpleMongoDbFactory(new MongoClient(mongoHost), mongoDatabase);
    }

    @Bean
    public MongoTemplate mongoTemplate() throws Exception {
        return new MongoTemplate(mongoDbFactory());
    }

    @Bean
    public static PropertySourcesPlaceholderConfigurer propertyConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }

}
```

这个类中，我们使用了`PropertySource`这个注解，并指定了配置文件应该从`classpath`中的`application.properties`中获取。

当我们执行应用时，配置文件会生效，一起正常！下来我们来构建一个可以独立发布的`jar`包，这样任何人都可以直接使用这个jar包，而不需要自己重新下载依赖，重新构建等，所以我们为`build.gradle`添加几条简单的命令：

```
jar {
    baseName = rootProject.name
    version =  '0.1.0'

    from {
        configurations.compile.collect { it.isDirectory() ? it : zipTree(it) }
    }

    manifest {
        attributes("Main-Class": "com.thoughtworks.mongo.Application")
    }
}
```

这样构建出来的包就会包含所有依赖，我们还显式的指定了该jar包里的`Main-Class`是`com.thoughtworks.mongo.Application`。

```sh
$ gradle build
$ java -jar build/libs/spring-mongo-demo-0.1.0.jar
```

会得到

```
[{name=Juntao, age=30}, {name=Abruzzi, age=28}]
```

很好，我们成功的访问了数据库，并打印出了集合中的所有元素，而且这个应用程序是可以独立发布的了。

#### 系统配置文件

如果仔细想想使用场景，你会发现如果数据库连接发生变化了，我们修改`application.properties`文件，而该文件是打包在jar包内的！

这意味这应用程序和它的环境并没有分离，简单来说，我们需要提供机制来让外部的配置可以覆盖包内的配置。

最简单的方式下，我们创建一个新的`properties`文件，并让应用程序最后使用这个配置，这样就可以达到覆盖的目的了。当然，如果外部没有提供`properties`文件，应用还可以使用内部的配置提供功能。

`spring`在版本4之后，提供了比`PropertySource`更强大的注解`PropertySources`，它支持定义多个配置源，并形成一个链表，这样后边的元素就可以覆盖前面的元素了。


```java
@Configuration
@ComponentScan(value = "com.thoughtworks.mongo.*")
@PropertySources({
        @PropertySource(value = "classpath:application.properties"),
        @PropertySource(value = "file:/etc/spring-mongo/application.properties", ignoreResourceNotFound=true)
})
public class AppConfig {
	//...
}
```

我们定义了两个配置源，一个是`application.properties`，另一个是绝对路径下`/etc/spring-demo/`下的同名文件。`ignoreResourceNotFound=true`保证如果找不到该配置，也不会报错。

这样，如果我们需要新的数据库连接/配置，只需要在`/etc/spring-demo`下创建同名文件，并设置新的值即可。

这样做的好处是，应用程序无需做任何修改，配置信息外化到了环境中，部署应用程序的环境来确定应用具体如何与外部依赖交互。

#### 环境变量

另一种常用的方式是使用环境变量，这种方式下，我们只需要修改启动脚本，就可以将信息传递给应用程序，这中方式在`UNIX`世界已经存在多年。

`spring`提供了`Environment`对象，该对象提供了对环境的封装，其中包含了环境应用了那些`profile`的，系统配置，操作系统环境变量，以及所有的配置源`propertySources`：

```sh
{activeProfiles=[], defaultProfiles=[default], propertySources=[systemProperties,systemEnvironment,URL [file:/etc/spring-mongo/application.properties],class path resource [application.properties]]}
```

这样，我们的代码中使用的

```java
@Autowired
private Environment environment;

@Bean
public MongoDbFactory mongoDbFactory() throws Exception {
    String mongoHost = environment.getProperty("mongo.host");
    String mongoDatabase = environment.getProperty("mongo.database");

    return new SimpleMongoDbFactory(new MongoClient(mongoHost), mongoDatabase);
}
```

都自然的可以从Java环境变量中获得配置信息：

```sh
java -Dmongo.host=10.29.10.212 -Dmongo.database=prod-db -jar build/libs/spring-mongo-demo-0.1.0.jar
```

这样，配置信息就通过外部传入。这里仅仅是一个很简单的例子，项目中的配置信息会非常多，而且可能会有多种方式混用的场景：部分信息放在系统的环境变量中，如`/etc/profile`或者`.bashrc`中，另外一部分信息则存储在应用特定的properties中。

总体而言，这些配置信息都需要和应用程序本身分离。这和持续交付的实践其实也是相关的，在持续交付中，我们只会生成一个二进制包。这个二进制包在部署流水线上一直使用，在回归测试、性能测试等任务中，这个包会被部署在不同的环境中，并被测试有效性。这样我们才对应用程序自身有更多的信心。