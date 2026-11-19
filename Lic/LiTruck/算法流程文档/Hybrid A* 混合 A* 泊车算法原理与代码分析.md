## 1. Hybrid A* 是什么？

Hybrid A*（混合 A*）是一种结合了 A* 搜索思想和车辆运动学模型的路径规划算法。

普通二维 A*：

- 在二维栅格地图上搜索。
    
- 每个栅格通常作为一个搜索节点。
    
- 车辆可以向相邻栅格移动。
    
- 不直接考虑车辆转向角、轴距、最小转弯半径等运动学约束。
    

Hybrid A*：

- 搜索节点表示车辆的运动学状态。
    
- 节点包含车辆的位置和航向角。
    
- 根据车辆运动学模型生成下一时刻的候选状态。
    
- 搜索过程中考虑前进、倒车、转向等约束。
    
- 通常使用二维 DP 栅格地图提供启发式代价。
    
- 使用 Reed-Shepp（RS）曲线尝试直接连接当前节点与目标点。
    

核心思想：

__利用 A_ 的搜索策略寻找车辆运动学状态，同时利用 RS 曲线尝试快速连接终点。_*

---

## 2. 首先区分三个不同层级

理解 Hybrid A*，最重要的是区分：

1. 地图栅格
    
2. 搜索节点
    
3. 轨迹采样点
    

它们不是同一个东西。

### 2.1 地图栅格

地图栅格是对二维环境进行离散化后的单元。

例如：

```
xy_grid_resolution = 0.5 m
```

表示每个栅格的边长为 0.5 m。

栅格地图主要用于：

- 表示障碍物。
    
- 生成 DP 启发式代价地图。
    
- 查询当前位置到目标区域的代价。
    
- 辅助碰撞检测或代价计算。
    

注意：

Hybrid A* 并不是简单地按照二维栅格逐格扩展车辆。

### 2.2 搜索节点 Node3d

Hybrid A* 中，一个搜索节点表示车辆的一个离散化运动学状态。

通常可以表示为：

```
Node3d = (x, y, phi)
```

其中：

- x：车辆位置 X。
    
- y：车辆位置 Y。
    
- phi：车辆航向角。
    

还可能保存：

- kappa：曲率。
    
- pre_node：父节点。
    
- xs、ys：当前运动段的轨迹采样点。
    
- phis：采样点对应的航向角。
    
- kappas：采样点对应的曲率。
    
- direction：前进或倒车方向。
    
- steering：转向输入。
    
- cost：搜索代价。
    

注意：

一个 Node3d 的搜索索引通常代表这次运动扩展的终点状态，但它内部可能保存从父节点到自身终点的整段轨迹。

因此：

**一个搜索节点不一定只保存一个坐标点，它还可能保存一整段局部运动轨迹。**

### 2.3 轨迹采样点

轨迹采样点用于描述车辆从当前节点运动到下一个节点的过程。

例如：

```
当前节点 A
    |
    | 0.2 m
    |
采样点 1
    |
    | 0.2 m
    |
采样点 2
    |
    | 0.2 m
    |
下一个节点 B
```

这些点主要用于：

- 描述局部运动轨迹。
    
- 检查车辆运动过程中是否碰撞障碍物。
    
- 保存最终输出路径所需的轨迹信息。
    

通常不会把每个轨迹采样点都放入 OPEN 队列。

真正参与搜索的是运动扩展生成的候选 Node3d。

---

## 3. 三种分辨率和步长的区别

Hybrid A* 中容易混淆三个参数。

|参数|含义|作用|
|---|---|---|
|xy_grid_resolution|二维地图栅格分辨率|地图离散化|
|状态离散分辨率|x、y、航向角的离散化精度|搜索节点索引与状态去重|
|step_size|每次运动积分的距离|控制轨迹采样间隔|

注意：

状态离散分辨率需要结合 Node3d 的 GetIndex() 和相关配置确认，不能直接认为它等于 xy_grid_resolution。

### 3.1 举例

假设：

```
xy_grid_resolution = 0.5 m
step_size = 0.2 m
```

那么：

- 地图栅格每格边长为 0.5 m。
    
- 车辆每次运动积分 0.2 m。
    
- 一次运动扩展可能包含多个轨迹采样点。
    

因此，轨迹采样间隔可以比地图栅格更小。

**采样点更密集，不代表搜索节点也按照同样的间隔增加。**

---

## 4. Next_node_generator()：普通节点扩展

