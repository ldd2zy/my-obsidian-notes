## 一、问题现象

项目通过 ROS1 容器终端运行时，可以正常读取 `parking_fence.yaml`，加载其中的电子围栏 ID 1 和 ID 2。

但是通过 CLion 直接启动 `run_hybrid_astar` 进行断点调试时，无法读取 YAML 文件，最终使用程序中写死的默认围栏 ID 123。

CLion 输出：

```
[WARN] LoadStaticFenceFromYaml failed: bad file
[WARN] No static fence loaded from YAML, using default fence.
[INFO] Final static fence count: 1
[INFO] Fence ID: 123, point count: 4
```

原本希望加载的围栏：

- ID 1
    
- ID 2
    

实际只加载了默认围栏：

- ID 123
    

## 二、问题排查过程

### 1. 检查 ROS 节点和私有参数

查看当前运行节点：

```
rosnode list
```

输出：

```
/rosout
/run_hybrid_astar
```

检查节点私有参数：

```
rosparam get /run_hybrid_astar/static_fence_yaml_path
```

最初返回：

```
ERROR: Parameter [/run_hybrid_astar/static_fence_yaml_path] is not set
```

说明 CLion 启动的节点没有获得对应的 ROS 私有参数。

尝试手动设置：

```
rosparam set /run_hybrid_astar/static_fence_yaml_path \
/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml
```

再次检查：

```
rosparam get /run_hybrid_astar/static_fence_yaml_path
```

成功返回：

```
/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml
```

说明 ROS 参数服务器已经存在该参数。

### 2. 检查 YAML 文件是否存在

在 ROS1 容器中执行：

```
ls -l /home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml
```

文件存在：

```
-rwxr-xr-x 1 1000 1000 619 ... parking_fence.yaml
```

但 CLion 仍然提示：

```
LoadStaticFenceFromYaml failed: bad file
```

因此需要进一步检查程序实际运行环境能否访问该文件。

### 3. 在 C++ 中增加文件访问诊断

为了区分参数传递错误和文件访问错误，在 `LoadStaticFenceFromYaml()` 中临时增加诊断代码。

头文件：

```
#include <unistd.h>
#include <cerrno>
#include <cstring>
#include <fstream>
```

诊断代码：

```
ROS_INFO_STREAM("========== YAML FILE DEBUG ==========");
ROS_INFO_STREAM("YAML path: " << yaml_path);

ROS_INFO_STREAM("access result: "
                << access(yaml_path.c_str(), R_OK));

ROS_INFO_STREAM("errno: " << errno
                << ", error: " << strerror(errno));

std::ifstream file(yaml_path);

ROS_INFO_STREAM("ifstream is_open: " << file.is_open());
ROS_INFO_STREAM("ifstream good: " << file.good());

ROS_INFO_STREAM("=====================================");
```

CLion 输出：

```
YAML path:
/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml

access result: -1
errno: 2, error: No such file or directory

ifstream is_open: 0
ifstream good: 0
```

由此确认：

1. ROS 参数已经成功读取。
    
2. YAML 路径字符串已经成功传入函数。
    
3. 但是当前 CLion 程序运行环境中找不到该文件。
    
4. `YAML::LoadFile()` 的 `bad file` 是文件无法打开导致的。
    

问题已经从 ROS 参数传递转移到了 Docker 文件路径映射。

### 4. 检查 CLion 和 ROS1 是否使用同一个 Docker 容器

在 Ubuntu 宿主机执行：

```
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Image}}\t{{.Mounts}}"
```

发现存在两个容器：

```
CONTAINER ID   NAMES           IMAGE
0c6bb603ac24   sharp_einstein  my-perfect-ros:v1
db6ead45e3b2   ros1            my-perfect-ros:v1
```

其中：

- `ros1`：平时通过 `ros1` 命令进入的 ROS1 容器。
    
- `sharp_einstein`：CLion Docker Toolchain 自动创建的调试容器。
    

虽然两个容器使用同一个 Docker 镜像，但它们是两个独立容器，文件系统和挂载目录不一样。

### 5. 检查两个容器的目录映射

#### CLion 调试容器

执行：

