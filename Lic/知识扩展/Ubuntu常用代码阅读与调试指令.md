
# Ubuntu / ROS 常用代码阅读与调试命令

## 1. 文件查找

```bash
# 查找指定文件
find . -name "文件名"

# 查找所有 C++ 文件
find . -type f \( -name "*.cpp" -o -name "*.h" -o -name "*.hpp" \)

# 查找所有 Python 文件
find . -name "*.py"

# 查找指定目录下的文件
find ./src -type f

# 查找指定目录下的可执行文件
find . -type f -executable
```

## 2. 代码搜索（grep / rg）

```bash
# 递归搜索关键词，显示文件名和行号
grep -rn "关键词" .

# 只搜索 C++ 文件
grep -rn "关键词" . --include="*.cpp" --include="*.h"

# 只搜索 Python 文件
grep -rn "关键词" . --include="*.py"

# 忽略大小写搜索
grep -rin "关键词" .

# 搜索函数定义或调用
grep -rn "函数名" . --include="*.cpp" --include="*.h"

# 搜索 C++ main 函数
grep -rn "int main" . --include="*.cpp"

# 搜索 Python 程序入口
grep -rn "__main__" . --include="*.py"

# 排除 build、devel 等目录
grep -rn "关键词" . --exclude-dir=build --exclude-dir=devel

# 使用 ripgrep（推荐，速度更快）
rg "关键词"

# 只搜索 C++ 文件
rg "关键词" -g '*.cpp' -g '*.h'

# 只搜索 Python 文件
rg "关键词" -g '*.py'
```

## 3. 代码查看与提取

```bash
# 查看整个文件
cat main.cpp

# 查看文件前 50 行
head -n 50 main.cpp

# 查看文件最后 50 行
tail -n 50 main.cpp

# 查看指定行范围（例如 100~200 行）
sed -n '100,200p' main.cpp

# 查看代码并显示行号
nl -ba main.cpp

# 查看指定行范围并显示行号
nl -ba main.cpp | sed -n '100,200p'

# 分页查看文件
less main.cpp

# 实时查看日志
tail -f log.txt
```

less 常用快捷键：

- `/关键词`：搜索关键词
- `n`：下一个匹配
- `N`：上一个匹配
- `空格`：下一页
- `b`：上一页
- `g`：跳到文件开头
- `G`：跳到文件末尾
- `q`：退出

## 4. 函数调用关系追踪

```bash
# 搜索函数定义及调用位置
rg "函数名" . -g '*.cpp' -g '*.h'

# 搜索类定义
rg "class 类名" . -g '*.cpp' -g '*.h'

# 搜索结构体定义
rg "struct 结构体名" . -g '*.cpp' -g '*.h'

# 搜索变量使用位置
rg "变量名" .

# 搜索 C++ main 函数
rg "int main" -g '*.cpp'

# 搜索 ROS Topic 发布
rg "advertise" -g '*.cpp'

# 搜索 ROS Topic 订阅
rg "subscribe" -g '*.cpp'

# 搜索 ROS 参数读取
rg "getParam|param\\(" -g '*.cpp'

# 搜索 Python Publisher 和 Subscriber
rg "Publisher|Subscriber" -g '*.py'

# 搜索 Python 参数读取
rg "get_param" -g '*.py'
```

## 5. C++ 代码调试（GDB）

```bash
# 使用 GDB 启动程序
gdb ./程序名

# 设置断点
break main

# 在指定文件行设置断点
break main.cpp:100

# 在指定函数设置断点
break 函数名

# 启动程序
run

# 带参数启动
run 参数1 参数2

# 单步执行，不进入函数
next

# 单步执行，进入函数
step

# 继续运行
continue

# 查看变量
print 变量名

# 修改变量值
set var 变量名=数值

# 查看局部变量
info locals

# 查看当前函数参数
info args

# 查看函数调用栈
backtrace

# 查看当前代码
list

# 查看所有断点
info breakpoints

# 删除断点
delete 断点编号

# 查看线程
info threads

# 切换线程
thread 线程编号

# 退出
quit
```

C++ 编译时建议开启 Debug 模式，保留调试符号：

```bash
catkin_make -DCMAKE_BUILD_TYPE=Debug
```

## 6. Python 代码调试

代码中插入断点：

```python
import pdb
pdb.set_trace()
```

常用调试命令：

```text
n       下一步，不进入函数
s       进入函数
c       继续运行
p x     打印变量 x
pp x    格式化打印变量 x
l       查看当前代码
bt      查看调用栈
w       查看当前执行位置
q       退出调试
```

Python 命令行调试：

```bash
# 使用 pdb 运行 Python 程序
python3 -m pdb main.py

# 语法检查
python3 -m py_compile main.py

# 查看 Python 版本
python3 --version
```

## 7. ROS1 节点调试

```bash
# 查看运行中的节点
rosnode list

# 查看节点详细信息
rosnode info /节点名称

# 查看节点是否正常运行
rosnode ping /节点名称

# 查看节点进程
ps aux | grep 节点名称

# 查看节点所在机器
rosnode machine /节点名称

# 结束指定节点
rosnode kill /节点名称
```

## 8. ROS1 Topic 调试