这是 Hybrid A* 普通运动学扩展的核心函数。

### 4.1 确定转向角和前进/倒车方向

代码：

```
if (next_node_index < static_cast<double>(next_node_num_) / 2) {
    steering =
        -max_steer_angle_ +
        (2 * max_steer_angle_ / (static_cast<double>(next_node_num_) / 2 - 1)) *
            static_cast<double>(next_node_index);

    traveled_distance = step_size_;
} else {
    size_t index = next_node_index - next_node_num_ / 2;

    steering =
        -max_steer_angle_ +
        (2 * max_steer_angle_ / (static_cast<double>(next_node_num_) / 2 - 1)) *
            static_cast<double>(index);

    traveled_distance = -step_size_;
}
```

这里将候选运动分为两组：

第一组：

- 前进。
    
- 转向角从负最大转角到正最大转角。
    

第二组：

- 倒车。
    
- 转向角同样从负最大转角到正最大转角。
    

例如：

```
前进：
左转
小角度左转
直行
小角度右转
右转

倒车：
左转
小角度左转
直行
小角度右转
右转
```

具体候选数量由 `next_node_num_` 决定。

### 4.2 计算一次扩展的运动长度

代码：

```
double arc = std::sqrt(2) * xy_grid_resolution_;

const size_t move_steps =
    std::max<size_t>(1, static_cast<size_t>(arc / step_size_));
```

其中：

```
arc = √2 × xy_grid_resolution
```

这是栅格对角线长度。

由于：

```
static_cast<size_t>(arc / step_size_)
```

会向下取整，因此：

```
move_steps = max(1, floor(arc / step_size))
```

一次扩展的实际运动长度为：

```
L = move_steps × step_size
```

例如：

```
xy_grid_resolution = 0.5 m
step_size = 0.2 m
```

计算：

```
arc = √2 × 0.5
    = 0.7071 m

move_steps = floor(0.7071 / 0.2)
           = 3

L = 3 × 0.2
  = 0.6 m
```

因此，这次运动扩展实际走 0.6 m。

注意：

一次扩展的长度并不一定等于一个栅格的边长，也不一定等于栅格对角线。

### 4.3 计算车辆曲率

代码：

```
const double next_kappa =
    std::tan(steering) / vehicle_param_.wheel_base;
```

对应车辆运动学关系：

```
kappa = tan(steering) / wheel_base
```

其中：

- steering：前轮转向角。
    
- wheel_base：车辆轴距。
    
- kappa：车辆轨迹曲率。
    

曲率越大，车辆转弯越急。

### 4.4 生成局部轨迹采样点

代码：

```
for (size_t i = 0; i < move_steps; ++i) {
    const double next_x =
        last_x + traveled_distance * cos(last_phi);

    const double next_y =
        last_y + traveled_distance * sin(last_phi);

    const double next_phi =
        NormalizeAngle(last_phi + traveled_distance * next_kappa);

    intermediate_x.push_back(next_x);
    intermediate_y.push_back(next_y);
    intermediate_phi.push_back(next_phi);
    intermediate_kappa.push_back(next_kappa);

    last_x = next_x;
    last_y = next_y;
    last_phi = next_phi;
}
```

每次循环：

1. 根据当前航向角计算下一点的位置。
    
2. 根据车辆曲率更新航向角。
    
3. 保存当前采样点。
    
4. 更新位置和航向角，进入下一次运动积分。
    

其中：

```
采样间隔 = |traveled_distance| = step_size
```

例如：

```
step_size = 0.2 m
move_steps = 3
```

生成：

```
0.0 m：起点
0.2 m：采样点1
0.4 m：采样点2
0.6 m：终点
```

总共 4 个状态点，3 个运动间隔。

### 4.5 创建下一个搜索节点

代码：

```
std::shared_ptr<Node3d> next_node =
    std::shared_ptr<Node3d>(
        new Node3d(
            intermediate_x,
            intermediate_y,
            intermediate_phi,
            intermediate_kappa,
            XYbounds_,
            planner_open_space_config_));

next_node->SetPre(current_node);
next_node->SetDirec(traveled_distance > 0.0);
next_node->SetSteer(steering);
```

这里非常关键。

`Node3d` 接收的是一整组采样点：

```
intermediate_x
intermediate_y
intermediate_phi
intermediate_kappa
```

所以：

**一次普通扩展生成一个 Node3d，而这个 Node3d 内部保存这一次扩展的整段局部轨迹。**

同时：

