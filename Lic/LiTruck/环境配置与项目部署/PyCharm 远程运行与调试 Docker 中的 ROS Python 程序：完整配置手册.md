
> 适用环境：Ubuntu 20.04 / ROS Noetic / Docker / PyCharm Professional 2024.1.7  
> 项目路径：`/home/ldd/mult_truck-main`  
> Docker 容器：`ros1`  
> 目标：在宿主机 PyCharm 中，直接运行和断点调试 Docker 容器内的 ROS Python 程序，不需要为每个 Python 文件单独创建配置。

---

# 1. 最终实现效果

完成配置后，可以在 PyCharm 中实现：

1. **通用运行**：对当前打开的任意 Python 文件，使用同一个 Run 配置运行。
    
2. **通用断点调试**：对当前打开的任意 Python 文件，使用同一个 Debug 配置运行，并在 PyCharm 中命中断点。
    
3. **自动加载 ROS 环境**：启动业务程序前，自动加载 ROS Noetic 和 catkin 工作空间环境。
    
4. **自动检查 ROS Master**：如果容器中的 `roscore` 尚未启动，启动器会尝试启动它。
    
5. **查看调试信息**：在 PyCharm 中查看变量、调用栈，进行单步执行和函数跟踪。
    

整体运行逻辑：

```
PyCharm（Ubuntu 宿主机）
        |
        | 运行 run_in_ros.py
        v
宿主机 Python 启动器
        |
        | docker exec
        v
Docker 容器 ros1
        |
        | source ROS Noetic
        | source catkin 工作空间
        v
容器内 Python 业务程序
        |
        +---- 普通运行：直接执行 Python 文件
        |
        +---- 断点调试：pydevd_pycharm 连接宿主机 5678 端口
                                      |
                                      v
                              PyCharm Debug Server
                                      |
                                      v
                               断点 / 变量 / 调用栈
```

**重要：** 本方案不是在宿主机直接运行 ROS 业务代码，而是由宿主机启动器进入 Docker，在容器内部执行真正的 ROS Python 程序。

---

# 2. 环境与路径配置

本项目使用的环境如下：

|项目|配置|
|---|---|
|宿主机项目路径|`/home/ldd/mult_truck-main`|
|Docker 容器名称|`ros1`|
|Docker 网络模式|`host`|
|ROS 发行版|Noetic|
|ROS 环境脚本|`/opt/ros/noetic/setup.bash`|
|Catkin 环境脚本|`/home/ldd/mult_truck-main/devel/setup.bash`|
|PyCharm 版本|Professional 2024.1.7|
|PyCharm Build|241.19416.19|
|远程调试端口|`5678`|
|容器内 Python|`python3`|
|宿主机启动器 Python|`/home/ldd/mult_truck-main/venv/bin/python`|

## 2.1 宿主机和容器路径

本方案假设宿主机与 Docker 容器内的项目路径相同：

```
宿主机：
/home/ldd/mult_truck-main

Docker 容器：
/home/ldd/mult_truck-main
```

容器必须能够访问宿主机项目目录对应的文件。

检查容器内文件：

```
docker exec ros1 test -f \
/home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py \
&& echo "容器内文件存在"
```

如果容器内找不到文件，需要检查 Docker 的目录挂载配置。

注意：PyCharm 的路径映射不能代替 Docker 的文件挂载。

## 2.2 Docker 网络模式

本方案使用 Docker 的 `host` 网络模式。

由于远程调试需要容器内的 Python 程序连接宿主机 PyCharm 的调试端口，因此需要保证容器能够访问：

```
127.0.0.1:5678
```

---

# 3. 安装远程调试依赖

远程调试客户端运行在 Docker 容器内部，因此需要在容器内安装与 PyCharm 版本匹配的调试包。

## 3.1 进入容器

宿主机终端执行：

```
docker exec -it ros1 bash
```

## 3.2 安装调试包

容器内执行：

```
python3 -m pip install "pydevd-pycharm~=241.19416.19"
```

这里的版本号对应当前 PyCharm Build：

```
241.19416.19
```

如果以后升级 PyCharm，需要重新确认匹配的 `pydevd-pycharm` 版本。

## 3.3 验证安装

宿主机终端执行：

```
docker exec ros1 python3 -c \
"import pydevd_pycharm; print('PyCharm Debug SUCCESS')"
```

