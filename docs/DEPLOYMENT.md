# 本AGV真机部署手册

本手册说明历史V85集成的文件和流程。**上传GitHub不等于已在车上安装或启用控制。** V94成功率不足、V95尚在验证，不随本次发布替换车端权重。

## 1. 需要准备什么

|项目|本项目已知条件|
|---|---|
|AGV|双驱动轮，原底盘驱动、导航、定位、急停正常|
|车端软件|ROS 1 Melodic、Python 2.7.17、NumPy 1.13.3是历史实跑版本；不是ROS 2直接可用包|
|容器|历史容器名`ros_env-run`；部署前用`docker ps`确认，不假定所有车相同|
|ROS工作区|现场`install/setup.bash`及`lyric_msgs`自定义消息/服务；仓库不含厂商导航二进制|
|输入|双原始激光、1083融合扫描、odom、TF、路径、活动任务、地图查询、参考速度和运行状态，见输入文档|
|控制输出|`/yocs_cmd_vel_mux/input/move_base_nav`，由已有mux接到原底盘；遥控/硬急停不能被新输入覆盖|
|模型|仓库`models/v35`含历史actor及哈希；不需要在车端安装PyTorch或Stable-Baselines3|
|实物核查|前激光相对车体偏移、本车车头外沿、箱体是否高于扫描面、通道宽度、实测车边净空|
|现场|低速固定泡沫、无人绕行区、有人守车体独立急停。遥控空格曾恢复导航，不等同持续硬急停|

`ros_env-run`和工作区路径是本车历史名称，不包含IP或登录信息。先核对实际安装，不能直接把此表当成新车环境已满足。

## 2. 上传文件，保持原服务不变

可在工作电脑克隆，再复制到AGV主机临时目录；或在网络可用的AGV主机上克隆。私有仓库登录走GitHub官方授权/已有凭据，不把密码或令牌写进命令、README或`.env`上传。

```bash
# AGV宿主机：将HOST_PROJECT改为你实际上传的项目完整路径。
HOST_PROJECT=/tmp/agv-ppo-navigation
AGV_CONTAINER=ros_env-run
docker ps --format '{{.Names}}'
docker exec "$AGV_CONTAINER" mkdir -p /opt/agv-ppo-navigation
docker cp "$HOST_PROJECT/." "$AGV_CONTAINER:/opt/agv-ppo-navigation"
```

先放入新目录，不覆盖原运行目录。历史目录为`/root/ros_workspaces/ppo_shadow_v22_20261031/auto_navigation_v85`，服务名仍是`amr_ppo_auto_v62`，名称不一致是版本延续，不表示还在运行V62代码。真实当前状态必须检查，不能引用历史RUNNING当作现在在线。

## 3. 容器内无运动预检

```bash
source /opt/ros/melodic/setup.bash
source /root/ros_workspaces/ros_ws/install/setup.bash
cd /opt/agv-ppo-navigation
python -c "import sys,numpy,rospy,tf; from lyric_msgs.msg import CurrentMapId; from lyric_msgs.srv import GeneralJsonServer; print(sys.version); print(numpy.__version__)"
python tools/check_actor_onboard.py
python -m py_compile runtime/*.py
rostopic list
rosservice type /get_json_map_data_service
rostopic echo -n 1 /path_nav_status
rostopic echo -n 1 /odom
rostopic echo -n 1 /yocs_cmd_vel_mux/active
rosparam get /yocs_cmd_vel_mux
```

最后一个参数路径须按现场launch/配置核实；若没有该全局参数，读取实际mux节点私有参数或其YAML，不凭空新建一套优先级。

检查任务为空、车已停稳；记录原服务状态和实际配置文件。预检不调用任务下发或发布速度。

```bash
# 无速度发布：缺省不加--enable-motion。
python runtime/run_agv.py   --directory /tmp/amr-ppo-shadow   --model-directory /opt/agv-ppo-navigation/models/v35   --front-x-candidate 0.45
```