```
next_node->SetPre(current_node);
```

建立父子关系。

搜索过程中可以通过父节点关系回溯整条路径。

---

## 5. Hybrid A* 的代价是怎么算的？

Hybrid A* 通常采用：

```
f(n) = g(n) + h(n)
```

其中：

- g(n)：从起点到当前节点的累计代价。
    
- h(n)：当前节点到目标点的启发式代价。
    
- f(n)：用于 OPEN 优先队列排序的总代价。
    

### 5.1 g(n)：累计实际代价

常见代价项包括：

- 行驶距离代价。
    
- 倒车惩罚。
    
- 前进和倒车切换惩罚。
    
- 转向惩罚。
    
- 转向变化惩罚。
    
- 障碍物相关代价。
    

例如：

```
g(next) =
    g(current)
    + 距离代价
    + 倒车惩罚
    + 换挡惩罚
    + 转向惩罚
    + 转向变化惩罚
```

注意：

这只是常见 Hybrid A* 代价构成，具体项目是否采用这些代价项，需要检查 `CalculateNodeCost()`。

### 5.2 h(n)：启发式代价

你的代码中：

```
grid_a_star_heuristic_generator_->GenerateDpMap(
    ex,
    ey,
    XYbounds_,
    obstacles_linesegments_vec_);
```

这里生成 DP 栅格启发式地图。

它根据：

- 目标位置。
    
- 地图边界。
    
- 障碍物。
    

计算地图中不同位置到目标区域的代价。

搜索过程中，Hybrid A* 可以查询 DP 地图，获得当前节点对应的启发式代价。

注意：

DP 地图的生成与 Hybrid A* 的运动学搜索是两个不同层级。

DP 地图主要提供 h(n)，而 Hybrid A* 通过 Next_node_generator() 生成车辆运动学状态。

因此不能简单地认为 Hybrid A* 就是二维栅格 A*。

---

## 6. Plan()：完整搜索流程

你提供的 Plan() 代码可以分为以下几个阶段。

### 6.1 初始化

```
open_set_.clear();
close_set_.clear();
open_pq_ = decltype(open_pq_)();
final_node_ = nullptr;
has_last_successful_rsp_ = false;
```

作用：

- 清空 OPEN 集合。
    
- 清空 CLOSE 集合。
    
- 清空优先队列。
    
- 清空最终节点。
    
- 重置 RS 相关状态。
    

### 6.2 创建起点和终点

```
start_node_.reset(
    new Node3d({sx}, {sy}, {sphi}, {0.0},
               XYbounds_, planner_open_space_config_));

end_node_.reset(
    new Node3d({ex}, {ey}, {ephi}, {0.0},
               XYbounds_, planner_open_space_config_));
```

创建起点和终点 Node3d。

随后：

```
if (!ValidityCheck(start_node_)) return false;
if (!ValidityCheck(end_node_)) return false;
```

检查起点和终点是否满足有效性约束。

### 6.3 生成 DP 启发式地图

```
grid_a_star_heuristic_generator_->GenerateDpMap(
    ex,
    ey,
    XYbounds_,
    obstacles_linesegments_vec_);
```

根据目标点和障碍物生成 DP 启发式地图。

### 6.4 起点进入 OPEN

```
open_set_.emplace(start_node_->GetIndex(), start_node_);

open_pq_.emplace(
    start_node_->GetIndex(),
    start_node_->GetCost());
```

起点进入 OPEN 集合和优先队列。

### 6.5 从 OPEN 中取出当前节点

```
while (!open_pq_.empty()) {
    const std::string current_id = open_pq_.top().first;

    open_pq_.pop();

    std::shared_ptr<Node3d> current_node =
        open_set_[current_id];
```

从优先队列中取出当前代价最小的节点。

### 6.6 尝试 RS 解析扩展

```
if (ShouldTryAnalyticExpansion(current_node)) {
    ++analytic_expansion_num;

    if (AnalyticExpansion(current_node)) {
        break;
    }
}
```

这里是整个算法的重要部分。

每当搜索到一个符合条件的普通搜索节点，就尝试通过 RS 曲线直接连接目标点。

RS 解析扩展的目标是：

```
当前搜索节点
      ↓
RS 曲线
      ↓
目标节点
```

如果成功：

```
AnalyticExpansion(current_node) == true
```

就直接跳出搜索循环。

如果失败：

继续普通 Hybrid A* 节点扩展。

### 6.7 普通运动学扩展

