# 🐳 ROS 混合开发环境全链路配置与避坑指南

## 📌 核心环境说明
- **宿主机**: Ubuntu 24.04 (Noble)
- **代码路径**: `/home/ldd/mult_truck-main`（**铁律：代码永远放宿主机挂载进容器，绝不直接写在容器内，保证数据不死**）
- **基础镜像**: `fishros2/ros:noetic-desktop-full`
- **最终定制镜像**: `my-perfect-ros:v1`
- **IDE 版本**: PyCharm 2024.1.7 / CLion 2024.x
- **Docker 版本**: 24.0.7 (API 1.43)

---

## 🛠️ 阶段一：Docker 环境降级与系统配置（解决 IDE 不兼容坑）

> **踩坑记录 1**：宿主机自带或通过 apt 安装的 Docker 版本太新（如 29.x），API 高达 1.44+。导致 PyCharm 2024.1 连接时报 `Status 400: client version 1.24 is too old. Minimum supported API version is 1.40`。
> **踩坑记录 2**：试图通过修改 `/etc/docker/daemon.json` 来配置镜像加速，由于 JSON 格式错误（少逗号、多双引号、中文标点），导致 Docker 守护进程直接崩溃报 `invalid JSON`，连 `docker ps` 都彻底罢工。

### 1. 卸载现有高版本 Docker
```bash
sudo systemctl stop docker
sudo systemctl stop docker.socket
sudo apt-get remove -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker.io
```

### 2. 安装 Docker 24.0.7 静态二进制包（最纯净）
```bash
wget https://download.docker.com/linux/static/stable/x86_64/docker-24.0.7.tgz
tar -xzvf docker-24.0.7.tgz
sudo cp docker/* /usr/bin/
rm -rf docker-24.0.7.tgz docker/
```

### 3. 配置 Systemd 系统服务（实现开机自启，告别手动 `dockerd &`）
```bash
sudo tee /etc/systemd/system/docker.service > /dev/null <<'EOF'
[Unit]
Description=Docker Application Container Engine
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
```

### 4. 安全配置镜像加速器
> **提示**：必须保证 JSON 格式绝对正确，建议直接复制以下命令。
```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "registry-mirrors": [
    "https://docker.xuanyuan.me",
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io"
  ]
}
EOF
sudo systemctl restart docker
```

---

## 🚀 阶段二：镜像拉取与网络玄学排坑

> **踩坑记录**：直连 Docker Hub 报 `connection reset by peer`；配置了加速器依然被限流报 `toomanyrequests`；甚至拉取到一半 Docker 突然回退去连官方网又被墙。

### 1. 前缀直拉法（强制不走官方通道）
```bash
# 使用国内镜像站完整前缀拉取
sudo docker pull docker.m.daocloud.io/fishros2/ros:noetic-desktop-full

# 拉取成功后，必须将长名字 Tag 回官方名字，供后续 IDE 和脚本使用
sudo docker tag docker.m.daocloud.io/fishros2/ros:noetic-desktop-full fishros2/ros:noetic-desktop-full
```

### 2. 解决拉取被限流的技巧
- 遇到 `toomanyrequests`，**千万不要删除本地已下载的层**，等待 5-10 分钟后重试，Docker 会断点续传。
- 或者在多个镜像源（轩辕、毫秒、DaoCloud、1Panel）之间切换命令重试。

---

## 💾 阶段三：容器的备份、恢复与安全启动（核心避坑）

> **踩坑记录 1**：直接用 `docker run -it -d 镜像 /bin/bash`，因为没有前台进程，容器会**秒退（Exited 0）**。
> **踩坑记录 2**：直接 `docker commit` 备份的镜像，在 IDE 里运行报 `Status 400: No command specified`，因为 commit 丢失了启动命令。
> **踩坑记录 3**：跨大版本 Commit 的镜像，重新加载时报 `cannot execute binary file`，底层彻底损坏。

### 1. 正确的镜像备份方式（将保活命令写死进镜像）
```bash
# 备份当前容器，并强制注入 CMD 保活命令
sudo docker commit --change='CMD ["tail", "-f", "/dev/null"]' ros1 my-perfect-ros:v1
```

### 2. 传统的离线备份与恢复（最安全）
```bash
# 导出镜像为本地 tar 文件
sudo docker save -o ~/ros_backup.tar my-perfect-ros:v1

# 备份文件所有权获取
sudo chown ldd:ldd ~/ros_backup.tar

# 恢复镜像
sudo docker load -i ~/ros_backup.tar

```


### 3. 容器启动的“黄金命令”
必须使用 `tail -f /dev/null` 保活，并挂载代码、映射网络和 GUI：
```bash
# 1. 放行宿主机 X11 图形界面（重启电脑后需重新执行）
xhost +local:docker

# 2. 启动容器
sudo docker run -it -d \
  --name ros1 \
  --net=host \
  --privileged=true \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v /home/ldd/mult_truck-main:/home/ldd/mult_truck-main \
  my-perfect-ros:v1 \
  tail -f /dev/null
```

---

## 🔐 阶段四：权限设置与包管理排坑

> **踩坑记录 1**：CLion 编译报错 `Cannot create directory cmake-build-debug`，因为容器以 root 运行，在挂载目录生成的文件夹宿主机用户没权限访问。
> **踩坑记录 2**：在容器内手贱 `apt install gedit` 或 `gnome-terminal`，引发依赖地狱，安装了几百个桌面包导致终端卡死、系统崩溃。

### 1. 解决宿主机 Docker 执行权限
```bash
# 将当前用户加入 docker 组，避免每次敲 sudo（执行后必须注销重新登录系统生效）
sudo usermod -aG docker $USER
```