```bash
# 查看所有 Topic
rostopic list

# 查看 Topic 信息
rostopic info /话题名称

# 实时查看消息
rostopic echo /话题名称

# 查看 Topic 发布频率
rostopic hz /话题名称

# 查看消息带宽
rostopic bw /话题名称

# 查看消息类型
rostopic type /话题名称

# 查看消息结构
rosmsg show 消息类型

# 发布测试消息
rostopic pub /话题名称 消息类型 消息内容
```

## 9. ROS1 参数调试

```bash
# 查看所有参数
rosparam list

# 查看指定参数
rosparam get /参数名称

# 设置参数
rosparam set /参数名称 参数值

# 删除参数
rosparam delete /参数名称

# 保存参数
rosparam dump params.yaml

# 加载参数
rosparam load params.yaml
```

## 10. ROS1 图形化调试

```bash
# 查看节点与 Topic 通信关系
rqt_graph

# 查看 TF 坐标关系图
rosrun tf view_frames

# 实时查看两个坐标系之间的 TF
rosrun tf tf_echo base_link map

# 查看 TF 树
rosrun tf tf_monitor

# 启动 RViz
rviz

# 查看 ROS 日志
rqt_console
```

## 11. ROS 功能包与编译

```bash
# 查找 ROS 功能包路径
rospack find 功能包名

# 查看功能包依赖
rospack depends 功能包名

# 查看功能包列表
rospack list

# 编译整个工作空间
catkin_make

# 编译指定功能包
catkin_make --pkg 功能包名

# Debug 模式编译
catkin_make -DCMAKE_BUILD_TYPE=Debug

# 清理编译文件
catkin_make clean

# 重新加载工作空间环境
source devel/setup.bash

# 启动 Launch 文件
roslaunch 功能包名 文件名.launch

# 查看 Launch 文件参数
roslaunch 功能包名 文件名.launch --args

# 查看 CMake 编译目标
grep -n "add_executable\\|add_library" CMakeLists.txt
```

## 12. C++ 编译与程序检查

```bash
# 编译单个 C++ 文件
g++ main.cpp -o main

# 编译并开启调试信息
g++ -g main.cpp -o main

# 指定 C++ 标准
g++ -std=c++14 main.cpp -o main

# 查看可执行文件类型
file ./main

# 查看程序依赖的动态库
ldd ./main

# 查看程序符号
nm -C ./main

# 查看程序链接的动态库
ldd ./main | grep "not found"
```

## 13. 进程与系统调试

```bash
# 查看进程
ps aux | grep 进程名

# 实时查看系统资源
htop

# 查看 CPU 和内存占用
top

# 查看进程打开的文件
lsof -p PID

# 查看程序监听的端口
ss -lntp

# 结束进程
kill PID

# 强制结束进程
kill -9 PID

# 实时查看日志
tail -f 日志文件
```

## 14. Git 代码追踪

```bash
# 查看工作区修改状态
git status

# 查看代码修改内容
git diff

# 查看暂存区修改
git diff --cached

# 查看最近 10 次提交
git log --oneline -10

# 查看某个文件的修改历史
git log --oneline 文件名

# 查看每一行代码的修改记录
git blame 文件名

# 查看某次提交的详细修改
git show 提交ID

# 查看当前分支
git branch

# 查看所有分支
git branch -a
```

---

## 15. 日常最常用命令速查

| 需求 | 命令 |
|---|---|
| 查找文件 | `find . -name "文件名"` |
| 搜索代码 | `rg "关键词"` |
| 搜索函数 | `rg "函数名" -g '*.cpp' -g '*.h'` |
| 搜索变量 | `rg "变量名" .` |
| 搜索 main | `rg "int main" -g '*.cpp'` |
| 查看指定代码 | `sed -n '100,200p' 文件名` |
| 显示行号 | `nl -ba 文件名` |
| 查看函数调用线索 | `rg "函数名" .` |
| 查看 ROS 节点 | `rosnode list` |
| 查看 ROS 话题 | `rostopic list` |
| 查看话题数据 | `rostopic echo /话题名` |
| 查看话题频率 | `rostopic hz /话题名` |
| 查看节点通信图 | `rqt_graph` |
| 查看 TF | `rosrun tf tf_echo base_link map` |
| C++ 断点调试 | `gdb ./程序名` |
| Python 断点调试 | `python3 -m pdb main.py` |
| 查看代码修改 | `git diff` |

## 16. 推荐日常工作流程

阅读一个陌生 ROS 项目时：

1. `rospack find 功能包名`：找到功能包路径。
2. `find . -type f`：了解项目文件结构。
3. `rg "main"`：找到程序入口。
4. `rg "函数名"`：追踪函数定义和调用。
5. `sed -n '起始行,结束行p' 文件名`：提取关键代码。
6. `cat CMakeLists.txt`：了解程序编译关系。
7. `rqt_graph`：分析 ROS 节点通信。
8. `rostopic echo`：检查消息数据。
9. `GDB / PDB`：定位程序运行问题。
10. `git diff`：检查代码修改。

**推荐工具：**

- ripgrep：快速搜索代码。
- GDB：C++ 断点调试。
- PDB：Python 断点调试。
- RViz：机器人状态及规划结果可视化。
- rqt_graph：ROS 节点通信关系分析。
- Git：代码版本管理。
