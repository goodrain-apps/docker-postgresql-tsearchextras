#[dockerfile] postgresql with tsearchextras for zulip

这个仓库fork自 [sameersbn/postgresql](https://github.com/sameersbn/docker-postgresql) 并且为了配合 zulip 一起使用还安装了 [全文索引](https://github.com/zbenjamin/tsearch_extras)（tsearch-extras）的支持。

___

# 目录

- [部署到好雨云](#部署到好雨云)
  - [一键部署](#一键部署)
  - [环境变量](#环境变量)
  - [启动与停止](#启动与停止)
  - [数据安全](#数据安全)
  - [更新](#更新)

- [部署到本地](#部署到本地)
  - [拉取镜像](#拉取镜像)
  - [构建镜像](#构建镜像)
  - [数据持久化](#数据持久化)
  - [初始化用户和库](#初始化用户和库)
  - [创建快照或者从库](#创建快照或者从库)
  - [启用 Unaccent](#启用 Unaccent)
  - [主机 UID / GID 的映射](#主机 UID / GID 的映射)
  - [Shell 访问](#Shell 访问)
  - [更新](#更新)
- [项目参与和讨论](#项目参与和讨论)
- [更新日志](Changelog.md)


# 部署到好雨云

## 一键部署
点击下面的 按钮会跳转到 好雨应用市场的应用首页中，可以通过一键部署按钮安装

<a href="http://app.goodrain.com/app/28/" target="_blank"><img src="http://www.goodrain.com/images/deploy/button_120201.png"></img></a>

**注意：**
这个版本的`PostgreSQL` 是专门针对 `Zulip` 的制定版本，安装`Zulip`时会自动安装，如果用户想独立安装这个特殊版本也是可以的。

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


# 部署到本地

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

## 初始化用户和库

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

你可以利用 `PSQL_MODE` 变量，配合 `REPLICATION_HOST`, `REPLICATION_PORT`, `REPLICATION_USER` 和 `REPLICATION_PASS` 变量来创建一个现有数据库的快照并且启动流复制。

你的主库必须支持复制，并且超级用户有相关的操作权限。 主库 将 `PSQL_MODE` 变量设置为 `master`，再运行一个示例，将`PSQL_MODE` 变量设置为 `slave` 或者 `snapshot` 。

创建一个主库实例：

```bash
docker run --name='psql-master' -it --rm \
  -e 'PSQL_MODE=master' -e 'PSQL_TRUST_LOCALNET=true' \
  -e 'REPLICATION_USER=replicator' -e 'REPLICATION_PASS=replicatorpass' \
  -e 'DB_NAME=dbname' -e 'DB_USER=dbuser' -e 'DB_PASS=dbpass' \
  goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

创建一个复制流实例：

```bash
docker run --name='psql-slave' -it --rm  \
  --link psql-master:psql-master  \
  -e 'PSQL_MODE=slave' -e 'PSQL_TRUST_LOCALNET=true' \
  -e 'REPLICATION_HOST=psql-master' -e 'REPLICATION_PORT=5432' \
  -e 'REPLICATION_USER=replicator' -e 'REPLICATION_PASS=replicatorpass' \
  goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

## 启用 Unaccent (文本搜索字典去掉重音)

unaccent 是一个文本搜索字典，它从词汇中去掉重音符号（变音标志符号）。 这是一个过滤词典，这意味着它的输出总是传递给下一个字典（如果存在的话），而不像常规行为的字典。 这允许对全文搜索进行重音不敏感的处理。

默认 unaccent 被设置为 `false`

```bash
docker run --name postgresql -d \
  -e 'DB_UNACCENT=true' \
  goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

# 主机 UID / GID 的映射

默认情况下容器在运行postgres时会使用 `postgres` 用户和组，它们的 `uid` 和 `gid` 都是不可控的。但如果在做持久化数据卷挂载时主机看到的数据属主可能会觉得莫名其妙。因此为了保证统一，可以在运行镜像时指定主机的某个账号（可以是postgres账号）的`uid`和`gid`来映射到容器中的`postgres` 账号中。通过配置 `USERMAP_UID` 和 `USERMAP_GID` 变量来实现，如下：

```bash
docker run --name=postgresql -it --rm [options] \
  --env="USERMAP_UID=$(id -u postgres)" --env="USERMAP_GID=$(id -g postgres)" \
 goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

## Shell 访问

为了方便的调试和维护，有些时候需要进入到容器内部进行操作。如果你使用的docker 版本是基于 `1.3.0` 或者更高版本，可以执行  `docker exec` 命令，如下：

```bash
docker exec -it postgresql bash
```

如果你使用的是比较旧的版本，你需要使用 [nsenter](http://man7.org/linux/man-pages/man1/nsenter.1.html) 系统工具 (安装 util-linux 包) 来访问容器的shell。

部分linux发行版本 (如：ubuntu) 还在使用旧版本的 util-linux 包，它是不包含 `nsenter` 工具的。但这也难不倒咱们伟大的程序员， @jpetazzo 大侠创建了一个可以安装 `nsenter`的镜像。利用这个镜像可以安装 `nsenter` 工具。

为了安装 `nsenter` 在你的主机中执行如下命令：

```bash
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
```

完成后，可以通过下面的命令进入到你的容器命令行中

```bash
sudo docker-enter postgresql
```

更多的信息参见 https://github.com/jpetazzo/nsenter

## 更新

通过下面3步操作更新到最新的发布版本。

- **第1步**: 停止当前运行的容器

```bash
docker stop postgresql
```

- **Step 2**: 更新docker镜像

```bash
docker pull goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

- **Step 3**: 启动新镜像

```bash
docker run --name postgresql -d [OPTIONS] goodrain.io/docker-zulip-postgresql-tsearchextras:latest
```

# 项目参与和讨论

如果你觉得这个镜像很有用或者愿意共同改进项目，可以通过如下形式参与：

- 如果有新特性或者bug修复，请发送 一个 Pull 请求，我们会及时反馈。
- 可以访问我们的好雨社区参与[评论](http://t.goodrain.com/t/postgresql-with-tsearchextras-for-zulip/118/1)
