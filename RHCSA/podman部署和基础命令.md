# Podman 常用命令笔记

> 学习日期：2026-10-22  
> 环境：openEuler / Linux，使用普通用户运行 Podman  
> 目标：熟悉镜像、容器生命周期、交互操作与基础排障。

## 1. 基本概念

- **镜像（Image）**：创建容器所使用的模板，通常包含应用程序及所需的用户空间依赖。
- **容器（Container）**：镜像创建出的实例，可以运行、停止、删除。
- **宿主机（Host）**：运行 Podman 和容器的系统。Linux 容器共享宿主机内核，但可以在镜像中携带 Python、Nginx 等用户空间程序。
- **端口映射**：`-p 宿主机端口:容器端口`，例如 `-p 8085:80` 表示访问宿主机的 8085 端口会被转发到容器的 80 端口。
- **数据挂载**：把宿主机上的目录或文件映射到容器中，便于保留或编辑数据。

## 2. 查看镜像和容器

| 命令 | 作用 |
|---|---|
| `podman images` | 列出本地镜像 |
| `podman image ls` | 与 `podman images` 类似，列出本地镜像 |
| `podman ps` | 列出正在运行的容器 |
| `podman ps -a` | 列出全部容器，包括已停止的容器 |
| `podman inspect <容器名或ID>` | 查看容器详细配置、状态、挂载、网络等信息 |
| `podman port <容器名或ID>` | 查看容器的端口映射 |

示例：

```bash
podman images
podman ps
podman ps -a
podman port web-lab
podman inspect web-lab
```

常见状态：

- `Up`：容器正在运行。
- `Exited`：容器已经退出。
- `Created`：容器对象已经创建，但尚未成功启动。

`podman ps` 默认只显示运行中的容器；排查“容器去哪了”时，常用 `podman ps -a`。

## 3. 获取镜像

```bash
podman pull docker.io/library/nginx:alpine
podman pull docker.io/library/python:3.12-alpine
```

- `pull`：下载镜像到本地。
- `nginx:alpine`、`python:3.12-alpine` 是镜像名和标签。
- 如果运行 `podman run` 时本地还没有对应镜像，Podman 通常会尝试拉取它。

## 4. 创建并运行容器：`podman run`

示例：启动 Nginx，并把宿主机 8085 端口映射到容器 80 端口。

```bash
podman run -d \
  --name web-lab \
  -p 8085:80 \
  docker.io/library/nginx:alpine
```

常见参数：

| 参数 | 作用 |
|---|---|
| `-d` | 在后台运行容器 |
| `--name <名称>` | 指定容器名称，名称不能与已有容器重复 |
| `-p <宿主机端口>:<容器端口>` | 配置端口映射 |
| `-it` | 常用于需要交互终端的容器 |
| `--rm` | 容器退出后自动删除容器对象 |
| `-v <宿主机路径>:<容器路径>` | 挂载宿主机目录或文件 |
| `-w <容器内路径>` | 设置容器内工作目录 |

需要注意：

1. `podman run` 会创建并启动一个新容器，不是单纯启动旧容器。
2. 即使原容器已停止，只要容器对象还存在，通常也不能复用相同名称。
3. 修改下一次 `run` 命令中的端口，不会修改此前已经创建的容器配置。
4. 若提示名称已被占用，先运行 `podman ps -a` 查看已有容器，再决定启动、删除或换名。

## 5. 容器生命周期

| 命令 | 作用 |
|---|---|
| `podman start <容器>` | 启动已有容器 |
| `podman stop <容器>` | 停止容器 |
| `podman restart <容器>` | 重启容器 |
| `podman rm <容器>` | 删除容器对象 |
| `podman rm -f <容器>` | 强制停止并删除容器；使用前确认目标 |
| `podman rmi <镜像>` | 删除本地镜像 |

关键区别：

- `start`：启动已经创建的容器。
- `run`：创建并启动新容器。
- `stop`：停止运行，但保留容器对象。
- `rm`：删除容器对象。
- `rmi`：删除镜像；若仍有容器依赖该镜像，可能无法删除。
- 删除容器不等于删除镜像，也不保证会删除挂载到宿主机的文件。

安全习惯：动手 `rm` 或 `rmi` 前，先用 `podman ps -a` 或 `podman images` 确认要操作的对象。不要随意使用大范围清理命令。

## 6. 进入容器、执行命令

进入正在运行的容器：

```bash
podman exec -it web-lab sh
```

在容器中检查文件后，可用 `exit` 返回宿主机：

```sh
ls /
exit
```