```
docker inspect sharp_einstein \
--format '{{.Id}} {{json .Mounts}}'
```

关键结果：

```
宿主机：
/home/ldd/mult_truck-main/src

映射到容器：
/tmp/src
```

CLion 将宿主机的 `src` 目录映射到了容器的 `/tmp/src`。

因此，CLion 容器中的 YAML 文件实际路径应该是：

```
/tmp/src/ConfigFile/yaml/parking_fence.yaml
```

#### ROS1 容器

执行：

```
docker inspect ros1 \
--format '{{.Id}} {{json .Mounts}}'
```

关键结果：

```
宿主机：
/home/ldd/mult_truck-main

映射到容器：
/home/ldd/mult_truck-main
```

ROS1 容器挂载的是整个工程目录。

因此，ROS1 容器中的 YAML 文件路径是：

```
/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml
```

### 6. 两个容器的路径对比

|环境|YAML 文件路径|
|---|---|
|Ubuntu 宿主机|`/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml`|
|ROS1 容器|`/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml`|
|CLion 调试容器|`/tmp/src/ConfigFile/yaml/parking_fence.yaml`|

**根本原因：两个容器访问同一份宿主机文件时，使用的容器内部绝对路径不同。**

## 三、最终解决方法

不修改 C++ 源码，不修改 YAML 内容，不修改 Docker 挂载配置。

只修改 ROS 参数中的 YAML 路径，使其指向 CLion 容器中的实际路径。

### 1. 验证 CLion 容器中的文件

在 Ubuntu 宿主机执行：

```
docker exec sharp_einstein \
ls -l /tmp/src/ConfigFile/yaml/parking_fence.yaml
```

确认文件存在。

### 2. 设置 ROS 私有参数

进入 ROS1 容器：

```
ros1
```

选择：

```
1：进入容器终端
```

执行：

```
rosparam set /run_hybrid_astar/static_fence_yaml_path \
/tmp/src/ConfigFile/yaml/parking_fence.yaml
```

检查：

```
rosparam get /run_hybrid_astar/static_fence_yaml_path
```

返回：

```
/tmp/src/ConfigFile/yaml/parking_fence.yaml
```

### 3. 重新启动 CLion 调试

1. 停止 CLion 当前调试程序。
    
2. 保留刚才设置的 ROS 参数。
    
3. 点击 CLion 绿色调试按钮。
    
4. 检查围栏加载日志。
    

最终成功加载：

```
1
2

Loaded static fences from yaml, count=2
```

不再使用默认围栏 ID 123。

---

## 四、ROS 三种常见参数传递方式

ROS1 中常见的参数传递方式主要有三种：

1. roslaunch 配置文件传参
    
2. rosparam set 命令行传参
    
3. CLion 启动前任务自动传参
    

### 方式一：roslaunch 配置文件传参

这是 ROS 项目正常运行时最常用的参数配置方式。

通过 `.launch` 文件统一加载参数，再启动 ROS 节点。

#### 1. 设置单个参数

```
<launch>

    <node pkg="hybrid_a_star"
          type="run_hybrid_astar"
          name="run_hybrid_astar"
          output="screen">

        <param name="static_fence_yaml_path"
               value="/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml"/>

    </node>

</launch>
```

对应 C++ 读取：

```
ros::NodeHandle pnh("~");

std::string yaml_path;

pnh.getParam("static_fence_yaml_path", yaml_path);
```

节点名称为：

```
/run_hybrid_astar
```

对应私有参数：

```
/run_hybrid_astar/static_fence_yaml_path
```

#### 2. 加载整个 YAML 参数文件

例如：

```
vehicle_length: 8.2
vehicle_width: 3.5
wheel_base: 4.975
```

launch 文件：

```
<launch>

    <node pkg="hybrid_a_star"
          type="run_hybrid_astar"
          name="run_hybrid_astar"
          output="screen">

        <rosparam command="load"
                  file="$(find hybrid_a_star)/config/planner.yaml"/>

    </node>

</launch>
```

这样 YAML 中的参数会加载到节点对应的命名空间中。

#### 3. 优缺点

优点：

- 参数集中管理。
    
- 不需要每次手动设置。
    