这个模式的输入来自实机，actor可运行，但适配器不建立速度Publisher。历史权重、全字段、时间新鲜和TF必须满足；没有任务时等待是正常状态，不要伪造任务字段让检查“通过”。只读采集可直接用：

```bash
python runtime/shadow164_capture_v69.py --output /tmp/agv_readonly.jsonl --seconds 180 --front-x-candidate 0.45
```

正常下发导航由现场原系统操作。核对观测164维有限、融合元数据、任务/路径绑定、扫描时间差、定位、速度参考及日志缺项。自由点导航的显式shadow开关不能解释为自由点运动已验收。

## 4. 安装服务前备份

在导航取消、车停稳、旧adapter不持有控制权之后，保存：

- 原Supervisor配置、原运行源码、原actor及SHA256、原mux配置。
- `supervisorctl status amr_ppo_auto_v62`输出和旧代码实际路径。
- 原`decisions.jsonl`末态及任务、odom的同步记录。

不要在旧adapter还持有控制权时仅停止进程来“切换版本”；先按原流程取消任务、确认停止、完成释放。

提供的`config/amr_ppo_shadow.conf.example`是另名`amr_ppo_shadow`的默认只读配置，`autostart=false`且不含`--enable-motion`。复制前按车端目录修改，只是示例，仓库不会自动覆盖现用`amr_ppo_auto_v62`配置。只读服务可以与原服务观察同一输入，但运动Publisher只允许保留一个接管实例。

## 5. 首次运动测试

先确认只读记录和模型回归通过。现场固定泡沫和通道测量完成、有人监护且独立急停可用，再停用旧运动实例并启动本次低速试验；不要并行两个adapter争用同一mux输入。

```bash
# 以下命令明确允许此进程发布运动；只在现场准备完成后执行。
python runtime/run_agv.py   --directory /var/log/amr-ppo-trial   --model-directory /opt/agv-ppo-navigation/models/v35   --front-x-candidate 0.45   --enable-motion
```

本车历史V85为遇障停稳自动接管模式；开启后不是每趟都等聊天人工按启动。PPO接管限速0.2m/s，参考局部规划跟随上限0.12m/s，原导航速度另由原系统控制。先单箱、再已验证双箱，逐一检查绕过、间距、回线、交还和终点任务状态。不能将本仓库V94/V95源文件复制覆盖V85然后直接运行。

必须读取mux活动通道与输出，确认发布命令被正确采纳、遥控覆盖保持有效。独立急停与上层原系统遇障停车的含义不同：PPO正是接管原遇障停车后的局部绕行，不能指望原导航的同一零速同时继续约束PPO。

## 6. 停止、释放和回退

先在原导航取消任务，并核对活动任务列表为空、odom停稳。如果adapter仍持有控制权，保留它的零速保持，完成确认后通过其release文件协议释放：在实际`--directory`目录写入`release.json`。该协议自身也核对任务为空与新鲜odom；不能绕过它删除lease或伪造状态。

```bash
# 仅在上述取消与停稳确认之后，路径必须是实际运行目录。
printf '{}\n' > /var/log/amr-ppo-trial/release.json
```

现场危险时先操作独立硬急停。空格键短暂停车可能自动恢复，不作为持续隔离。

恢复保存的旧服务配置与源码/actor哈希，执行Supervisor reread/update后检查其实际状态，再核对无任务时不发布PPO速度。回退不重写机器人TF、碰撞体或厂商导航包。

## 7. 新训练模型怎样才可部署

完整链路是：仿真回归通过 → 固定检查点 → 导出与车端NumPy计算等价的actor → 合成和真实输入逐帧对照（历史容差≤2e-6）→ Python 2.7/NumPy目标环境预检 → 只读影子验证 → 现场低速受控运动 → 保存成功版本并保留回退。

仓库只包含历史V35 actor的回归脚本，没有声称它能通用导出任意新SB3模型。V94是PPO ZIP、V35是特定网络NPZ，文件格式和网络结构都必须验证，不能改文件名替换。当前V94只有35.9%到点率，尚不满足把它当稳定生产替换版本的依据。
