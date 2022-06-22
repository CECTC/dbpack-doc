# 快速开始

> 在你的本地环境运行 DBPack。

## 获取代码

通过 Git Clone 代码到你的本地文件夹：

```
git clone git@github.com:cectc/dbpack.git
```

确保 Git 已经安装到你的开发环境，并设置好 `PATH` 环境变量。

## 编译

进入代码目录，运行 `make build-local` 进行编译，完成后，在 `dist` 文件夹下将生成对应的二进制文件。

## Docker 镜像

```
docker pull cectc/dbpack:latest
```

dbpack 默认读取根目录下的 config.yaml 配置文件，启动 dbpack 时请为其指定配置文件。

例如：

```
docker run -p 13306:13306 --name dbpack -v /root/config.yaml:/config.yaml -d cectc/dbpack:latest
```