```
close_set_.emplace(
    current_node->GetIndex(),
    current_node);

for (size_t i = 0; i < next_node_num_; ++i) {
    auto next_node =
        Next_node_generator(current_node, i);

    if (next_node == nullptr) continue;

    if (close_set_.find(next_node->GetIndex())
        != close_set_.end()) continue;

    if (!ValidityCheck(next_node)) continue;

    if (open_set_.find(next_node->GetIndex())
        == open_set_.end()) {

        CalculateNodeCost(current_node, next_node);

        open_set_.emplace(
            next_node->GetIndex(), next_node);

        open_pq_.emplace(
            next_node->GetIndex(),
            next_node->GetCost());
    }
}
```

这个阶段：

1. 当前节点进入 CLOSE。
    
2. 生成多个不同转向和行驶方向的候选节点。
    
3. 检查节点是否为空。
    
4. 检查节点是否已经访问。
    
5. 检查节点是否碰撞或越界。
    
6. 计算候选节点代价。
    
7. 将有效节点加入 OPEN。
    

然后继续下一轮搜索。

---

## 7. Reed-Shepp（RS）解析曲线是什么？

RS 曲线是一种考虑车辆最小转弯半径、允许前进和倒车的运动学路径。

它可以由以下运动段组合构成：

- L：左转弧线。
    
- R：右转弧线。
    
- S：直线。
    

例如：

```
RSR
```

表示：

```
右转 → 直行 → 右转
```

每一段还可能是前进或倒车。

你之前的日志中出现：

```
Reed-Shepp path selected:
pattern = RSR
total_length = 2.991 m

RS segments:
R backward 0.252 m
S backward 2.629 m
R backward 0.110 m
```

说明本次选中的 RS 曲线由三段倒车运动组成。

RS 曲线的优点：

- 可以快速连接两个车辆姿态。
    
- 考虑车辆转弯半径。
    
- 支持前进和倒车组合。
    
- 能够减少普通搜索需要扩展的节点数量。
    

但是：

**RS 曲线本身不保证一定无碰撞，必须经过有效性和碰撞检查。**

---

## 8. LoadRSPinCS()：RS 终点如何进入搜索结构？

你的代码：

```
std::shared_ptr<Node3d> HybridAStar::LoadRSPinCS(
    const std::shared_ptr<ReedSheppPath> reeds_shepp_to_end,
    std::shared_ptr<Node3d> current_node) {

    std::shared_ptr<Node3d> end_node =
        std::shared_ptr<Node3d>(
            new Node3d(
                reeds_shepp_to_end->x,
                reeds_shepp_to_end->y,
                reeds_shepp_to_end->phi,
                reeds_shepp_to_end->kappa,
                XYbounds_,
                planner_open_space_config_));

    end_node->SetPre(current_node);

    close_set_.emplace(
        end_node->GetIndex(), end_node);

    return end_node;
}
```

这里：

1. 从 RS 路径中取出 x、y、phi、kappa。
    
2. 创建 RS 终点 Node3d。
    
3. 设置父节点为当前搜索节点。
    
4. 将 RS 终点节点加入 CLOSE。
    
5. 返回 RS 终点节点。
    

这里需要注意：

RS 路径本身可能包含一整组轨迹采样点。

但仅从这个函数看，不能完全确认 RS 的全部轨迹点是如何保存到最终节点中的。

还需要结合：

- Node3d 构造函数。
    
- ReedSheppPath 的数据结构。
    
- AnalyticExpansion() 中的路径赋值逻辑。
    

才能确认 RS 轨迹是否完整保留。

---

## 9. RS 失败后，普通搜索会不会输出路径？

这是理解你当前代码最重要的地方之一。

答案：

__RS 失败后，普通 Hybrid A_ 会继续搜索，但按照目前提供的 Plan() 代码，它不会直接把普通搜索路径作为最终泊车路径输出。_*

流程：

```
起点
  ↓
进入 OPEN
  ↓
取出当前节点
  ↓
尝试 RS
  ↓
RS 成功？
  ├── 是
  │    ↓
  │  设置 final_node_
  │    ↓
  │  跳出搜索循环
  │
  └── 否
       ↓
     当前节点进入 CLOSE
       ↓
     Next_node_generator()
       ↓
     生成普通搜索节点
       ↓
     计算代价
       ↓
     新节点进入 OPEN
       ↓
     继续搜索
       ↓
     再次尝试 RS
```

### 9.1 RS 多次失败，但后续成功

