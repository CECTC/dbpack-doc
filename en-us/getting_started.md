# Getting started

> Run DBPack at your local enviroment.

## Get Code

Git clone code to you local folder:

```
git clone git@github.com:cectc/dbpack.git
```

Make sure [Git is installed](https://git-scm.com/downloads) on your machine and in your system's `PATH`.

## Build

Go to the code directory and run `make build-local` to compile. Once completed, the corresponding binary will be generated under the `dist` folder.

## Docker Image

```
docker pull cectc/dbpack:latest
```

By default, dbpack reads the `config.yaml` configuration file in the root directory. Specify a configuration file for dbpack when you start it.

E.g:

```
docker run -p 13306:13306 --name dbpack -v /root/config.yaml:/config.yaml -d cectc/dbpack:latest
```
