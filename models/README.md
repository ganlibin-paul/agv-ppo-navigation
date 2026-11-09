# 模型

`v35/actor_v33.npz`是历史V33策略通过V35导出的NumPy actor，架构164→128→128→2，三层tanh；不是V94长训模型。

actor SHA256：`61fda141d4f72cf4ceff017a23c973cce5f533f4c74602901e266d3b9a6d7573`。

`export_report.json`是历史导出比对报告；其中真实帧误差属于历史测量，真实输入本身不上传。`synthetic_golden.npz`提供1000行合成观测及期望动作，供离线回归。

V92/V94检查点需要另行从持有者的服务器获取，记录哈希后用于仿真。不能将PPO ZIP重命名为NPZ，也不能将任意同为164维的模型默认认定为可替换模型。