- 适合正常 ROS 系统运行。
    
- 可以统一启动多个节点。
    
- 支持加载整个 YAML 参数文件。
    

缺点：

- CLion 直接启动可执行文件时，不会自动读取 launch 文件。
    
- 如果需要使用 launch 启动并进行调试，需要采用相应的调试方式。
    

### 方式二：rosparam set 命令行传参

适合临时调试和修改参数。

#### 1. 设置参数

```
rosparam set /run_hybrid_astar/static_fence_yaml_path \
/tmp/src/ConfigFile/yaml/parking_fence.yaml
```

#### 2. 查看参数

```
rosparam get /run_hybrid_astar/static_fence_yaml_path
```

#### 3. 查看所有参数

```
rosparam list
```

#### 4. 删除参数

```
rosparam delete /run_hybrid_astar/static_fence_yaml_path
```

#### 5. 加载整个 YAML 参数文件

```
rosparam load planner.yaml /run_hybrid_astar
```

#### 6. 保存参数

```
rosparam dump params_backup.yaml
```

#### 7. 优缺点

优点：

- 操作简单。
    
- 适合快速测试。
    
- 不需要修改源码。
    
- 可以随时修改参数。
    

缺点：

- 需要手动执行。
    
- ROS Master 重启后需要重新加载。
    
- 不适合大量参数的长期管理。
    
- 注意参数命名空间必须和 C++ 读取位置一致。
    

### 方式三：CLion Before launch 自动传参

适合 CLion 直接启动 C++ 节点，同时需要 GDB 断点调试的场景。

核心思路：

在 CLion 启动 C++ 程序之前，自动执行一个脚本，提前设置 ROS 参数。

#### 1. 创建脚本

在 Ubuntu 宿主机创建：

```
mkdir -p ~/scripts
```

创建文件：

```
gedit ~/scripts/set_fence_param.sh
```

写入：

```
#!/bin/bash

# 检查 ROS1 容器是否运行
if ! docker ps --format '{{.Names}}' | grep -qx 'ros1'; then
    echo "错误：ROS1 容器未运行"
    exit 1
fi

# 设置 ROS 私有参数
docker exec ros1 bash -lc '
source /opt/ros/noetic/setup.bash

rosparam set /run_hybrid_astar/static_fence_yaml_path \
/tmp/src/ConfigFile/yaml/parking_fence.yaml

rosparam get /run_hybrid_astar/static_fence_yaml_path
'

if [ $? -ne 0 ]; then
    echo "参数设置失败"
    exit 1
fi

echo "电子围栏参数设置成功"
```

#### 2. 添加执行权限

```
chmod +x ~/scripts/set_fence_param.sh
```

#### 3. 测试脚本

```
~/scripts/set_fence_param.sh
```

正常输出：

```
/tmp/src/ConfigFile/yaml/parking_fence.yaml
电子围栏参数设置成功
```

#### 4. 配置 CLion External Tools

打开：

```
File
  └── Settings
       └── Tools
            └── External Tools
```

点击 `+` 新建工具。

配置：

|配置项|内容|
|---|---|
|Name|Set Fence YAML Param|
|Program|`/home/ldd/scripts/set_fence_param.sh`|
|Arguments|留空|
|Working directory|`$ProjectFileDir$`|

保存。

#### 5. 配置 Before launch

打开：

```
Run
  └── Edit Configurations
       └── run_hybrid_astar
            └── Before launch
```

点击 `+`，选择：

```
Run External Tool
```

选择刚才创建的：

```
Set Fence YAML Param
```

保存。

#### 6. 最终启动流程

以后点击 CLion 绿色调试按钮：

```
执行 External Tool
        |
        v
执行 set_fence_param.sh
        |
        v
通过 docker exec 进入 ros1 容器
        |
        v
执行 rosparam set
        |
        v
设置 ROS 私有参数
        |
        v
CLion 启动 run_hybrid_astar
        |
        v
C++ 节点读取 ROS 参数
        |
        v
加载 parking_fence.yaml
```

#### 7. 优缺点

优点：

- 保留 CLion 直接启动程序的方式。
    
- 保留 GDB 断点调试。
    
