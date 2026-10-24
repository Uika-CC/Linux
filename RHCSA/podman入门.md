# Podman 入门实践：构建并运行一个 Nginx 网站

## 一、核心概念

### 1. 镜像（Image）

镜像是用于创建容器的模板，包含应用程序及其运行所需的文件。

例如：

```
docker.io/library/nginx:alpine
localhost/first-nginx:1.0
```

镜像引用通常由仓库地址、镜像名称和 tag 等部分组成。

- `docker.io`：Docker Hub 的仓库地址。
- `library`：官方镜像常用的命名空间。
- `nginx`：镜像名称。
- `alpine`：一个常见的镜像标签，表示采用 Alpine Linux 基础环境的构建。
- `localhost/first-nginx:1.0`：本地构建的网站镜像及其版本标签。

查看本地镜像：

```
podman images
```

### 2. Tag（标签）

Tag 用于标识镜像的某个版本或变体。

例如：

```
first-nginx:1.0
first-nginx:2.0
```

修改网站后，可以构建新的版本，同时保留旧版本：

```
podman build -t localhost/first-nginx:2.0 .
```

注意：标签是人为定义的名称，并不保证版本一定更新。重新构建镜像也不会自动更新已经运行的容器。

### 3. 容器（Container）

容器是基于镜像创建的运行实例。同一个镜像可以创建多个容器。

镜像与容器的关系：

```
Containerfile → 镜像 → 容器
```

### 4. Containerfile 与 Dockerfile

这两个文件都可以用于描述如何构建镜像。Podman 支持这两种文件名，默认会查找 `Containerfile` 或 `Dockerfile`。

常见指令：

| 指令      | 作用                                     |
| --------- | ---------------------------------------- |
| `FROM`    | 指定基础镜像                             |
| `COPY`    | 在构建时复制文件到镜像                   |
| `WORKDIR` | 设置工作目录                             |
| `RUN`     | 在构建过程中执行命令                     |
| `CMD`     | 指定容器默认执行的命令                   |
| `EXPOSE`  | 声明应用通常使用的端口，不会自动发布端口 |

## 二、实践：部署自己的 Nginx 网站

### 1. 创建网站目录

```
mkdir -p ~/podman-lab
cd ~/podman-lab
```

创建 `index.html`：

```
cat > index.html <<'EOF'
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>我的第一个容器网站</title>
</head>
<body>
    <h1>Hello, Podman!</h1>
    <p>这个网站运行在我自己构建的容器镜像中。</p>
</body>
</html>
EOF
```

### 2. 编写 Containerfile

```
cat > Containerfile <<'EOF'
FROM docker.io/library/nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
```

解释：

- `FROM`：使用 Nginx Alpine 镜像作为基础。
- `COPY`：把当前目录的网页文件复制到 Nginx 默认的网站目录。
- `EXPOSE 80`：声明 Nginx 使用的容器端口。

### 3. 构建镜像

```
podman build -t localhost/first-nginx:1.0 .
```

其中：

- `-t`：为镜像指定名称和标签。
- `.`：指定当前目录为构建上下文，Podman 可以读取其中的 `Containerfile` 和 `index.html`。

### 4. 运行容器

```
podman run -d --name first-web -p 8090:80 localhost/first-nginx:1.0
```

参数解释：

- `-d`：后台运行。
- `--name first-web`：指定容器名称。
- `-p 8090:80`：将宿主机的 8090 端口映射到容器的 80 端口。
- `localhost/first-nginx:1.0`：指定运行的镜像。

### 5. 验证网站

查看运行中的容器：

```
podman ps
```

访问网站：

```
curl http://localhost:8090
```

如果返回自己编写的 HTML，说明网站已经成功部署。

## 三、查看容器日志

### 常用命令

```
# 查看容器日志
podman logs first-web

# 持续跟踪日志
podman logs -f first-web

# 查看最后 20 行日志
podman logs --tail 20 first-web

# 显示日志时间戳
podman logs -t first-web
```

按 `Ctrl+C` 可以退出持续跟踪模式，一般不会停止容器。

注意：Nginx 的访问日志是否能通过 `podman logs` 看到，取决于镜像的日志配置。可以先执行 `curl http://localhost:8090`，再查看日志。

### 查看容器状态

```
# 查看正在运行的容器
podman ps

# 查看所有容器，包括已经退出的
podman ps -a
```

`podman logs` 查看日志，`podman ps -a` 查看容器状态，两者用途不同。

## 四、修改网站与版本管理

修改 `index.html` 不会自动更新已经构建好的镜像，因为 `COPY` 在构建时复制文件。

修改后重新构建：

```
podman build -t localhost/first-nginx:2.0 .
```

再基于新镜像创建容器，才能运行新版网站。

注意：新镜像不会自动升级旧容器；如果容器名冲突，需要为新容器使用不同的名称。保留旧镜像可以帮助回滚，但回滚时也需要创建或启动基于旧镜像的容器。

## 五、本次实践总结

本次实践完成了以下操作：

- 编写 `Containerfile` 和 `index.html`。
- 使用 `podman build` 构建镜像。
- 使用 `podman run` 创建并启动容器。
- 使用 `-p` 配置端口映射。
- 使用 `curl` 验证网站能够正常响应。
- 学习使用 `podman images`、`podman ps -a` 和 `podman logs`。

核心理解：**镜像负责封装应用，容器负责运行应用，端口映射负责让宿主机能够通过指定端口访问容器内的服务。**