如果输出：

```
PyCharm Debug SUCCESS
```

说明安装成功。

---

# 4. 创建通用启动器 run_in_ros.py

在宿主机项目根目录创建：

```
/home/ldd/mult_truck-main/run_in_ros.py
```

该文件负责：

1. 获取 PyCharm 当前需要运行的 Python 文件。
    
2. 检查宿主机文件是否存在。
    
3. 检查 Docker 容器是否运行。
    
4. 检查容器内是否存在对应文件。
    
5. 检查 ROS Master 是否启动。
    
6. 如果 ROS Master 未启动，尝试自动启动。
    
7. 加载 ROS Noetic 和 catkin 工作空间环境。
    
8. 根据参数选择普通运行或远程断点调试。
    

## 4.1 完整代码

```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import sys
import subprocess
import signal

CONTAINER = "ros1"

WORKSPACE = "/home/ldd/mult_truck-main"

ROS_SETUP = "/opt/ros/noetic/setup.bash"

CATKIN_SETUP = WORKSPACE + "/devel/setup.bash"


def main():

    # 获取 PyCharm 传入的当前 Python 文件
    if len(sys.argv) < 2:
        print("错误：没有获取到当前 Python 文件路径")
        sys.exit(1)

    python_file = os.path.abspath(sys.argv[1])
    debug_mode = "--debug" in sys.argv

    if debug_mode:
        print("当前模式：PyCharm 远程断点调试")
    else:
        print("当前模式：普通运行")

    print("=" * 50)
    print("PyCharm Docker ROS 通用运行器")
    print("运行文件：", python_file)
    print("=" * 50)

    # 检查宿主机文件
    if not os.path.isfile(python_file):
        print("错误：Python 文件不存在")
        sys.exit(1)

    # 检查 Docker 容器
    check = subprocess.run(
        ["docker", "inspect", "-f",
         "{{.State.Running}}", CONTAINER],
        capture_output=True,
        text=True
    )

    if check.returncode != 0 or check.stdout.strip() != "true":
        print("错误：Docker 容器未运行")
        sys.exit(1)

    # 检查容器内文件
    check = subprocess.run(
        ["docker", "exec", CONTAINER,
         "test", "-f", python_file]
    )

    if check.returncode != 0:
        print("错误：容器内找不到 Python 文件")
        sys.exit(1)

    # 自动启动 ROS Master
    master_check = subprocess.run(
        [
            "docker", "exec", CONTAINER,
            "bash", "-c",
            "timeout 2 bash -c '</dev/tcp/127.0.0.1/11311' "
            "2>/dev/null"
        ]
    )

    if master_check.returncode != 0:

        print("ROS Master 未启动，正在启动...")

        subprocess.Popen(
            [
                "docker", "exec", "-d", CONTAINER,
                "bash", "-c",
                "source /opt/ros/noetic/setup.bash && "
                "nohup roscore >/tmp/roscore.log 2>&1"
                " </dev/null &"
            ]
        )

        import time

        for _ in range(20):

            time.sleep(1)

            result = subprocess.run(
                [
                    "docker", "exec", CONTAINER,
                    "bash", "-c",
                    "timeout 1 bash -c "
                    "'</dev/tcp/127.0.0.1/11311' "
                    "2>/dev/null"
                ]
            )

            if result.returncode == 0:
                break
        else:
            print("错误：ROS Master 启动失败")
            sys.exit(1)

    print("ROS Master 连接成功")
    print("开始运行 Python 程序", flush=True)

    # 根据运行模式选择执行命令
    if debug_mode:

        print("正在启动远程调试客户端...")

        python_command = (
            "exec python3 -u -c '"
            "import sys, runpy, pydevd_pycharm; "
            "script=sys.argv[1]; "
            "sys.argv=[script]; "
            "pydevd_pycharm.settrace("
            "\"127.0.0.1\", "
            "port=5678, "
            "stdoutToServer=True, "
            "stderrToServer=True, "
            "suspend=True"
            "); "
            "runpy.run_path(script, run_name=\"__main__\")"
            "' \"$1\""
        )

    else:

        python_command = 'exec python3 -u "$1"'

    command = [
        "docker", "exec", "-i",
        CONTAINER,
        "bash", "-c",

        "source /opt/ros/noetic/setup.bash && "
        "source /home/ldd/mult_truck-main/devel/setup.bash && "
        "export PYTHONUNBUFFERED=1 && "
        + python_command,

        "run_in_ros",
        python_file
    ]

    process = subprocess.Popen(command)

    try:
        process.wait()

    except KeyboardInterrupt:

        print("\n正在停止 Python 程序...")

        process.send_signal(signal.SIGINT)

        try:
            process.wait(timeout=5)
        except subprocess.TimeoutExpired:
            process.terminate()
            process.wait()

    sys.exit(process.returncode or 0)


if __name__ == "__main__":
    main()
```