不进入交互 Shell，也可以直接执行命令：

```bash
podman exec web-lab nginx -v
```

`exec` 在**已运行的容器**内执行新命令；容器需要处于运行状态。

## 7. 查看日志与资源

```bash
podman logs web-lab
podman logs -f web-lab
podman stats web-lab
```

- `logs`：查看容器输出的日志。
- `logs -f`：持续跟踪新日志，按 `Ctrl+C` 停止跟踪（通常不会停止容器）。
- `stats`：查看容器的资源使用情况。

当网页打不开时，可以先检查：

```bash
podman ps -a
podman port web-lab
podman logs web-lab
```

然后再检查端口映射、服务日志和访问地址。

## 8. 实验：在容器里运行 Python，不在宿主机安装 Python

运行一次性 Python 命令：

```bash
podman run --rm docker.io/library/python:3.12-alpine \
  python --version
```

执行 Python 代码：

```bash
podman run --rm docker.io/library/python:3.12-alpine \
  python -c 'print("Hello from container!")'
```

- Python 解释器来自镜像，不要求宿主机另外安装 Python。
- `--rm` 会在容器退出后自动删除容器，但不会因此删除镜像。
- Linux 容器仍然使用宿主机提供的 Linux 内核和硬件资源。

**Shell 提醒：** 在交互式 Bash 中，双引号内的 `!` 可能触发历史记录展开。上例使用单引号包住 Python 代码，并在 Python 代码内部使用双引号，避免这个问题。

## 9. 实验：把宿主机目录挂载到容器

在宿主机目录中创建 `main.py` 后，可以把当前目录挂载到容器 `/app`，再执行脚本：

```bash
podman run --rm \
  -v "$PWD":/app:Z \
  -w /app \
  docker.io/library/python:3.12-alpine \
  python main.py
```

参数解释：

- `-v "$PWD":/app:Z`：把当前宿主机目录挂载到容器的 `/app`。在启用 SELinux 的 Linux 系统上，`:Z` 请求为该挂载路径设置供此容器使用的 SELinux 标签；不要随意对系统重要目录使用它。
- `-w /app`：将容器的当前工作目录设置为 `/app`。
- `python main.py`：使用镜像中的 Python 解释器执行挂载进来的脚本。

优点：修改宿主机的 `main.py` 后，再次运行命令即可执行新代码，一般不需要重建镜像。这里的代码文件保存在宿主机目录中，不会因为临时容器删除而随之消失。

## 10. Shell 多行输入：Here Document

创建文件时常见的写法：

```bash
cat > main.py <<'EOF'
import platform

print("Hello from container!")
print("Python version:", platform.python_version())
print("Running on:", platform.system())
EOF
```

- `cat` 接收输入并输出文本。
- `> main.py` 将输出写入文件；文件原来存在时会被覆盖。
- `<<'EOF'` 表示后续多行文本输入，直到单独一行 `EOF` 结束。
- `EOF` 只是自定义的结束标记，可以换成其他一致的标记。
- 结束标记加引号时，正文里的 `$HOME` 等变量不会被 Bash 提前展开。
- `>>` 是追加重定向，不会覆盖原文件。

## 11. 推荐的基础排障顺序

容器服务无法访问时，可以按这个顺序检查：

1. `podman ps -a`：确认容器是否存在、当前状态如何。
2. `podman port <容器>`：确认端口映射。
3. `podman logs <容器>`：查看应用输出和错误。
4. `podman inspect <容器>`：查看详细配置、挂载与状态。
5. 在宿主机测试服务，例如 `curl http://localhost:8085`（将端口替换为实际映射端口）。

不要一遇到问题就删除容器。先观察状态与日志，通常更有利于理解故障原因。

## 12. 今日练习清单

- [ ] `podman images`：查看镜像。
- [ ] `podman ps` 与 `podman ps -a`：比较输出。
- [ ] `podman port web-lab`：查看端口映射。
- [ ] `podman logs web-lab`：查看日志。
- [ ] `podman inspect web-lab`：浏览容器详细配置。
- [ ] `podman exec -it web-lab sh`：进入容器后使用 `exit` 返回。
- [ ] 用 Python 镜像运行一次性命令。

## 参考资料

- [Podman 官方文档](https://docs.podman.io/)
- [podman run](https://docs.podman.io/en/latest/markdown/podman-run.1.html)
- [podman ps](https://docs.podman.io/en/latest/markdown/podman-ps.1.html)
- [podman exec](https://docs.podman.io/en/latest/markdown/podman-exec.1.html)
