## 1. 环境说明

### 1.1 软件环境

|项目|配置|
|---|---|
|操作系统|Ubuntu 20.04|
|ROS 版本|ROS Noetic|
|IDE|CLion 2026.3|
|编译工具|CMake + GCC + G++|
|调试工具|GDB|
|容器环境|Docker|
|Docker 镜像|`my-perfect-ros:v1`|
|容器名称|`ros1`|
|CMake 生成器|Unix Makefiles|
|C++ 标准|C++14|

### 1.2 项目目录结构

宿主机项目路径：

```
/home/ldd/mult_truck-main/
├── build/
├── devel/
└── src/
    ├── CMakeLists.txt
    ├── ConfigFile/
    ├── control/
    ├── decision/
    ├── hybrid_a_star/
    ├── msg_gen/
    ├── multtruck_pkg/
    ├── v2n/
    └── test.py
```

### 1.3 Docker 环境说明

宿主机项目目录：

```
/home/ldd/mult_truck-main
```

挂载到 Docker 容器中的相同路径。

使用宿主机脚本：

```
ros1
```

选择容器后进入 ROS 容器。

容器提示符示例：

```
root@ldd-ThinkBook-16-G8-IRL:/home/ldd#
```

注意：

- CLion 在宿主机运行。
    
- ROS 环境和编译工具在 Docker 容器中。
    
- CLion Docker Toolchain 会创建自己的临时工具容器。
    
- 临时工具容器与长驻的 `ros1` 容器不是同一个容器。
    
- 不要把两个容器的文件路径混为一谈。
    

---

## 2. 配置项目 CMakeLists.txt

### 2.1 检查 CMakeLists.txt

进入 ROS 容器：

```
ros1
```

检查项目文件：

```
ls -l /home/ldd/mult_truck-main/src/CMakeLists.txt
```

原始文件是一个软链接：

```
CMakeLists.txt -> /opt/ros/noetic/share/catkin/cmake/toplevel.cmake
```

由于宿主机没有 `/opt/ros/noetic`，CLion 在宿主机侧无法正常解析这个软链接。

### 2.2 将软链接替换为普通文件

在 ROS 容器中执行：

```
cp /opt/ros/noetic/share/catkin/cmake/toplevel.cmake \
/home/ldd/mult_truck-main/src/CMakeLists.txt
```

然后回到宿主机终端，修改文件所有权：

```
sudo chown ldd:ldd /home/ldd/mult_truck-main/src/CMakeLists.txt
```

检查：

```
ls -l /home/ldd/mult_truck-main/src/CMakeLists.txt
```

确认它已经是普通文件，而不是软链接。

---

## 3. CLion 打开项目

启动 CLion。

选择：

```
File → Open
```

打开：

```
/home/ldd/mult_truck-main/src
```

注意：

这里打开的是 `src` 目录，而不是工作空间根目录。

因为 `src` 目录下包含整个 Catkin 工作空间的顶层 `CMakeLists.txt`。

---

## 4. 配置 Docker Toolchain

打开：

```
Settings → Build, Execution, Deployment → Toolchains
```

添加 Docker Toolchain。

### 4.1 Docker 配置

选择 Docker，并指定镜像：

```
my-perfect-ros:v1
```

### 4.2 容器启动参数

在 Container options 中填写：

```
-u 0 --network host -e LD_LIBRARY_PATH=/opt/ros/noetic/lib -e ROS_MASTER_URI=http://127.0.0.1:11311 -e ROS_HOSTNAME=127.0.0.1
```

参数说明：

|参数|作用|
|---|---|
|`-u 0`|使用 root 用户运行容器|
|`--network host`|与宿主机共享网络|
|`LD_LIBRARY_PATH`|指定 ROS 动态库搜索路径|
|`ROS_MASTER_URI`|指定 ROS Master 地址|
|`ROS_HOSTNAME`|指定 ROS 通信主机地址|

### 4.3 检查工具链

确认 CLion 能识别：

- CMake
    
- C Compiler（GCC）
    
