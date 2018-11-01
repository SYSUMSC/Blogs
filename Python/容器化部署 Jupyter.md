# 容器化部署 Jupyter

## Jupyter 介绍

[Jupyter Notebook](https://jupyter-notebook.readthedocs.io/en/stable/) 是基于网页的用于交互计算的应用程序，其可被应用于全过程计算：开发、文档编写、运行代码和展示结果。

我们知道 [Docker](https://www.docker.com/) 是一种软件容器平台，是部署应用的利器。在本篇文章中，我们通过 [Jupyter Docker Stacks](https://github.com/jupyter/docker-stacks) 快速搭建机器学习的学习环境。

## Docker 部署

### 文件目录

我们打算将容器的存储卷与本地的 `Jupyter/`  挂载，从而实现容器数据的持久化存储。

```bash
mkdir Jupyter
```

为避免在使用容器的过程中，出现权限问题，我们可以更改文件夹的使用权限。

```bash
sudo chmod -R 777 Jupyter/
```

在完成以上步骤后，我们需要添加 `start.sh` 和 `start-notebook.sh` 添加至文件夹内。

### start.sh

```shell
docker run -d -p 8888:8888 -e NB_USER=jiahonzheng --user root \
-w /home/jiahonzheng -v "$PWD":/home/jiahonzheng \
--name jupyter jupyter/tensorflow-notebook \
start-notebook.sh --NotebookApp.password='sha1 of your password'
```

我们可以参照 [Common Features](https://jupyter-docker-stacks.readthedocs.io/en/latest/using/common.html#notebook-options) 的内容，通过修改 `start.sh` ，完成容器的自定义化设置。这里，我们通过添加 `NotebookApp.password` 的配置，实现了 Jupyter 的密码设置。为获取密码的 `SHA1` 值，我们可以通过在线的 [Jupyter](http://jupyter.org/try) 执行以下指令获取。


```python
from notebook.auth import passwd
passwd()
```

### start-notebook.sh

```shell
#!/bin/bash
# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.

set -e

if [[ ! -z "${JUPYTERHUB_API_TOKEN}" ]]; then
	# launched by JupyterHub, use single-user entrypoint
	exec /usr/local/bin/start-singleuser.sh $*
else
	if [[ ! -z "${JUPYTER_ENABLE_LAB}" ]]; then
		. /usr/local/bin/start.sh jupyter lab $*
	else
		. /usr/local/bin/start.sh jupyter notebook $*
	fi
fi
```

为避免 `start.sh` 和 `start-notebook.sh` 被误删，我们可以通过 `chattr` 指令设置文件的权限。

```bash
sudo chattr +a start.sh start-notebook.sh
```

### 启动容器

最后，我们可以通过执行 `start.sh` 启动 Jupyter。

```bash
. start.sh
```