### 2. 解决挂载目录的编译权限
在 CLion 的 Docker 工具链 `Run options` 中填入：`--user=0`。
如果宿主机文件被容器改成了 root 权限，在宿主机强行夺回：
```bash
sudo chown -R ldd:ldd /home/ldd/mult_truck-main
```

### 3. 容器内安装依赖的正确姿势
如果必须安装软件，切勿直接裸敲 `apt install`。使用无交互模式防止卡死：
```bash
export DEBIAN_FRONTEND=noninteractive
apt install -y <软件名>
```
*(强烈建议：文本编辑用 `nano` 或 `vim`，千万别在容器装带 GUI 的文本编辑器)*

---

## 💻 阶段五：CLion (C++) 配置与防坑指南

> **踩坑记录 1**：C 编译器路径填成了 `/usr/bin/g++`，导致 CMake 探测报错 `The C compiler identification is unknown`。
> **踩坑记录 2**：在 CLion 中对单文件点“运行”，报错 `不支持远程工具链`。

### 1. 工具链配置
1. `Settings` -> `Build, Execution, Deployment` -> `Toolchains`。
2. 创建 `Docker`，镜像选 `my-perfect-ros:v1`。
3. `Run options` 填 `--user=0`。
4. **极其重要**：C 编译器必须是 `/usr/bin/gcc`，C++ 编译器必须是 `/usr/bin/g++`。

### 2. 正确运行代码的方式
不要用单文件运行。必须把代码加入 `CMakeLists.txt`：
```cmake
add_executable(hello hello.cpp)
```
点击 `Tools` -> `CMake` -> `Reset Cache and Reload Project`，选择 `hello` 目标运行。

---

## 🐍 阶段六：PyCharm (Python) 配置与防坑指南

> **踩坑记录 1**：代码满屏红线，连 `sys` 和 `os` 都找不到，其实是 PyCharm 辅助进程 `generator3` 崩溃，且未识别 ROS 的 `dist-packages` 路径。
> **踩坑记录 2**：执行 `roslaunch` 缺少 `paho` 和 `pandas` 库，节点启动后立刻崩溃死亡。
> **踩坑记录 3**：终端在宿主机，但总是误以为在容器里。

### 1. 解释器配置与 ROS 路径注入
1. `Settings` -> `Python Interpreter` -> `Add Interpreter` -> `On Docker`。镜像选 `my-perfect-ros:v1`，Python 路径选 `/usr/bin/python3`。
2. 点击解释器旁的齿轮 ⚙️ -> `Show All...` -> 📁 文件夹图标 -> 点 `+` 号，手动添加：
   `/opt/ros/noetic/lib/python3/dist-packages`
3. 如果工作空间有自定义 msg，再添加：
   `/home/ldd/mult_truck-main/devel/lib/python3/dist-packages`

### 2. 补全缺失的依赖库
进入容器内部执行：
```bash
sudo docker exec -it ros1 bash
pip install pandas paho-mqtt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

---

## 🐟 阶段七：鱼香 ROS 脚本报错还原与处理

> **踩坑记录**：运行 `wget http://fishros.com/install -O fishros && . fishros` 时，脚本执行 `sudo systemctl restart docker` 卡住（13秒超时），手动 `Ctrl+C` 中断后脚本异常退出，且改乱了 Docker 配置。

### 1. 彻底清理中断的残留并还原配置
```bash
# 干掉可能卡在后台的安装脚本进程
pkill -f fishros
pkill -f install.py
rm -f ~/fishros

# 删掉鱼香ROS 写入的加速地址（可选，若不需要可删掉整个配置文件）
sudo rm -f /etc/docker/daemon.json

# 重新加载系统配置并重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 🛠️ 阶段八：自定义 `~/.bashrc` 容器管理神器

> **踩坑记录**：容器停止时，选择“进入容器”会报错 `is not running`。原生 `docker exec bash -c "cd ..."` 退出时可能要按多次 `exit`。

请将以下代码加入 `~/.bashrc` 末尾，并执行 `source ~/.bashrc`：

```bash
ros1(){
    echo "==== ROS1容器管理 ===="
    echo "1:进入容器终端"
    echo "2:启动容器"
    echo "3:停止容器"
    echo "4:重启容器"
    echo "5:查看日志"
    read -p "请输入选项：" op
    case $op in
        1) 
            # 核心优化：先静默启动（如果已经启动会直接跳过），再进入
            sudo docker start ros1 >/dev/null 2>&1
            # 使用 -w 参数直接指定工作目录，比 bash -c 嵌套更优雅
            sudo docker exec -it -w /home/ldd ros1 bash -c "source /opt/ros/noetic/setup.bash && exec bash" 
            ;;
        2) sudo docker start ros1 ;;
        3) sudo docker stop ros1 ;;
        4) sudo docker restart ros1 ;;
        5) sudo docker logs -f ros1 ;;
        *) echo "输入错误" ;;
    esac
}
```

---

## 🧹 阶段九：高频 Docker 命令宝典与清理

### 1. 日常排查命令
```bash
# 查看正在运行的容器
sudo docker ps
# 查看所有容器（包括已退出的）
sudo docker ps -a
# 查看本地所有镜像
sudo docker images
```

### 2. 清理 IDE 产生的僵尸容器与垃圾（极度推荐）
IDE 后台探测会在 `docker ps -a` 里生成大量名为随机单词、状态为 `Created` 的僵尸容器。定期清理可释放空间：
```bash
# 清理所有已停止的容器、悬挂镜像和构建缓存
sudo docker system prune -f

# 单独清理未被使用的镜像
sudo docker image prune -f

# 强制删除指定容器或镜像
sudo docker rm -f <容器ID>
sudo docker rmi -f <镜像ID或名字>
```