- C++ Compiler（G++）
    
- Debugger（GDB）
    

以上工具均正常识别后，保存配置。

注意：

不要额外创建或选择 ROS 环境变量文件。当前配置已经通过容器启动参数设置了所需环境变量。

---

## 5. 配置 CMake Profile

打开：

```
Settings → Build, Execution, Deployment → CMake
```

配置如下：

|配置项|内容|
|---|---|
|Name|Debug|
|Build type|Debug|
|Toolchain|前面配置的 Docker Toolchain|
|Generator|Unix Makefiles|
|CMake options|`-DCMAKE_PREFIX_PATH=/opt/ros/noetic`|

CMake options：

```
-DCMAKE_PREFIX_PATH=/opt/ros/noetic
```

Build directory 使用 CLion 当前配置生成的构建目录。

不要随意修改已经正常工作的构建目录。

### 5.1 CMake 配置过程

CLion 会在 Docker 工具容器中执行类似命令：

```
cmake \
-DCMAKE_BUILD_TYPE=Debug \
-DCMAKE_PREFIX_PATH=/opt/ros/noetic \
-G "Unix Makefiles" \
-S /tmp/src \
-B /tmp/src/cmake-build-debug-docker
```

注意：

`/tmp/src` 是 CLion 临时 Docker 工具容器中的项目映射路径，不是长驻 `ros1` 容器中的路径。

### 5.2 检查 CMake 配置结果

正常情况下可以看到：

```
-- Configuring done
-- Generating done
-- Build files have been written to:
```

并且 CLion 能够识别项目中的 CMake Targets。

当前项目识别到的主要 ROS 包：

```
multtruck_pkg
msg_gen
decision
hybrid_a_star
v2n
```

---

## 6. 配置 ROS Master

CLion 负责编译和运行程序，但 ROS 节点运行时还需要 ROS Master。

### 6.1 启动 ROS Master

打开一个宿主机终端：

```
ros1
```

进入容器后执行：

```
source /opt/ros/noetic/setup.bash
roscore
```

保持这个终端运行，不要关闭。

### 6.2 验证 ROS Master

另外打开一个终端，进入 ROS 容器：

```
ros1
```

执行：

```
pgrep -af rosmaster
```

如果能够看到类似：

```
/usr/bin/python3 /opt/ros/noetic/bin/rosmaster --core -p 11311
```

说明 ROS Master 已经启动。

### 6.3 验证网络连接

在宿主机终端执行：

```
python3 -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1',11311)); s.close()"
```

注意：正确的完整命令如下：

```
python3 -c "import socket; s=socket.socket(); print(s.connect_ex(('127.0.0.1',11311))); s.close()"
```

输出：

```
0
```

表示宿主机能够连接 ROS Master。

---

## 7. 配置 CLion 运行环境变量

打开：

```
Run → Edit Configurations
```

选择对应的 CMake Application 运行配置。

在 Environment variables 中添加：

```
ROS_MASTER_URI=http://127.0.0.1:11311
ROS_HOSTNAME=127.0.0.1
```

例如：

```
ROS_MASTER_URI=http://127.0.0.1:11311;ROS_HOSTNAME=127.0.0.1
```

这样 CLion 启动的 ROS 节点就知道应该连接哪个 ROS Master。

注意：

这里的运行配置环境变量和 Docker Toolchain 的容器启动参数作用不同：

- Toolchain 参数：设置 CLion 临时工具容器的运行环境。
    
- Run Configuration：设置具体 ROS 程序启动时的运行环境。
    

两处均保留当前已经验证成功的配置。

---

## 8. 配置运行目标（CMake Target）

这是 CLion 与 PyCharm 使用体验上的一个重要区别。

CLion 的 CMake Application 运行配置是围绕 CMake Target 建立的。

### 8.1 查看项目中的可执行目标

例如 `hybrid_a_star/CMakeLists.txt` 中：

