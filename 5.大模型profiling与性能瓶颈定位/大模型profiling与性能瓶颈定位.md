# 大模型 Profiling 与性能瓶颈定位

Profiling：采集并分析模型运行时的耗时、算子、资源利用率和时间线数据，使性能优化从经验驱动转为数据驱动。

核心原则：先定位问题，再实施优化；每个结论都应有日志、指标或 Trace 作为证据。

## 为什么要做 Profiling

### 定位时间花在哪里

Profiling 首先回答“慢在哪里”：

- Prefill 阶段慢。
- Decode 阶段慢。
- Host 下发慢。
- 某类算子或某个 kernel 特别耗时。

### 判断资源是否浪费

通过时间线观察：

- NPU 是否存在长时间空闲。
- CPU 是否处于等待状态。
- 算子之间是否存在明显间隙。
- 计算、访存和调度是否连续。

### 验证优化是否有效

修改配置或代码后，使用相同模型、输入、输出长度和执行方式复测，对比修改前后的指标和 Trace，确认优化是否真正生效。

## 性能分析基础定位方法

### 区分 Prefill 与 Decode

- **Prefill**：一次处理完整输入上下文。
- **Decode**：自回归地逐步生成后续 token。

两个阶段的计算特征和瓶颈类型不同，必须分别统计、分别分析。

### 关注核心指标

| 指标 | 含义 | 主要关联阶段 |
| --- | --- | --- |
| TTFT | Time To First Token，首 token 延迟 | Prefill |
| TPOT | Time Per Output Token，每个输出 token 的平均耗时 | Decode |
| TPS | Tokens Per Second，每秒生成的 token 数 | Decode |
| Throughput | 单位时间内系统处理的总 token 数 | 整体吞吐 |
| Average Inference Time | 一次推理的平均总耗时 | 端到端 |

### 构建证据链

```text
性能现象 --> Trace / 日志证据 --> 瓶颈假设 --> 修改一个变量 --> 复测结果
```

### 设计对比实验

固定其他条件，只改变一个变量，例如：

- 执行模式。
- 输入长度。
- `max_new_tokens`。
- batch 大小。

这样才能判断单项改动对性能的真实影响。

## 标准工作流：先测量，再优化

### 1. 运行 Baseline

固定以下条件并记录原始结果：

- 模型与精度配置。
- 输入内容和输入长度。
- 输出长度。
- batch 大小。
- 执行模式。

稳定的 Baseline 是优化结论可信的前提。

### 2. 开启 Profiler

只在需要分析的实验中开启 Profiler，避免采集开销影响常规性能结果。

### 3. 观察 Trace

按以下顺序观察：

1. 区分 Prefill 和 Decode 阶段。
2. 查看 NPU 时间线是否存在空泡。
3. 查看最长、最密集的算子区域。
4. 再定位具体算子名称和 kernel。

### 4. 提出假设

根据现象提出可验证的假设，例如：

- Host 下发开销明显。
- Decode 访存压力较高。
- 某类算子调用次数过多。
- 算子之间存在同步或调度间隙。

### 5. 只修改一个变量

一次只调整一项配置或代码，避免多个变化相互干扰。

### 6. 复测并记录

使用与 Baseline 相同的方法再次运行，对比指标和 Trace 的变化，判断假设被支持还是被推翻。

这六步通常需要多次迭代：

```text
观察 --> 假设 --> 修改 --> 复测 --> 再观察
```

## 日志阅读与基线建立

开启 Profiler 之前，可以先从普通运行日志中获得性能概览。

### 日志中可直接获取

- `prefill time`：输入处理阶段耗时。
- `decode step time(s)`：每一步生成耗时。
- `average inference time`：平均推理耗时。
- 生成 token 数：实际输出长度。

### 由日志换算指标

$$
\operatorname{TTFT}\approx \operatorname{prefill\ time}+\operatorname{首步开销}
$$

$$
\operatorname{TPOT}\approx \operatorname{mean}(\operatorname{decode\ step\ time})
$$