## 4.2 代码功能说明

|代码|功能|
|---|---|
|`sys.argv[1]`|获取当前需要运行的 Python 文件|
|`os.path.isfile()`|检查宿主机文件是否存在|
|`docker inspect`|检查容器是否运行|
|`docker exec test -f`|检查容器内文件是否存在|
|端口 `11311` 检查|判断 ROS Master 是否启动|
|`roscore`|ROS Master 未启动时尝试自动启动|
|`source /opt/ros/noetic/setup.bash`|加载 ROS Noetic 环境|
|`source .../devel/setup.bash`|加载 catkin 工作空间环境|
|`python3 -u`|无缓冲方式运行 Python 程序|
|`pydevd_pycharm.settrace()`|建立远程调试连接|
|`runpy.run_path()`|执行指定 Python 文件|
|`process.wait()`|等待业务程序结束|
|`KeyboardInterrupt`|处理用户中断操作|

## 4.3 普通运行与断点调试的区别

普通运行时，启动器在容器中执行：

```
python3 -u "$1"
```

断点调试时，启动器在容器中执行：

```
import sys
import runpy
import pydevd_pycharm

pydevd_pycharm.settrace(
    "127.0.0.1",
    port=5678,
    stdoutToServer=True,
    stderrToServer=True,
    suspend=True
)

runpy.run_path(script, run_name="__main__")
```

关键参数说明：

|参数|作用|
|---|---|
|`127.0.0.1`|调试连接地址|
|`5678`|PyCharm Debug Server 监听端口|
|`stdoutToServer=True`|将标准输出转发到调试器|
|`stderrToServer=True`|将错误输出转发到调试器|
|`suspend=True`|连接成功后先暂停程序|
|`runpy.run_path()`|执行指定 Python 文件|

注意：当前启动器将业务脚本的 `sys.argv` 设置为仅包含脚本路径。如果以后业务脚本需要接收额外命令行参数，需要扩展启动器的参数解析和转发逻辑。

---

# 5. PyCharm 运行配置总览

最终只需要维护一个启动器：

```
/home/ldd/mult_truck-main/run_in_ros.py
```

PyCharm 中创建三个配置：

|配置名称|类型|作用|
|---|---|---|
|ROS Docker Remote Debug Server|Python Debug Server|监听 5678，接收容器调试连接|
|ROS Docker 通用运行|Python|普通运行当前 Python 文件|
|ROS Docker 通用调试|Python|远程断点调试当前 Python 文件|

其中，普通运行配置沿用此前已经使用的配置，不需要重新创建另一套启动脚本。

---

# 6. 原有普通运行配置

在增加远程断点调试之前，项目已经有一套普通运行流程。

其核心逻辑是：

```
PyCharm
   ↓
宿主机 Python
   ↓
run_in_ros.py
   ↓
docker exec
   ↓
Docker 容器 ros1
   ↓
加载 ROS 环境
   ↓
执行目标 Python 文件
```

## 6.1 原有普通运行配置参数

进入：

```
Run → Edit Configurations...
```

选择原有 Python 配置，确认参数如下：

|配置项|设置值|
|---|---|
|Name|`ROS Docker 通用运行`|
|类型|Python|
|Python interpreter|宿主机项目虚拟环境|
|Script path|`/home/ldd/mult_truck-main/run_in_ros.py`|
|Parameters|`$FilePath$`|
|Working directory|`/home/ldd/mult_truck-main`|

解释器选择：

```
/home/ldd/mult_truck-main/venv/bin/python
```

此前宿主机虚拟环境使用 Python 3.12。

这里需要区分：

- 宿主机 Python：负责运行 `run_in_ros.py`。
    
- 容器内 Python：负责真正执行 ROS 业务代码。
    
