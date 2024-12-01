+++
date = '2024-12-01T20:18:00+08:00'
title = "OpenGork 安装与配置"
+++
参考: 
- <https://github.com/oracle/opengrok/wiki/How-to-setup-OpenGrok> 
- <https://hub.docker.com/r/opengrok/docker/>

## Docker
首先创建存放 OpenGrok 数据的文件夹:
```sh
mkdir ~/opengrok/{src,etc,data}
```

创建 `docker-compose.yml`, 并输入如下内容:
```yaml
version: "3"

# More info at https://github.com/oracle/opengrok/docker/
services:
  opengrok:
    container_name: opengrok
    image: opengrok/docker:latest
    ports:
      - "8080:8080/tcp"
    environment:
      SYNC_PERIOD_MINUTES: '60'
    # Volumes store your data between container upgrades
    volumes:
       - '~/opengrok/src/:/opengrok/src/'  # source code
       - '~/opengrok/etc/:/opengrok/etc/'  # folder contains configuration.xml
       - '~/opengrok/data/:/opengrok/data/'  # index and other things for source code
```
其中, `~/opengrok/src/` 等是你本机上的的路径, 且:
- `~/opengrok/src/` 存放源码的目录;
- `~/opengrok/etc/` 存放配置文件的目录;
- `~/opengrok/data/` 存放文件索引的目录.

使用 `docker-compose` 工具来启动容器:
```
docker-compose up -d
```
我们此时就可以在 `localhost:8080` 处访问到 OpenGrok 的服务了.

但由于我们还未向 `src` 目录放置源码并创建索引, 此时访问站点就什么也没有.

假设我们有项目 `/path/to/llvm-project` 和 `/path/to/triton`, 那么我们需要将其复制到 `src` 目录:
```
cp -r /path/to/llvm-project ~/opengrok/src/
cp -r /path/to/triton ~/opengrok/src/
```

*注意*: 单纯的符号链接好像行不通!

然后进入 docker 容器中:
```
docker exec -it opengrok bash
cd /opengrok/
```
使用 `opengrok-indexer` 命令来创建索引:
```
opengrok-indexer \
-J=-Djava.util.logging.config.file=/opengrok/etc/logging.properties \
-a /opengrok/lib/opengrok.jar -- \
-c /usr/local/bin/ctags \
-s /opengrok/src -d /opengrok/data -H -P -S -G \
-W /opengrok/etc/configuration.xml -U http://localhost:8080/
```
这个与 [How-to-setup-OpenGrok](https://github.com/oracle/opengrok/wiki/How-to-setup-OpenGrok) 中的命令相比, 修改了两处:
- `/opengrok/dist/lib/opengrok.jar` 改为 `/opengrok/lib/opengrok.jar`;
- `http://localhost:8080/source` 改为 `http://localhost:8080/`.

该命令执行完后, 就可以看到我们的 `triton` 项目了.
