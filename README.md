# Real Flight

```bash
# Terminal 1
./sh_files/rspx4.sh

# Terminal 2
roslaunch px4ctrl run_ctrl_real.launch

# Terminal 3
roslaunch diff_planner real_single_drone.launch

# Terminal 4
```


<img src="images/nus_logo.png" alt="nus logo" align="right" height="80" />

# Diff-Planner

## 概述
**Diff-Planner** 是为**微分智飞**公司旗下教育无人机子品牌**非凸空间**适配的单机导航避障算法。其基于开源算法 **[EGO-Planner-v2](https://github.com/ZJU-FAST-Lab/EGO-Planner-v2)** ，并由原班人马深度参与算法优化。在继承 **EGO-Planner** 优秀框架的基础上，针对教育无人机平台的特殊需求进行了全面适配和增强，旨在提供更稳定、更可靠的科研体验。

<p align="center">
  <img src="images/navigation.gif" alt="nav" width="600" />
</p>

## 算法优化
- **Diff-Planner** 在 **[EGO-Planner-v2](https://github.com/ZJU-FAST-Lab/EGO-Planner-v2)** 基础上做了多处优化，包括：
>+ **修复**局部规划时 A* 终点在障碍物里，尝试沿 A* 起始方向推出障碍物时的bug。

>+ **修复**优化过程中频繁打印局部目标点在障碍物里（"Local target in collision, skip this planning."），规划器卡死的bug。

>+ **修复**在状态机中使用 **planNextWaypoint()** 导致状态机卡死的bug。

>+ **新增**规划优化异常检测， 避免动力学不可行的轨迹发出，新增了动力学容忍值（[advanced_param_exp.xml](src/diff_planner/plan_manage/launch/include/advanced_param_exp.xml)中设置）：\
\<param name="optimization/vel_tolerance" value="1.0" type="double"/>  \
\<param name="optimization/acc_tolerance" value="1.0" type="double"/>

>+ **修复**遇到大障碍物后，无人机在大障碍物面前反复徘徊卡死的bug，同时增加是否使用大障碍物检测的开关（[advanced_param_exp.xml](src/diff_planner/plan_manage/launch/include/advanced_param_exp.xml)中设置）：\
>\<param name="fsm/enable_stuck_detect" value="true"/> 

>+ **新增**优化失败次数过多的处理。

>+ **新增**激光雷达建图 **raycast** 版本，使激光雷达建图更加稳定。

>+ **新增**用户接口 **user_command** 功能包，使用户能以多种方式设置途径点以及设置返程点。

>+ **traj_server** 节点**新增**yaw角控制接口，用户可根据需要在规划过程中控制无人机yaw角。

- **本项目会长期维护并根据用户反馈持续优化。**

## 运行环境
本项目基于ROS1开发，请根据所使用ubuntu版本安装对应版本ROS1，支持ubuntu16.04, 18.04和20.04。


## 仿真运行步骤

### 1. 下载源码并编译:
```
git clone https://github.com/DifferentialRobotics/Diff-Planner.git
cd Diff-Planner
catkin_make
```

### 2. 单机rviz手动指点飞行：
```
cd Diff-Planner
source devel/setup.zsh # 如果使用bash终端，则执行: source devel/setup.bash
roslaunch diff_planner run_sim_single.launch
```
使用rviz中的**3D Nav Goal**插件，在地图上按住左键选择目标点x-y平面位置，按住左键不松手同时按住右键上下拖动调整目标点z轴位置，之后松开鼠标即发送目标点，无人机开始规划。
<p align="center">
  <img src="images/rviz_test.gif" alt="rviz_tes" width="600" />
</p>


### 3. 单机预设点飞行：
在 [points.yaml](src/user_command/multipoint/config/points.yaml) 文件中 `test1` 下设置期望途经点，`test_back` 下设置返程目标点，之后通过以下指令执行任务：
```
cd Diff-Planner
source devel/setup.zsh
roslaunch diff_planner run_sim_single.launch
cd Diff-Planner #新建终端
./sh_files/pub_trigger.sh #开始执行任务 或在rviz中用2D Nav Goal插件在地图任意位置点击也能开始执行任务
./sh_files/back.sh #开始返程规划
```
注：通过修改 [multipointplan_sim.launch](src/user_command/multipoint/launch/multipointplan_sim.launch) 中的 `fligt_type` 可实现多种指点规划方式，如自定义到达每个途经点过程中的飞机yaw角，控制到达每个途经点后的停留时间等，详见 [points.yaml](src/user_command/multipoint/config/points.yaml) 顶部注释。

### 4. 集群预设点飞行：
在 [run_sim_swarm.launch](src/diff_planner/plan_manage/launch/sim/run_sim_swarm.launch) 中设置每架无人机的目标点 `target0_x/y/z`，之后通过以下指令执行任务：
```
cd Diff-Planner
source devel/setup.zsh
roslaunch diff_planner run_sim_swarm.launch
cd Diff-Planner #新建终端
./sh_files/pub_swarm_trigger.sh #开始执行任务
```
<p align="center">
  <img src="images/swarm.gif" alt="rviz_tes" width="600" />
</p>

## 实机运行步骤

### 1. 下载源码并编译
  ```
  git clone --recursive https://github.com/DifferentialRobotics/Diff-Planner.git
  cd Diff-Planner
  catkin_make
  ```

### 2. 雷达定位自主避障飞行
0. 按照 **faster-lio** 中的readme配置好雷达定位参数，并启动雷达定位脚本测试定位是否正常：
    ```
    cd Diff-Planner
    ./sh_files/run_lio.sh
    ```
    启动脚本后拿起无人机在空中晃动，并沿飞行场地行走一圈后放回原地，查看定位是否稳定。

1. 在配置文件 [multipointplan_exp_lio.launch](src/user_command/multipoint/launch/multipointplan_exp_lio.launch) 中
查看设置的飞行模式

    >+  `fligt_type` 为 1 表示选择 `test1` 模式进行多点规划，即依次经过各途经点

    >+  `fligt_type` 为 2 表示选择 `test2` 模式进行多点规划，可以控制到达各途径点后的停留时间

    >+  其余模式的定义可在 [points.yaml](src/user_command/multipoint/config/points.yaml) 中找到
    
    >+  当设置 `auto_planning` 为1时，起飞悬停后会开始自动规划
    
    >+  当设置 `auto_landing` 为1时，到达最后一个目标点后会自动降落

2. 在 [points.yaml](src/user_command/multipoint/config/points.yaml) 中设置 `test1` 对应的途径点坐标，以及返程规划的途径点坐标。

3. 检查 [ctrl_param_fpv.yaml](src/realflight_modules/px4ctrl/config/ctrl_param_fpv.yaml) 中起飞高度 `takeoff_height` 设置是否合适。

4. 打开遥控器，确认拨杆位置是否正确。

5. 进入 Diff-Planner 工作空间并启动雷达定位导航脚本
    ```
    cd ~/Diff-Planner
    ./sh_files/run_all_lio.sh
    ```
    > 注意：若没有执行规划任务需求只是测试程序，请保持无人机处于急停状态或关闭遥控器，以免误触遥控器导致无人机自动起飞！！！

    待程序完全启动后 rviz 界面如下所示：
    ![rviz_point_cloud](images/rviz_point_cloud_1.png)

6. 起飞：
    ```
    cd ~/Diff-Planner
    ./sh_files/takeoff.sh
    ```

7. 执行规划任务：
    ```
    cd ~/Diff-Planner
    ./sh_files/pub_trigger.sh
    ```

8. 返程规划：
    ```
    cd ~/Diff-Planner
    ./sh_files/back.sh
    ```

9. 降落：
    ```
    cd ~/Diff-Planner
    ./sh_files/land.sh
    ```
    > 注意：待无人机落地后及时锁桨并在任务终端输入 ctrl+c 结束任务。
    > 起飞/执行规划任务/返程规划/降落也可使用遥控器控制，详见配套产品手册。

### 3. 视觉定位自主避障飞行
若要使用**视觉定位**下规划，需要先在 [run_exp_single_vio.launch](src/diff_planner/plan_manage/launch/exp/run_exp_single_vio.launch) 中替换深度相机内参 `cx/cy/fx/fy`，内参查看方式：
```
cd Diff-Planner
./sh_files/run_vins.sh
rostopic echo /camera/depth/camera_info
```
消息中的K矩阵即为深度相机内参，注意矩阵中的顺序为 `fx/cx/fy/cy`。

视觉定位下规划脚本为 [run_single_vio.sh](sh_files/run_single_vio.sh) ，
其余配置方式与程序执行逻辑与雷达定位一致。

## 致谢与声明
本项目在开发过程中参考并使用了 **[EGO-Planner-v2](https://github.com/ZJU-FAST-Lab/EGO-Planner-v2)**，特此感谢浙江大学 **FAST-Lab** 团队的开源贡献。

相关代码均严格遵循原项目的开源许可协议使用，用户在使用本项目时，请务必遵守相应的许可证条款。

# Q&A
请随时提交问题或讨论，我们会在看到问题后尽快回复。