- 容器内 ROS Noetic 使用自己的 Python 环境，不会因为宿主机虚拟环境是 Python 3.12 就变成 Python 3.12。
    

## 6.2 原有普通运行参数

核心参数：

```
$FilePath$
```

`$FilePath$` 是 PyCharm 的宏，会自动替换为当前打开 Python 文件的绝对路径。

例如当前打开：

```
/home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py
```

PyCharm 实际调用的命令类似：

```
/home/ldd/mult_truck-main/venv/bin/python \
/home/ldd/mult_truck-main/run_in_ros.py \
/home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py
```

## 6.3 普通运行方式

1. 确认 Docker 容器 `ros1` 已经启动。
    
2. PyCharm 打开需要运行的 Python 文件。
    
3. 右上角选择 `ROS Docker 通用运行`。
    
4. 点击绿色三角形。
    
5. 查看 Run 控制台。
    

不需要为每个 Python 文件创建单独的配置。

---

# 7. PyCharm 远程 Debug Server 配置

远程 Debug Server 的作用是：

在宿主机 PyCharm 中监听一个端口，等待 Docker 容器内的 Python 调试客户端连接。

## 7.1 新建配置

打开：

```
Run → Edit Configurations...
```

点击左上角 `+`，选择：

```
Python Debug Server
```

## 7.2 配置参数

|配置项|设置值|
|---|---|
|Name|`ROS Docker Remote Debug Server`|
|IDE host name|`127.0.0.1`|
|Port|`5678`|

路径映射设置：

|Local path|Remote path|
|---|---|
|`/home/ldd/mult_truck-main`|`/home/ldd/mult_truck-main`|

如果界面中有 `Suspend after connect` / `连接后挂起` 选项，可以保持当前已经验证成功的配置。

真正控制远程 Python 客户端连接后是否暂停的关键参数是启动器中的：

```
suspend=True
```

## 7.3 启动 Debug Server

保存配置后，在 PyCharm 右上角选择：

```
ROS Docker Remote Debug Server
```

点击绿色三角形运行。

启动成功后，控制台应出现类似：

```
Starting debug server at port 5,678

Use the following code to connect to the debugger:

import pydevd_pycharm
pydevd_pycharm.settrace(
    '127.0.0.1',
    port=5678,
    stdoutToServer=True,
    stderrToServer=True
)

Waiting for process connection…
```

此时 PyCharm 正在监听 5678 端口。

## 7.4 检查监听端口

Ubuntu 宿主机终端执行：

```
ss -lntp | grep 5678
```

如果看到类似：

```
LISTEN 0 50 *:5678 *:* users:(("java",pid=3229,fd=231))
```

说明 PyCharm 已经监听 5678 端口。

---

# 8. 创建通用断点调试配置

断点调试配置与普通运行配置基本相同，只需要多传递一个 `--debug` 参数。

## 8.1 新建配置

进入：

```
Run → Edit Configurations...
```

点击 `+`，选择：

```
Python
```

## 8.2 配置参数

|配置项|设置值|
|---|---|
|Name|`ROS Docker 通用调试`|
|Python interpreter|`/home/ldd/mult_truck-main/venv/bin/python`|
|Script path|`/home/ldd/mult_truck-main/run_in_ros.py`|
|Parameters|`$FilePath$ --debug`|
|Working directory|`/home/ldd/mult_truck-main`|

与普通运行配置相比，关键差异只有：

```
普通运行：
$FilePath$

断点调试：
$FilePath$ --debug
```

启动器检测到 `--debug` 后，就会在 Docker 容器内启动远程调试客户端，连接宿主机的 5678 端口。

## 8.3 推荐的配置分工

- `ROS Docker Remote Debug Server`：使用绿色三角形启动，保持监听。
    
- `ROS Docker 通用调试`：使用绿色三角形启动启动器，让容器内程序主动连接 Debug Server。
    

这样可以避免将外层启动器自身的 PyCharm Debug 会话，与容器内业务程序的远程调试会话混淆。

注意：宿主机启动器的外层调试端口，与容器内业务程序使用的 `5678` 端口不是一回事。

---

# 9. 日常运行步骤

## 9.1 普通运行

1. 确认 Docker 容器 `ros1` 正在运行。
    
2. 打开 PyCharm。
    
3. 打开需要运行的 Python 文件。
    
4. 选择 `ROS Docker 通用运行`。
    
5. 点击绿色三角形。
    