例如：

```
第1轮：RS失败
第2轮：RS失败
第3轮：RS失败
第4轮：RS失败
第5轮：RS成功
```

那么：

- 前四轮进行普通 Hybrid A* 搜索。
    
- 第五轮找到可行 RS 连接。
    
- 规划成功。
    
- 输出完整路径。
    

### 9.2 RS 始终失败

如果：

```
while (!open_pq_.empty())
```

最终搜索结束，而：

```
final_node_ == nullptr
```

那么：

```
if (final_node_ == nullptr) {
    return false;
}
```

整个规划失败。

也就是说：

**普通搜索扩展了很多节点，并不代表最终已经有一条可输出的泊车路径。**

当前代码需要成功建立 RS 解析连接，才能走到最终路径输出。

---

## 10. GetResult()：最终路径如何回溯？

你的代码核心：

```
std::shared_ptr<Node3d> current_node = final_node_;

while (current_node->GetPreNode() != nullptr) {

    std::vector<double> x = current_node->GetXs();
    std::vector<double> y = current_node->GetYs();
    std::vector<double> phi = current_node->GetPhis();
    std::vector<double> kappa = current_node->GetKappas();

    std::reverse(x.begin(), x.end());
    std::reverse(y.begin(), y.end());
    std::reverse(phi.begin(), phi.end());
    std::reverse(kappa.begin(), kappa.end());

    x.pop_back();
    y.pop_back();
    phi.pop_back();
    kappa.pop_back();

    hybrid_a_x.insert(
        hybrid_a_x.end(), x.begin(), x.end());

    hybrid_a_y.insert(
        hybrid_a_y.end(), y.begin(), y.end());

    hybrid_a_phi.insert(
        hybrid_a_phi.end(), phi.begin(), phi.end());

    hybrid_a_kappa.insert(
        hybrid_a_kappa.end(), kappa.begin(), kappa.end());

    current_node = current_node->GetPreNode();
}
```

这个函数不是重新规划路径，而是把已经建立好的父子节点链还原成完整轨迹。

回溯关系：

```
final_node_
    ↓
RS 终点节点
    ↓
普通搜索节点
    ↓
普通搜索节点
    ↓
普通搜索节点
    ↓
start_node_
```

每个节点保存一段局部轨迹。

回溯时：

1. 获取当前节点的局部轨迹。
    
2. 将轨迹反转，使其变成从父节点到当前节点的顺序。
    
3. 删除重复的父节点位置。
    
4. 拼接到最终轨迹数组。
    
5. 沿父节点继续回溯。
    
6. 最后补上起点。
    

这样就得到从起点到终点的完整路径。

注意：

```
std::reverse(...)
```

是对输出路径点顺序进行反转，不代表车辆前进或倒车。

车辆的运动方向需要看节点的 direction 信息或轨迹数据。

---

## 11. 普通 Hybrid A* 和 RS 的区别

|对比项|普通 Hybrid A* 扩展|RS 解析扩展|
|---|---|---|
|作用|在搜索空间中逐步寻找连接位置|尝试直接连接当前节点和目标点|
|生成方式|Next_node_generator()|Reed-Shepp 曲线计算|
|运动约束|根据车辆运动学模型生成|根据最小转弯半径等约束生成|
|运动方向|前进或倒车|支持前进和倒车组合|
|轨迹点|一次扩展包含多个采样点|一条解析曲线包含多个采样点|
|搜索作用|产生新的搜索节点|尝试完成最终连接|
|失败后|继续普通搜索|等待后续节点再次尝试|

可以这样理解：

```
普通 Hybrid A*：
负责在搜索空间中寻找合适的连接位置。

RS：
负责尝试从这个位置直接连接终点。
```

---

## 12. 结合当前代码，完整算法流程

```
输入：
起点车辆姿态
目标车辆姿态
地图边界
障碍物信息
        |
        v
创建起点和终点 Node3d
        |
        v
检查起点和终点有效性
        |
        v
生成 DP 栅格启发式地图
        |
        v
起点进入 OPEN
        |
        v
OPEN 是否为空？
   |             |
   否            是
   |             |
   v             v
取出当前节点   规划失败
   |
   v
是否尝试 RS？
   |
   v
RS 解析扩展
   |
   +----------------+
   |                |
 RS成功           RS失败
   |                |
   v                v
设置final_node_  当前节点进入CLOSE
   |                |
   v                v
跳出搜索循环     Next_node_generator()
   |                |
   |                v
   |             生成多个候选节点
   |                |
   |                v
   |             碰撞检查、边界检查
   |                |
   |                v
   |             计算节点代价
   |                |
   |                v
   |             有效节点进入OPEN
   |                |
   |                v
   |             返回搜索循环
   |
   v
GetResult()
   |
   v
沿父节点回溯
   |
   v
拼接普通搜索轨迹和RS轨迹
   |
   v
输出最终泊车路径
```

