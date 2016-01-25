#[dockerfile] postgresql with tsearchextras for zulip

这个仓库fork自 [sameersbn/postgresql](https://github.com/sameersbn/docker-postgresql) 并且为了配合 zulip 一起使用还安装了 [全文索引](https://github.com/zbenjamin/tsearch_extras)（tsearch-extras）的支持。

___

# 目录

- [好雨云部署和使用]()
  - [一键部署]()
  - [环境变量]()
  - [启动与停止]()
  - [数据安全]()
  - [更新](#upgrading)

- [私有docker环境部署和使用]()
  - [安装部署](#installation)
  - [快速启动](#quick-start)
  - [持久化](#persistence)
  - [启动时创建用户和数据库](#creating-user-and-database-at-launch)
  - [创建快照或从数据库](#creating-a-snapshot-or-slave-database)
  - [主机 UID / GID 映射](#host-uid--gid-mapping)
  - [Shell 访问](#shell-access)
  - [更新](#upgrading)



- [更新日志](Changelog.md)
- [项目参与和讨论](#项目参与和讨论)


# 好雨云部署和使用

## 一键部署
点击下面的 按钮会跳转到 好雨应用市场的应用首页中，可以通过一键部署按钮安装

<a href="http://app.goodrain.com/app/28/" target="_blank"><img src="http://www.goodrain.com/images/deploy/button_120201.png"></img></a>

## 环境变量
| 变量名| 变量默认值| 说明|
|-----|---------|-----|
|POSTGRESQL_HOST| 127.0.0.1| 连接ip地址|
|POSTGRESQL_POST| 5432 | 连接端口|
|POSTGRESQL_USER| admin | 连接用户名|
|POSTGRESQL_USER| **随机** | 连接密码|
| POSTGRESQL_NAME|test|初始创建数据库|

## 启动与停止
在平台上控制 `PostgreSQL` 服务非常的方便，只需要点击 “启动” 按钮进行服务的启动，点击“关闭” 将服务停止即可。

## 数据安全
平台采用高速SSD固态硬盘来存储数据，并且会有自动备份机制将数据存3份，用户不必担心数据的丢失。

## 更新
当平台的应用版本检测到有更新时，会出现 如下的图标，可以直接点击更新来更新自己的服务。

[更新图标 - 暂缺]()

`注意：` 请认真查看更新日志，随意更新当前正常运行的应用有可能造成意外的问题。


# 私有 docker 环境部署与使用

## 拉取镜像
```bash
docker pull goodrain.io/postgresql-tsearchextras:latest
```

## 构建镜像

```bash
git clone https://github.com/goodrain-apps/docker-postgresql-tsearchextras.git
cd docker-postgresql-tsearchextras
docker build -t postgresql-tsearchextras
```

## 数据持久化

持久化目录需要挂载到容器的 `/var/lib/postgresql` 目录。

如果启用了 SELinux 需要修改目录的安全属性

```bash
mkdir -p /opt/postgresql/data
sudo chcon -Rt svirt_sandbox_file_t /opt/postgresql/data
```

运行的命令类似如下：

```bash
docker run --name postgresql -d \
  -v /opt/postgresql/data:/var/lib/postgresql goodrain.io/postgresql-tsearchextras:latest
```

外挂目录主要是为了确保容器在停止后数据不会丢失

## 启动时创建用户和数据库

该镜像允许在首次启动数据库时创建用户和数据库。

通过在启动命令中指定 `DB_USER` 和 `DB_PASS` 变量的值来创建用户和数据库。下面示例命令创建了一个 *dbuser* 用户，并且设置了密码为 *dbpass*。

```bash
docker run --name postgresql -d \
  -e 'DB_USER=dbuser' -e 'DB_PASS=dbpass' \
  goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

**注意**
- 如果没有指定密码，则用户不会被创建
- 如果用户已经存在，则所有操作都不会执行

当然你也可以在启动命令中指定 `DB_NAME` 变量来初始化创建一个数据库。

```bash
docker run --name postgresql -d \
  -e 'DB_NAME=dbname' goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

你还可以一次性创建多个数据库，通过设置 `DB_NAME` 变量，并且利用逗号来分开多个数据库名。 下面的命令创建了2个数据库，名字为  *dbname1* 和 *dbname2* 

```bash
docker run --name postgresql -d \
  -e 'DB_NAME=dbname1,dbname2' \
  goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

如果在初始化创建数据库的时候同时指定了 `DB_USER` 和 `DB_PASS` 变量，那么这个用户会对这个初始化的数据库有完全的访问权限。

例如：

```bash
docker run --name postgresql -d \
  -e 'DB_USER=dbuser' -e 'DB_PASS=dbpass' -e 'DB_NAME=dbname' \
  goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

它会创建 *dbuser* 用户，并且设置密码为 *dbpass* 。 还会创建一个名为 *dbname* 的数据库，*dbuser* 用户拥有*dbname* 数据库完全的访问权限。

`DB_LOCALE` 环境变量用来配置区域（locale），默认值是  C 。

`PSQL_TRUST_LOCALNET` 环境变量 用来配置 postgres 允许来自本地网络的链接。目的就是让其它容器以link的形式链接postgres时可以通过无验证的形式链接数据库。启用该选项需要将 `PSQL_TRUST_LOCALNET` 设置为 `true`。

如下：

```bash
docker run --name postgresql -d \
  -e 'PSQL_TRUST_LOCALNET=true' \
  goodrain.io//docker-zulip-postgresql-tsearchextras:latest
```

它的效果类似在 `pg_hba.conf` 文件中添加了如下的内容:

```
host    all             all             samenet                 trust
```

## 创建快照或者从库

You may use the `PSQL_MODE` variable along with `REPLICATION_HOST`, `REPLICATION_PORT`, `REPLICATION_USER` and `REPLICATION_PASS` to create a snapshot of an existing database and enable stream replication.

Your master database must support replication or super-user access for the credentials you specify. The `PSQL_MODE` variable should be set to `master`, for replication on your master node and `slave` or `snapshot` respectively for streaming replication or a point-in-time snapshot of a running instance.

Create a master instance

```bash
docker run --name='psql-master' -it --rm \
  -e 'PSQL_MODE=master' -e 'PSQL_TRUST_LOCALNET=true' \
  -e 'REPLICATION_USER=replicator' -e 'REPLICATION_PASS=replicatorpass' \
  -e 'DB_NAME=dbname' -e 'DB_USER=dbuser' -e 'DB_PASS=dbpass' \
  quay.io/galexrt/docker-zulip-postgresql-tsearchextras:latest
```

Create a streaming replication instance

```bash
docker run --name='psql-slave' -it --rm  \
  --link psql-master:psql-master  \
  -e 'PSQL_MODE=slave' -e 'PSQL_TRUST_LOCALNET=true' \
  -e 'REPLICATION_HOST=psql-master' -e 'REPLICATION_PORT=5432' \
  -e 'REPLICATION_USER=replicator' -e 'REPLICATION_PASS=replicatorpass' \
  quay.io/galexrt/docker-zulip-postgresql-tsearchextras:latest
```

# Enable Unaccent (Search plain text with accent)

Unaccent is a text search dictionary that removes accents (diacritic signs) from lexemes. It's a filtering dictionary, which means its output is always passed to the next dictionary (if any), unlike the normal behavior of dictionaries. This allows accent-insensitive processing for full text search.

By default unaccent is configure to `false`

```bash
docker run --name postgresql -d \
  -e 'DB_UNACCENT=true' \
  quay.io/galexrt/docker-zulip-postgresql-tsearchextras:latest
```

# Host UID / GID Mapping

Per default the container is configured to run postgres as user and group `postgres` with some unknown `uid` and `gid`. The host possibly uses these ids for different purposes leading to unfavorable effects. From the host it appears as if the mounted data volumes are owned by the host's user/group `[whatever id postgres has in the image]`.

Also the container processes seem to be executed as the host's user/group `[whatever id postgres has in the image]`. The container can be configured to map the `uid` and `gid` of `postgres` to different ids on host by passing the environment variables `USERMAP_UID` and `USERMAP_GID`. The following command maps the ids to user and group `postgres` on the host.

```bash
docker run --name=postgresql -it --rm [options] \
  --env="USERMAP_UID=$(id -u postgres)" --env="USERMAP_GID=$(id -g postgres)" \
  quay.io/galexrt/docker-zulip-postgresql-tsearchextras:latest
```


# Upgrading

To upgrade to newer releases, simply follow this 3 step upgrade procedure.

- **Step 1**: Stop the currently running image

```bash
docker stop postgresql
```

- **Step 2**: Update the docker image.

```bash
docker pull quay.io/galexrt/docker-zulip-postgresql-tsearchextras:latest
```

- **Step 3**: Start the image

```bash
docker run --name postgresql -d [OPTIONS] quay.io/galexrt/docker-zulip-postgresql-tsearchextras:latest
```

# Shell Access

For debugging and maintenance purposes you may want access the containers shell. If you are using docker version `1.3.0` or higher you can access a running containers shell using `docker exec` command.

```bash
docker exec -it postgresql bash
```

If you are using an older version of docker, you can use the [nsenter](http://man7.org/linux/man-pages/man1/nsenter.1.html) linux tool (part of the util-linux package) to access the container shell.

Some linux distros (e.g. ubuntu) use older versions of the util-linux which do not include the `nsenter` tool. To get around this @jpetazzo has created a nice docker image that allows you to install the `nsenter` utility and a helper script named `docker-enter` on these distros.

To install `nsenter` execute the following command on your host,

```bash
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
```

Now you can access the container shell using the command

```bash
sudo docker-enter postgresql
```

For more information refer https://github.com/jpetazzo/nsenter

# 项目参与和讨论

如果你觉得这个镜像很有用可以通过如下方式参与和改进项目：

- 如果有新特性或者bug修复，请发送 一个 Pull 请求，我们会及时反馈。
- 新用户可以查看 [PostgreSQL-TSearchExtras](https://github.com/goodrain-apps/docker-postgresql-tsearchextras/issues) 查看介绍文档和参与讨论