6. 查看 Run 控制台输出。
    

启动器会自动检查容器、检查 ROS Master、加载 ROS 环境并执行当前文件。

例如运行：

```
/home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py
```

控制台可能显示：

```
当前模式：普通运行
==================================================
PyCharm Docker ROS 通用运行器
运行文件：/home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py
==================================================
ROS Master 连接成功
开始运行 Python 程序
```

## 9.2 断点调试

按照以下顺序操作。

### 第一步：启动远程 Debug Server

选择：

```
ROS Docker Remote Debug Server
```

点击绿色三角形，确认控制台显示：

```
Waiting for process connection…
```

### 第二步：设置代码断点

打开需要调试的 Python 文件。

例如：

```
def main():

    debug_a = 100
    debug_b = 200

    debug_c = debug_a + debug_b  # 在这里设置断点

    print(debug_c)
```

在左侧行号栏点击可执行代码所在行。

建议将断点设置在：

- 变量赋值语句。
    
- 函数调用语句。
    
- 条件判断语句。
    
- 数据处理语句。
    

不要只在空行、注释或函数定义行上设置断点。

### 第三步：启动业务程序

选择：

```
ROS Docker 通用调试
```

点击绿色三角形。

### 第四步：确认远程连接

启动器输出类似：

```
当前模式：PyCharm 远程断点调试
ROS Master 连接成功
开始运行 Python 程序
正在启动远程调试客户端...
```

Debug Server 控制台连接成功后，可能显示：

```
Connected to pydev debugger (build 241.19416.19)
```

### 第五步：查看断点

当程序执行到断点所在行时，PyCharm 会暂停，并显示当前执行位置。

此时可以查看：

- Variables：变量窗口。
    
- Frames / Call Stack：调用栈。
    
- Console：程序输出。
    
- Debugger：线程、暂停状态和调试控制按钮。
    

如果断点所在函数尚未执行，程序不会停在那里。例如，某个 ROS 回调函数只有收到对应话题消息后才会执行。

---

# 10. 断点调试常用操作

## 10.1 常用快捷键

|操作|快捷键|作用|
|---|---|---|
|Resume Program|F9|继续执行，直到下一个断点|
|Step Over|F8|执行当前行，不进入被调用函数|
|Step Into|F7|进入当前行调用的函数内部|
|Step Out|Shift+F8|执行完当前函数并返回上一层|
|Stop|Ctrl+F2|停止当前调试任务|
|Evaluate Expression|Alt+F8|计算表达式、检查变量|

快捷键可能受操作系统和 PyCharm Keymap 设置影响，也可以直接点击 Debug 工具窗口中的按钮。

## 10.2 查看变量

例如：

```
target_x = 12.5
target_y = 8.6

vehicle_x = 10.0
vehicle_y = 5.0

dx = target_x - vehicle_x
dy = target_y - vehicle_y
```

在 `dx` 赋值处设置断点，暂停后可以在 Variables 窗口查看：

```
target_x = 12.5
target_y = 8.6
vehicle_x = 10.0
vehicle_y = 5.0
```

这样就能判断程序实际使用的数据是否正确。

## 10.3 单步执行

例如：

```
def main():
    init_map()
    receive_target()
    plan_path()
    send_command()
```

如果在 `init_map()` 处暂停：

- 按 F8：执行 `init_map()`，不进入函数内部。
    
- 按 F7：进入 `init_map()` 内部逐行查看。
    
- 按 F9：继续运行到下一个断点。
    

## 10.4 查看调用栈

调用栈记录当前程序是通过哪些函数调用到暂停位置的。

例如：

```
calculate()
process()
main()
<module>
```

表示：

```
程序入口
  ↓
main()
  ↓
process()
  ↓
calculate()
  ↓
当前暂停位置
```

点击不同的栈帧，可以查看对应函数的局部变量。

## 10.5 检查条件判断

例如：

```
if obstacle_detected and ttc < 2.0:
    stop_vehicle()
```

在 `if` 行设置断点，暂停后查看：

```
obstacle_detected
ttc
```

就能分析程序为什么进入停车分支。

## 10.6 表达式求值

程序暂停时按：

```
Alt + F8
```

可以输入：

```
target_x - vehicle_x
```

或者：

```
len(path)
```

查看表达式的计算结果。