- 不需要修改 C++ 源码。
    
- 可以自动设置多个 ROS 参数。
    
- 多个调试配置可以复用同一个参数初始化脚本。
    

缺点：

- 需要提前启动 ROS Master。
    
- 需要 ROS1 容器正常运行。
    
- 需要维护脚本中的参数命名空间和文件路径。
    
- 容器或节点名称变化时，需要同步调整脚本。
    

---

## 五、三种参数传递方式对比

|对比项|roslaunch|rosparam set|CLion Before launch|
|---|---|---|---|
|参数配置位置|launch 文件|终端命令|初始化脚本|
|是否自动执行|启动 launch 时自动|手动执行|启动 CLion 时自动|
|是否修改源码|不需要|不需要|不需要|
|是否支持多个参数|支持|支持|支持|
|是否适合正式运行|非常适合|一般|一般|
|是否适合 CLion 断点调试|需要额外调试配置|适合|非常适合|
|参数管理|集中管理|临时管理|脚本统一管理|

## 六、ROS 私有参数命名空间需要注意

C++ 中：

```
ros::NodeHandle pnh("~");
```

表示使用当前节点的私有命名空间。

例如节点名称：

```
/run_hybrid_astar
```

那么：

```
pnh.getParam("static_fence_yaml_path", yaml_path);
```

实际读取：

```
/run_hybrid_astar/static_fence_yaml_path
```

如果节点名称变成：

```
/run_hybrid_astar_agent1
```

实际读取的参数就变成：

```
/run_hybrid_astar_agent1/static_fence_yaml_path
```

因此，设置参数时必须保证节点名称和参数命名空间一致。

## 七、本次问题的经验总结

### 1. Docker 容器之间的绝对路径不一定相同

即使两个容器使用同一个镜像，也可能拥有不同的目录挂载。

同一个宿主机文件：

```
/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml
```

在 ROS1 容器中：

```
/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml
```

在 CLion 调试容器中：

```
/tmp/src/ConfigFile/yaml/parking_fence.yaml
```

必须以当前运行进程所在容器的文件路径为准。

### 2. ROS 参数读取成功不代表文件一定存在

本次已经证明：

```
Has param = 1
```

但仍然出现：

```
bad file
```

原因是 ROS 参数服务器只负责保存和传递路径字符串，不负责保证路径对应的文件存在。

### 3. 区分参数错误、路径错误和 YAML 格式错误

排查顺序：

```
第一步：检查 ROS 参数是否存在
第二步：检查参数内容是否正确
第三步：检查当前容器中路径是否存在
第四步：检查文件是否具有读取权限
第五步：检查 YAML::LoadFile() 是否成功
第六步：检查 YAML 数据结构和解析逻辑
```

### 4. 不要一遇到 YAML 加载失败就修改解析代码

本次 `YAML::LoadFile()` 没有问题。

真正的问题是 CLion Docker 容器内部不存在传入的绝对路径。

通过检查 Docker 容器挂载关系就解决了问题。

---

## 八、最终结论

本次问题的根本原因：

**CLion Docker Toolchain 自动创建了独立的调试容器** `**sharp_einstein**`**，该容器将宿主机工程的** `**src**` **目录挂载到了** `**/tmp/src**`**，而 ROS1 容器** `**ros1**` **将整个工程挂载到了** `**/home/ldd/mult_truck-main**`**。两个容器的文件路径不同，导致 CLion 无法读取原来的 YAML 文件。**

最终通过将 ROS 参数：

```
/home/ldd/mult_truck-main/src/ConfigFile/yaml/parking_fence.yaml
```

修改为：

```
/tmp/src/ConfigFile/yaml/parking_fence.yaml
```

解决了 YAML 文件读取问题。

**以后开发建议：**

- 正常 ROS 系统运行：优先使用 roslaunch 管理参数。
    
- 临时测试：使用 rosparam set。
    
- CLion 直接启动并断点调试：使用 Before launch 自动初始化参数。
    
- 多节点、多参数项目：尽量统一管理参数，避免每个节点重复配置。
    

核心原则：参数命名空间正确、Docker 文件路径正确、ROS Master 一致。