---

## 13. 当前项目日志应该如何理解？

之前的日志：

```
loops = 250
analytic_expansion = 250

next_gen_calls = 2490
next_nodes = 1885
valid_nodes = 1356
explored_nodes = 562
```

含义：

|统计项|含义|
|---|---|
|loops|搜索循环次数|
|analytic_expansion|RS 解析扩展尝试次数|
|next_gen_calls|普通候选节点生成函数调用次数|
|next_nodes|成功生成的候选节点数量|
|valid_nodes|通过有效性检查的节点数量|
|explored_nodes|新增搜索节点数量|

从这些统计可以看出：

1. 算法确实执行了普通 Hybrid A* 节点扩展。
    
2. 算法也在每轮搜索中尝试 RS 解析连接。
    
3. RS 不是单独运行一次，而是随着普通搜索节点不断尝试。
    
4. 最终是否成功，要看是否有一次 RS 连接成功。
    

之前日志还有：

```
Reed-Shepp path selected:
pattern = RSR
total_length = 2.991 m
```

说明本次成功选中了 RSR 类型的解析路径。

---

## 14. 当前算法设计中需要重点关注的问题

### 14.1 RS 是否只要可行就立即终止？

你的 Plan() 中：

```
if (AnalyticExpansion(current_node)) {
    break;
}
```

说明只要某次 RS 成功，就直接结束搜索。

这意味着当前代码并没有看到：

- 继续搜索其他连接位置。
    
- 比较多个 RS 成功方案的总代价。
    
- 比较普通搜索路径与 RS 路径的最终总代价。
    

因此，这种设计更接近：

**找到第一个满足条件的 RS 连接就结束，而不是搜索所有可行方案后选择全局最优路径。**

### 14.2 RS 失败是否可以使用普通搜索路径兜底？

当前代码没有看到普通搜索路径直接作为最终结果输出的逻辑。

可以考虑增加兜底方案：

- RS 成功：输出 RS 连接路径。
    
- RS 失败：如果普通搜索到达目标容差范围内，尝试直接以普通搜索路径作为结果。
    

但需要额外设计：

- 目标位置容差。
    
- 目标航向角容差。
    
- 车辆运动学约束检查。
    
- 完整轨迹碰撞检查。
    
- 终点连接与路径回溯逻辑。
    

不能直接把 CLOSE 集合中的任意节点当作最终路径。

### 14.3 轨迹采样间隔与碰撞检测

当前 Next_node_generator() 中：

```
step_size_
```

同时控制运动积分步长和轨迹采样间隔。

采样间隔过大，可能漏掉两个采样点之间的障碍物碰撞。

因此需要结合：

- 车辆尺寸。
    
- 障碍物尺寸。
    
- 地图分辨率。
    
- 采样步长。
    
- 碰撞检测方式。
    

确定合适的采样精度。

---

## 15. 一句话总结

Hybrid A* 泊车算法可以理解为：

**利用车辆运动学模型生成前进、倒车和不同转向的候选搜索节点，通过 OPEN/CLOSE 集合和代价函数不断搜索，同时利用二维 DP 栅格地图提供启发式代价，并在搜索过程中反复尝试 Reed-Shepp 解析曲线连接终点；一旦 RS 连接成功，就通过父节点回溯和局部轨迹拼接，输出完整泊车路径。**

当前这份代码的关键特点：

1. 一个搜索节点可以保存一整段局部轨迹采样点。
    
2. 轨迹采样点不等于搜索节点。
    
3. 一次普通扩展生成一个候选 Node3d。
    
4. `step_size_` 控制局部轨迹采样间隔。
    
5. `xy_grid_resolution_` 控制地图栅格大小，并参与计算一次扩展的长度。
    
6. DP 地图提供启发式代价，不等于 Hybrid A* 本身按照二维栅格逐格搜索。
    
7. 普通搜索负责寻找合适的 RS 连接位置。
    
8. 当前代码依赖 RS 成功建立最终连接；如果始终失败，规划最终返回失败。