注意：表达式求值也可能执行函数或改变程序状态。初学时建议只查看变量和计算简单表达式，不要随意调用发送控制指令、修改车辆状态的函数。

---

# 11. 结合多矿卡项目的调试建议

ROS 项目中，很多数据通过 Subscriber 回调函数更新，因此调试时要关注消息接收、坐标转换、规划和决策等关键位置。

|模块|推荐断点位置|主要检查内容|
|---|---|---|
|ROS 初始化|`rospy.init_node()` 附近|节点是否初始化成功|
|话题订阅|Subscriber 对应的回调函数|是否收到消息|
|目标点接收|目标点回调函数|目标点坐标、消息内容|
|GNSS/INS|经纬度转换、ENU 转换函数|定位坐标是否正确|
|TF|坐标变换函数|坐标系、变换结果|
|地图模块|地图加载和解析函数|地图是否正常加载|
|全局规划|A* 规划函数|起点、终点、路径结果|
|局部规划|TEB 规划相关函数|局部轨迹、速度和障碍物约束|
|障碍物检测|YOLO、毫米波雷达处理函数|检测结果、目标位置|
|决策模块|停车、启动、转向判断|条件变量、分支执行情况|
|控制模块|PID、CAN 指令发送函数|控制量和发送指令|

## 11.1 以目标点回调函数为例

假设代码如下：

```
rospy.Subscriber(
    "/target_point",
    PoseStamped,
    self.target_callback
)

def target_callback(self, msg):

    self.target_x = msg.pose.position.x
    self.target_y = msg.pose.position.y
```

如果想知道目标点有没有收到，应该在 `target_callback()` 内设置断点。

暂停后查看：

```
msg
self.target_x
self.target_y
```

如果断点一直没有命中，需要检查：

- 目标点话题是否存在。
    
- 是否有节点发布消息。
    
- 订阅的话题名称和消息类型是否一致。
    
- 回调函数是否被正确注册。
    
- 目标点消息是否真正到达。
    

## 11.2 ROS 程序调试注意事项

1. ROS 回调函数可能在独立线程中执行，断点暂停可能影响其他线程和回调。
    
2. 断点会改变程序运行时序，不适合直接用来评价实时性和控制周期。
    
3. 调试车辆控制、CAN 发送、自动启停等功能时，优先在仿真环境或安全测试环境中进行。
    
4. 对于频繁触发的回调函数，不建议长期在高频代码中设置大量断点。
    
5. 对于路径、点云、图像等大型数据，可以先检查数据尺寸、关键字段和少量样本。
    

---

# 12. 常见问题与排查方法

## 12.1 Docker 容器未运行

检查：

```
docker ps -a
```

如果容器已经停止，先启动：

```
docker start ros1
```

再运行 PyCharm 配置。

## 12.2 容器内找不到 Python 文件

检查宿主机文件：

```
ls -l /home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py
```

检查容器内文件：

```
docker exec ros1 ls -l \
/home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py
```

如果宿主机存在、容器内不存在，检查 Docker 项目目录挂载。

## 12.3 ROS Master 连接失败

检查容器中是否有 `roscore`：

```
docker exec ros1 bash -c \
"ps aux | grep '[r]oscore'"
```

检查 11311 端口：

```
docker exec ros1 bash -c \
"timeout 2 bash -c '</dev/tcp/127.0.0.1/11311' \
&& echo ROS_MASTER_OK"
```

查看启动日志：

```
docker exec ros1 tail -n 100 /tmp/roscore.log
```

如果 Master 没有启动成功，需要检查 ROS 安装和环境脚本是否存在。

## 12.4 无法导入 pydevd_pycharm

检查：

```
docker exec ros1 python3 -c \
"import pydevd_pycharm; print('OK')"
```

如果失败，重新在容器内安装与当前 PyCharm Build 匹配的调试包。

## 12.5 Debug Server 没有收到连接

按顺序检查：

1. PyCharm 的 `ROS Docker Remote Debug Server` 是否已经启动。
    
2. Debug Server 是否监听 `5678`。
    
3. 容器是否能访问宿主机的 `127.0.0.1:5678`。
    
4. 启动器是否选择了 `ROS Docker 通用调试`。
    
5. 参数是否为 `$FilePath$ --debug`。
    
6. 容器内是否安装了 `pydevd_pycharm`。
    

宿主机检查端口：

```
ss -lntp | grep 5678
```