$$
\operatorname{TPS}=\frac{\operatorname{生成\ token\ 数}}{\operatorname{生成总耗时}}
$$

固定 batch 时，可用下式估算整体吞吐：

$$
\operatorname{Throughput}\approx \operatorname{batch}\times\operatorname{TPS}
$$

每次运行应同时记录配置、日志关键行和计算得到的指标，形成可比较的性能矩阵。

## Profiler 的三类核心输出

这里重点使用三类互补文件，按照“聚合统计 --> 单个 kernel --> 时间线”的顺序逐层深入。

| 文件 | 主要用途 | 关注内容 |
| --- | --- | --- |
| `op_statistic.csv` | 按 OP 类型聚合耗时和调用次数 | Top 耗时算子类型、Count、Ratio |
| `kernel_details.csv` | 查看每个 kernel 的明细记录 | Name、Type、Duration、Wait Time |
| `trace_view.json` | 查看 Host 与 NPU 的时间线 | 任务是否连续、是否存在空泡 |

建议阅读顺序：

```text
op_statistic --> kernel_details --> trace_view
宏观热点          单个实例              时间线验证
```

## op_statistic：Top 耗时算子分析

`op_statistic.csv` 是聚合视角，适合快速判断时间主要消耗在哪类算子上。

| 指标 | 含义 | 读法 | 可能提示 |
| --- | --- | --- | --- |
| Total Time | 某类 OP 的总耗时 | 降序查看前几项 | 主要时间消耗在哪些算子类型 |
| Count | 该类 OP 的调用次数 | 对比不同 OP 的调用频次 | 是否存在大量小算子调用 |
| Ratio | 该类 OP 占总时间的比例 | 优先关注高占比项 | 确定优化优先级 |
| Prefill/Decode 对比 | 两阶段分别统计 | 分别查看两个阶段的 Top OP | 瓶颈是否只集中于某个阶段 |

分析步骤：

1. 先按 Ratio 或 Total Time 排序，找出占比最高的算子类型。
2. 对比 Prefill 与 Decode 的 Top OP，不要混合两个阶段下结论。
3. 发现异常类型后，到 `kernel_details.csv` 中按 Name 或 Type 筛选。

## kernel_details：逐 kernel 明细分析

`kernel_details.csv` 是微观视角，用于拆解单个 kernel 的耗时构成和具体耗时点。

| 关键列 | 含义 | 分析方式 | 异常信号 |
| --- | --- | --- | --- |
| Name / Type | 算子实例名与所属类型 | 按 Type 分组查看分布 | 某类算子反复出现且数量多 |
| Duration | 单次 kernel 执行耗时 | 降序查找最长 kernel | 少数 kernel 占据大量时间 |
| Wait Time | 任务在队列中的等待时间 | Wait 偏大时回到 Trace 查看间隙 | Host 下发、同步或排队瓶颈 |

分析步骤：

1. 按 Duration 降序，找出耗时最长的 kernel。
2. 检查对应 Name 和 Type，确认是否为预期算子。
3. 查看同类 kernel 是否大量重复出现。
4. Wait Time 明显偏大时，回到 Trace 定位对应时间段的间隙来源。

`op_statistic` 与 `kernel_details` 需要结合使用：前者定位热点类型，后者检查该类型的具体实例与耗时分布。

## trace_view：阶段识别与空泡定位

Trace 是时间线视角。分析时先识别阶段结构，再定位空泡和热点。

### Prefill 的典型 Trace

```text
CPU：tokenize
NPU：================ computing =================   ----- free / gap -----
```

Prefill 通常表现为一次较大的连续计算块。长输入会扩大矩阵乘法和 Attention 计算区域，计算结束后可能存在空闲区间。

### Decode 的典型 Trace

```text
NPU：=== step 1 ===  gap  === step 2 ===  gap  === step 3 === ...
```

Decode 通常表现为多个重复步骤。每步只生成一个新 token，随着步骤增加，应关注每步耗时、间隙和波动是否发生变化。

