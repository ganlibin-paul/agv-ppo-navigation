# Isaac Sim训练、域随机化与验收

## 已知运行环境

历史服务器使用Isaac Sim 4.5、RTX A6000；PPO通过Stable-Baselines3在Kit兼容Python环境运行。训练不是在ROS Python 2.7里运行，也不是开着Windows WebRTC客户端就能训练。NVIDIA软件与机器人USD不包含在仓库。

需要先取得：

- 本车已经验证驱动的USD，原始底盘、关节、碰撞和PhysX设置；以及USD引用的网格/材质。
- V94训练所需原V92检查点；V95对照所需V94检查点。分别记录SHA256。
- 可运行现有Isaac API的Kit环境、NumPy、PyTorch、gymnasium、Stable-Baselines3；在该环境记录实际`pip freeze`，本仓库未伪造一份所有版本都已复现的锁文件。
- Isaac官方安装目录与streaming启动脚本；headless/streaming由既有安装方式选择。

## 渲染机器路径模板

从仓库运行Python 3工具；它只生成文件，不启动Isaac、不杀现有进程：

```bash
python tools/render_simulation.py   --project-root /srv/agv-simulation   --robot-usd /srv/robot-assets/agv_drive_ok.usd   --ppo-v92 /srv/checkpoints/ppo_v92_final.zip   --ppo-v94 /srv/checkpoints/ppo_v94_final.zip
```

先检查生成的`/srv/agv-simulation/render_manifest.json`。两个权重参数都显式要求绝对路径；V94训练使用V92、V95使用V94，不能相互代替。

```bash
cd /path/to/isaac-sim-4.5
./isaac-sim.streaming.sh --exec /srv/agv-simulation/scripts/train_domain_boxes_v94.py
# 与上面训练分开运行；同一GPU/场景不要不经检查同时启动两份。
./isaac-sim.streaming.sh --exec /srv/agv-simulation/scripts/evaluate_route_fix_v95.py
```

脚本启动后检查输出目录的config/status/preflight；worker失败时应读取failure.txt，不把窗口仍开着当作仍在训练。后台可沿用系统已有进程管理，关闭仅观看用的WebRTC窗口不会终止远端训练，但服务器必须保持开机。

## V94实际场景

16环境：2个单箱、6个双箱、8个三箱。每回合随机静态布局，不是行驶途中突然插箱：

|参数|范围|
|---|---|
|箱体前后长度|0.20～0.60米|
|箱体横向宽度|0.35～0.75米|
|箱高|0.40～0.85米|
|第一箱x|4.2～5.4米|
|同侧布局箱间外沿净距|0.6～2.5米|
|轻错位布局净距|1.5～3米|
|大错位S布局净距|2.3～3.5米|
|通道内宽|3.3米|
|接管最高速度|0.2米/秒|
|起步位置/朝向扰动|y±0.08米、yaw±5度|
|扫描噪声/丢束|标准差1～4毫米、0～4%|

随机化改变显示、实际碰撞体和评估几何的同一参数；不改变原车车体结构和传感器契约。范围为设计压力测试，尚未声称覆盖真机全部动力学和时延分布。策略只用观测，箱体真值用于生成场景、奖励和距离验收，不能作为部署策略的额外输入。

## 验收和产物

V94目标262144步、训练墙钟预算8小时，完成后64组冻结完整路线：16原场景、16未见随机布局、32箱距探测。记录PhysX接触、旋转车身到各箱最短距离、到侧墙最短距离、完整到点/朝向/停稳、无进展与停滞原因。

主要文件：`progress.csv`、`checkpoint_*.zip`、`ppo_v94_final.zip`、`training_complete.json`、`evaluation.json`、`evaluation_trajectories_*.json`、`episode_layouts.jsonl`、`episode_metrics.jsonl`、`route_events.jsonl`、`clearance_summary.json`和`clearance_report.md`。学习曲线可从progress.csv画，但不能以平均奖励高替代到点率。

V95固定V94模型、共享布局与传感器种子，对8个代表场景做旧/新参考控制器16环境比较。模型不更新，不能把其状态中的历史`model.num_timesteps=262144`当成本次新增训练步数。新实现必须同时改善失败回归、不损伤成功回归和净空，再考虑续训。

## 修改源头

模板是这次发布的可渲染源码快照；修改`simulation/templates`后重跑render工具。不要修改生成目录再以为仓库已更新。该快照保留历史实验配置描述中的版本痕迹，真正执行的模型、环境与算法以相关入口/最终config覆盖字段为准。
