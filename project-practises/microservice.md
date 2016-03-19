
### structure 

```sh
$ mkdir -p microservice-demo
$ microservice-demo 
$ gradle init --type=java-library
```

```sh
➜  microservice-demo tree
.
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
└── src
    ├── main
    │   └── java
    │       └── Library.java
    └── test
        └── java
            └── LibraryTest.java

7 directories, 8 files
```

### 引入 spring-boot

```groovy
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:1.3.3.RELEASE")
    }
}

apply plugin: 'java'
apply plugin: 'idea'
apply plugin: 'spring-boot'

repositories {
    jcenter()
}

dependencies {
    compile 'org.slf4j:slf4j-api:1.7.13'
    compile "org.springframework.boot:spring-boot-starter-web:1.3.3.RELEASE"
    testCompile 'junit:junit:4.12'
}
```

```sh
$ gradle idea
$ open microservice-demo.ipr
```

### 版本控制

```sh
$ git init .
$ touch .gitignore
```

```
.gradle/
build/
microservice-demo.*
out/
```

### 启动

创建应用程序入口

```java
package com.thoughtworks.microservice.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

