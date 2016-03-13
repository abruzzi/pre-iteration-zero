## 配置外化

应用开发中，开发者会将诸如数据库配置信息，NFS服务器的地址，消息队列的大小等等信息保存到配置文件中。比如Java Web中的`application.properties`文件，Rails中的`database.yml`等。这样我们可以在不同的环境中方便切换，只需要修改几行配置信息，应用的代码则完全不用修改。

比如下面是一个`Rails`应用的数据库配置文件：

```yml
development:
  adapter: sqlite3
  database: db/development.sqlite3
  pool: 5
  timeout: 5000

test:
  adapter: sqlite3
  database: db/test.sqlite3
  pool: 5
  timeout: 5000
```
