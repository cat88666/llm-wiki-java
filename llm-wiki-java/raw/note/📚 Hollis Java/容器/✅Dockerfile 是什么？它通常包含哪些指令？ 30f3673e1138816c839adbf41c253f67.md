# ✅Dockerfile 是什么？它通常包含哪些指令？

所有者: junk01

# ✅Dockerfile 是什么？它通常包含哪些指令？

> 原文：[https://www.yuque.com/hollis666/asgng6/kicy7xnzotves07g](https://www.yuque.com/hollis666/asgng6/kicy7xnzotves07g)
> 

# 典型回答

Dockerfile 是一个文本文件，用于自动化地构建 Docker 镜像。Dockerfile 定义了从基础镜像开始，按步骤配置环境和部署应用所需要的所有命令。

如以下dockerfile：

1. **FROM** - 设置基础镜像，每个 Dockerfile 至少需要一条 FROM 指令作为镜像的基础。 
2. **LABEL** - 添加元数据到镜像，如作者、版本、描述等。 
3. **ENV** - 设置环境变量。 

 ****

1. **WORKDIR** - 为 RUN, CMD, ENTRYPOINT, COPY 和 ADD 设置工作目录。 

 ****

1. **ADD** 和 **COPY** - 将文件从构建环境复制到镜像。COPY 是推荐的方法，因为它更透明。ADD 除了复制本地文件还可以直接解压缩和从URL下载。 

 ****

1. **EXPOSE** - 声明容器运行时监听的端口。 

 ****

1. **HEALTHCHECK** - 告诉 Docker 如何测试容器以检查它是否仍在运行。 

 ****

1. **ENTRYPOINT** - 配置容器启动时运行的命令，允许将容器作为可执行文件。 

  ****

1. **CMD** - 提供容器默认的执行命令。每个 Dockerfile 只能有一个 CMD 指令，如果列出多个，只有最后一个会生效。 

 ****

1. **ONBUILD - 为镜像添加将在之后基于该镜像的 Dockerfile 中触发的触发器指令。** 

 ****

1. **RUN** - 执行命令并创建新的镜像层，常用于安装软件包。 
2. **VOLUME** - 创建挂载点来持久化数据或与其他容器共享数据。 
3. **USER** - 设置运行容器时的用户名或 UID。 
4. **ARG** - 定义构建时的变量，可用于传递动态数据如代理服务器、版本标签等。