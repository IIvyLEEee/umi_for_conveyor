# UR5e 真机推理使用说明

当前分支：`ur5e-reset-init-joints`，基于 `clear-demo`。这个分支做了两件事：

- 将 UR5e 复位关节角写入 robot config，并由 `--init_joints` 显式触发。
- 将 diffusion/DDIM 去噪步数从硬编码改成命令行参数。

## 基本启动

在仓库根目录运行：

```bash
python eval_real.py \
  -i data/models/epoch=0130-train_loss=0.012.ckpt \
  -o data/eval_conveyor/<run_name> \
  -rc example/eval_robots_config.yaml \
  --camera_reorder 0
```

按实际情况替换 checkpoint 和输出目录。

## UR5e 复位

复位不是自动触发的。只有启动命令里显式加 `-j` 或 `--init_joints` 才会触发：

```bash
python eval_real.py \
  -i data/models/epoch=0130-train_loss=0.012.ckpt \
  -o data/eval_conveyor/<run_name> \
  -rc example/eval_robots_config.yaml \
  --camera_reorder 0 \
  --init_joints
```

如果不想让 UR5e 启动时移动到复位关节角，不要加 `-j` / `--init_joints`。

当前 UR5e 的复位目标在 `example/eval_robots_config.yaml`：

```yaml
"joints_init_deg": [0, -90, -90, -90, 90, 0]
```

触发时机：`BimanualUmiEnv` 创建 UR5e controller 时读取该字段，将角度转成 radians，传给 `RTDEInterpolationController`。controller 连接 UR5e、设置 TCP/payload 后，会执行一次 `moveJ()`，然后才进入 500 Hz `servoL` 控制循环。

它不会在每个 episode 开始时触发，也不会在按 `c` 交给 policy 控制时触发。

## 去噪步数

现在可以用命令行修改 DDIM 去噪步数：

```bash
python eval_real.py ... --num_inference_steps 16
```

也可以用别名：

```bash
python eval_real.py ... --denoise_steps 16
```

## 等比例模拟慢推理

如果想在 RTX 4070 上模拟更慢的硬件，可以使用：

```bash
python eval_real.py ... --inference_latency_scale 3.0
```

这个参数会在每次 policy 推理结束后，根据本次实际耗时补 sleep：

```text
extra_sleep = measured_inference_latency * (scale - 1)
```

例如真实推理耗时约 54 ms：

```text
--inference_latency_scale 1.0 -> 约 54 ms，不额外等待
--inference_latency_scale 2.0 -> 约 108 ms
--inference_latency_scale 3.0 -> 约 162 ms
```

补 sleep 的位置在 `policy.predict_action()` 完成、action 转换完成之后，且在过期动作过滤之前。因此它会真实影响 `curr_time`，能模拟慢推理导致更多动作 timestamp 过期的情况。

默认值保持当前真机推理代码的行为：

```text
num_inference_steps = 4
```

这个参数会在 checkpoint 加载后设置：

```python
policy.num_inference_steps = num_inference_steps
```

它只改变每次 `policy.predict_action()` 内部的 DDIM 迭代次数，不改变动作 chunk 长度。当前 checkpoint 的 `shape_meta.action.horizon` 是 16，所以每次推理仍然输出 16 个动作 step。

默认控制频率是 10 Hz，因此：

```text
单个动作 step = 0.1 s
16-step chunk = 1.6 s
steps_per_inference 默认 6，即每 0.6 s 重新推理一次
```

实测参考：在 RTX 4070 上，dummy obs 下 `predict_action()` 大约是：

```text
--num_inference_steps 16: 54 ms
--num_inference_steps 4 : 33 ms
```