容器检查端口连通性：

```
docker exec ros1 bash -c \
"timeout 2 bash -c '</dev/tcp/127.0.0.1/5678' \
&& echo DEBUG_SERVER_OK"
```

## 12.6 Debug Server 显示 Connected，但断点没有命中

检查：

- 断点是否设置在真正会执行的代码行。
    
- 程序是否已经执行到该函数。
    
- 当前运行的是否是正在编辑的那个文件。
    
- 断点是否被禁用。
    
- 程序是否已经从断点所在位置执行过去。
    

检查宿主机与容器中的文件是否一致：

```
md5sum \
/home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py

docker exec ros1 md5sum \
/home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py
```

两边 MD5 应一致。

也可以检查断点附近的实际行号：

```
nl -ba /home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py \
| sed -n '760,780p'

docker exec ros1 bash -c \
"nl -ba /home/ldd/mult_truck-main/src/decision/scripts/truck_decision.py \
| sed -n '760,780p'"
```

路径映射应为：

```
/home/ldd/mult_truck-main
        ↕
/home/ldd/mult_truck-main
```

如果只是 Python 标准库 `runpy.py` 等路径没有映射，一般不影响项目业务脚本的断点映射。

## 12.7 为什么调用栈中会出现 main、<module>、runpy？

例如：

```
main, truck_decision.py:767
<module>, truck_decision.py:1058
_run_code, runpy.py:87
_run_module_code, runpy.py:97
run_path, runpy.py:265
<module>, <string>:1
```

这是因为启动器使用 `runpy.run_path()` 执行业务脚本，所以调用栈中会包含 `runpy` 和 `<string>` 等启动层函数。

这不代表业务程序出错。

只要能够在业务脚本的断点处暂停，并查看变量和调用栈，就说明远程调试链路正常。

## 12.8 为什么停止程序时显示退出码 254？

如果用户按下停止或中断，启动器会尝试向 Docker 执行进程发送 `SIGINT`。

在某些情况下，进程退出码可能表现为 `254`，这可能是中断信号对应的负返回码转换结果，并不一定意味着程序本身发生了 Python 异常。

应结合控制台是否出现：

```
正在停止 Python 程序...
```

以及程序是否由用户主动停止来判断。

---

# 13. 日常使用检查清单

## 普通运行

- Docker 容器 `ros1` 正在运行。
    
- PyCharm 打开了要运行的 Python 文件。
    
- 选择 `ROS Docker 通用运行`。
    
- 点击绿色三角形。
    
- 查看 Run 控制台。
    

## 断点调试

- Docker 容器 `ros1` 正在运行。
    
- 启动 `ROS Docker Remote Debug Server`。
    
- 确认 Debug Server 正在监听 `5678`。
    
- 在业务 Python 文件中设置断点。
    
- 选择 `ROS Docker 通用调试`。
    
- 确认参数为 `$FilePath$ --debug`。
    
- 点击绿色三角形启动业务程序。
    
- 确认 Debug Server 显示 `Connected to pydev debugger`。
    
- 确认程序在断点处暂停。
    
- 使用 Variables、Call Stack、F7、F8、F9 进行调试。
    

---

# 14. 最终配置汇总

## 14.1 三个 PyCharm 配置

|配置名称|类型|关键参数|
|---|---|---|
|`ROS Docker Remote Debug Server`|Python Debug Server|监听 `127.0.0.1:5678`|
|`ROS Docker 通用运行`|Python|`$FilePath$`|
|`ROS Docker 通用调试`|Python|`$FilePath$ --debug`|

## 14.2 核心文件

```
/home/ldd/mult_truck-main/run_in_ros.py
```

## 14.3 核心远程调试代码

```
pydevd_pycharm.settrace(
    "127.0.0.1",
    port=5678,
    stdoutToServer=True,
    stderrToServer=True,
    suspend=True
)
```

## 14.4 核心路径映射

```
Local:
  /home/ldd/mult_truck-main

Remote:
  /home/ldd/mult_truck-main
```

## 14.5 核心配置参数

```
普通运行：
$FilePath$

断点调试：
$FilePath$ --debug
```

**至此，Docker + ROS 的通用运行和远程断点调试配置完成。**

后续只需要打开需要运行的 Python 文件，选择对应配置即可，不必针对每个 Python 文件重复创建运行配置。