### Trace 的阅读顺序

1. **阶段识别**：确认当前区域属于 Prefill 还是 Decode。
2. **空泡定位**：查找 `Free / Gap`，判断资源是否被浪费。
3. **热点发现**：优先检查最长、最密集的算子区域。
4. **交叉验证**：与 `op_statistic` 的 Top OP、`kernel_details` 的 Duration 和 Wait Time 相互印证。

空泡可能来自 Host 等待、数据准备、同步或调度间隙。Trace 只能证明“这里存在间隙”，还需要结合其他文件判断来源。

## Prefill 与 Decode 的定位重点

| 对比项 | Prefill | Decode |
| --- | --- | --- |
| 阶段作用 | 处理完整 Prompt | 自回归逐 token 生成 |
| 计算形态 | 单次大块计算 | 短步骤重复多次 |
| 主要特征 | 输入长、矩阵和 Attention 计算集中 | 每步新增一个 token，易受访存和调度影响 |
| 核心指标 | TTFT | TPOT、TPS |
| 重点算子 | MatMul、Attention | 重复小算子、KV Cache 相关操作 |
| 重点观察 | 输入长度、计算热点、算力利用率 | 步均耗时、波动、调度间隙、KV Cache 读取 |
| 瓶颈倾向 | 计算密集，长输入时矩阵乘法开销大 | 访存密集和调度开销，生成步数会放大问题 |

实验设计原则：

- 改变输入长度，主要观察 Prefill 和 TTFT 的变化。
- 改变 `max_new_tokens`，主要拉长 Decode 的观察窗口。
- 分析时先确认阶段归属，再选择定位策略。

## 常见性能瓶颈的识别与定位

使用“现象 --> 证据 --> 下一步”的链条快速确定排查方向。

| 现象 | Profiling 证据 | 下一步 |
| --- | --- | --- |
| Prefill 慢 | MatMul / Attention 在 `op_statistic` 中占比高；Prefill Trace 中 computing 区域长 | 检查融合算子、执行模式和输入 shape 是否合理 |
| Decode 慢 | Decode step 均值高或波动大；`kernel_details` 中同类小 kernel 反复出现 | 检查小算子调度、KV Cache 访存和 stream 同步方式 |
| Wait 高 / 空泡多 | Trace 中存在明显间隙；`kernel_details` 中 Wait Time 偏大 | 检查 Host 下发速度、CPU/NPU 同步点和 stream 配置 |
| 结果波动大 | 同配置多次运行指标差异明显；Trace 结构不稳定 | 固定 warmup 轮次，排除编译干扰，确认环境一致性 |

### 排查原则

```text
先归类现象 --> 再查聚合统计 --> 下钻单个 kernel --> 用 Trace 验证 --> 再行动
```

不要跳过证据直接调整参数。优化完成后必须回到相同 Baseline 条件复测，形成闭环。

## 性能分析记录模板

### 实验配置

| 项目 | 记录值 |
| --- | --- |
| 模型与精度 |  |
| 执行模式 |  |
| batch |  |
| 输入长度 |  |
| 输出长度 / `max_new_tokens` |  |
| warmup 轮次 |  |

### 性能指标

| TTFT | TPOT | TPS | Throughput | Average Inference Time |
| ---: | ---: | ---: | ---: | ---: |
|  |  |  |  |  |

### 证据链

```text
现象：
日志指标：
op_statistic 证据：
kernel_details 证据：
Trace 证据：
瓶颈假设：
本次唯一修改项：
复测结果：
结论：支持 / 推翻假设
```

## 核心结论

1. Profiling 的目标是建立可解释、可复现的性能证据链。
2. Prefill 和 Decode 的计算特征不同，必须分阶段分析。
3. 日志用于建立低开销基线，Profiler 用于下钻具体瓶颈。
4. `op_statistic` 看宏观热点，`kernel_details` 看微观实例，Trace 看时间线和空泡。
5. 一次只修改一个变量，并在相同条件下复测，才能验证优化效果。