```
add_library(
    PATH_SEARCHER_LIB SHARED
    src/goal_pose_subscriber.cpp
    src/init_pose_subscriber.cpp
    src/hybrid_a_star_flow.cpp
    src/parking_obstacle_utils.cpp
    src/parking_path_utils.cpp
    src/xml_parser.cpp
)

add_executable(
    run_hybrid_astar
    app/run_hybrid_astar.cpp
)

target_link_libraries(
    run_hybrid_astar
    PATH_SEARCHER_LIB
)

add_executable(
    path_radius_analyzer
    src/path_radius_analyzer.cpp
)

target_link_libraries(
    path_radius_analyzer
    ${catkin_LIBRARIES}
)
```

这里有三个重要概念：

|名称|类型|是否可以独立运行|
|---|---|---|
|`PATH_SEARCHER_LIB`|SHARED 动态库|否|
|`run_hybrid_astar`|可执行程序|是|
|`path_radius_analyzer`|可执行程序|是|

### 8.2 创建运行配置

打开：

```
Run → Edit Configurations
```

点击：

```
+ → CMake Application
```

选择：

```
Target: run_hybrid_astar
```

配置环境变量：

```
ROS_MASTER_URI=http://127.0.0.1:11311
ROS_HOSTNAME=127.0.0.1
```

保存配置。

### 8.3 其他可执行程序

如果需要运行：

```
path_radius_analyzer
```

再创建一个对应的 CMake Application 配置。

每个常用可执行目标只需要配置一次，之后可以直接从顶部运行配置下拉框切换。

不需要每次重新创建运行配置。

### 8.4 如何调试普通 cpp 文件

例如：

```
hybrid_a_star_flow.cpp
```

它属于动态库：

```
PATH_SEARCHER_LIB
```

不是独立可执行程序，因此不能直接作为独立程序启动。

正确操作：

1. 打开 `hybrid_a_star_flow.cpp`。
    
2. 在需要调试的位置打断点。
    
3. 顶部运行配置选择 `run_hybrid_astar`。
    
4. 点击 Debug。
    
5. 当程序执行到断点位置时，CLion 会暂停。
    

原理：

```
run_hybrid_astar
        |
        v
PATH_SEARCHER_LIB
        |
        v
hybrid_a_star_flow.cpp
        |
        v
执行断点位置
```

只要多个 cpp 文件属于同一个可执行程序所链接的动态库，就可以使用同一个运行配置进行调试。

---

## 9. 编译、运行和调试

### 9.1 编译

选择对应的 CMake Target，点击 Build。

也可以使用快捷键：

```
Ctrl + F9
```

### 9.2 运行

选择对应的运行配置，点击绿色三角形 Run。

### 9.3 Debug

在代码中设置断点，选择对应的运行配置，点击 Debug。

### 9.4 注意事项

- 普通 cpp 文件不一定是独立可执行程序。
    
- 需要找到它所属的 CMake Target。
    
- 如果 cpp 属于动态库，应运行链接该动态库的可执行程序。
    
- 不同可执行程序需要各自的运行配置。
    
- 已经保存的运行配置不需要重复创建。
    

---

## 10. 验证 ROS 节点运行

运行：

```
run_hybrid_astar
```

正常情况下可以看到类似日志：

```
[INFO] offset_x 1
[INFO] offset_y 1
[INFO] vehicle_length 8.2
[INFO] vehicle_width 3.5
[INFO] wheel_base 4.975
[INFO] steering_angle 30
[INFO] map_resolution 0.5
[INFO] hybrid_a_star_max_iterations 20000
[INFO] rs_segment_direction_mode diverse
[WARN] No 'planner/target_roadmark_ids' param found. Will parse all roadMark objects.
[INFO] Published elec_fence (id 123) to /elec_fence, 4 corners (closed).
```

其中：

```
Published elec_fence to /elec_fence
```

说明程序已经成功发布 ROS Topic。

### 10.1 检查 Topic

在 ROS 容器中执行：

```
source /opt/ros/noetic/setup.bash
rostopic list
```

可以看到项目发布的 Topic，例如：

```
/elec_fence
/event_flag
/fusion/lidar_obstacle
/gps_imu
/initialpose
/move_base_simple/goal
/parking_path_from_hybrid_Astar
/parking_path_info_to_control
/parking_point/from_Decision_to_Parking
/parking_state
/rosout
/rosout_agg
/run_hybrid_astar/parsed_obstacle_points
/run_hybrid_astar/searched_path
/run_hybrid_astar/searched_tree
/run_hybrid_astar/vehicle_path
```

### 10.2 检查 Topic 消息

例如：

```
rostopic echo -n 1 /elec_fence
```

如果能够接收到 Marker 消息，说明 ROS 节点已经成功向 ROS Master 注册并发布消息。

---

## 11. 常见问题及解决方法

### 11.1 找不到 ROS 动态库

报错：

```
error while loading shared libraries:
libxmlrpcpp.so: cannot open shared object file
```

原因：

CLion 临时 Docker 工具容器没有正确设置 ROS 动态库搜索路径。

解决：

在 Docker Toolchain 的 Container options 中添加：

```
-e LD_LIBRARY_PATH=/opt/ros/noetic/lib
```

### 11.2 ROS Master Connection refused

报错：

```
Error in XmlRpcClient::writeRequest: write error (Connection refused)
```

原因可能是：

- ROS Master 没有启动。
    
- CLion 工具容器无法访问 ROS Master。
    
- ROS_MASTER_URI 配置不正确。
    

解决：

1. 在长驻 `ros1` 容器中启动 `roscore`。
    
2. Docker Toolchain 添加：
    

```
--network host
```

3. Toolchain 和运行配置中设置：
    

```
ROS_MASTER_URI=http://127.0.0.1:11311
ROS_HOSTNAME=127.0.0.1
```

### 11.3 CLion 找不到 CMake Target

检查：

1. `src/CMakeLists.txt` 是否正确。
    
2. Docker Toolchain 是否正常。
    
3. CMake Profile 是否选择正确的 Toolchain。
    
4. CMake 是否成功完成 Configure 和 Generate。
    

必要时重新加载 CMake 项目。

### 11.4 打开 cpp 文件后运行了错误的程序

原因：

顶部选中的运行配置与当前 cpp 文件所属的可执行程序不一致。

解决：

先确认 cpp 所属的 CMake Target，再选择正确的运行配置。

---

## 12. 最终配置清单

### Docker Toolchain

```
镜像：my-perfect-ros:v1
用户：root
网络：host
```

Container options：

```
-u 0 --network host -e LD_LIBRARY_PATH=/opt/ros/noetic/lib -e ROS_MASTER_URI=http://127.0.0.1:11311 -e ROS_HOSTNAME=127.0.0.1
```

### CMake Profile

```
Build type：Debug
Generator：Unix Makefiles
Toolchain：Docker Toolchain
CMake options：-DCMAKE_PREFIX_PATH=/opt/ros/noetic
```

### Run Configuration

```
类型：CMake Application
Target：run_hybrid_astar
```

Environment variables：

```
ROS_MASTER_URI=http://127.0.0.1:11311
ROS_HOSTNAME=127.0.0.1
```

### ROS Master

```
source /opt/ros/noetic/setup.bash
roscore
```

### 最终工作流程

```
启动 Docker 容器
       |
       v
启动 ROS Master（roscore）
       |
       v
打开 CLion 项目
       |
       v
CLion Docker Toolchain
       |
       v
CMake Configure / Generate
       |
       v
识别 CMake Targets
       |
       v
选择对应运行配置
       |
       v
Build / Run / Debug
       |
       v
ROS 节点连接 ROS Master
       |
       v
检查 ROS Topic 和运行结果
```

**配置完成后：**

- Docker Toolchain 不需要重复配置。
    
- CMake Profile 不需要重复配置。
    
- ROS 环境变量不需要每次重新填写。
    
- 常用 CMake Target 的运行配置只需创建一次。
    
- 调试同一个可执行程序关联的多个 cpp 文件时，只需要切换断点，不需要切换运行配置。