# 大模型量化配图清单

## 设计约束

- 使用可维护的 SVG，确保数值、公式标签和中文文本可精确控制。
- 避免信息卡片式框图，优先使用数轴、格点、矩阵分区、柱形分布和数据块等示意关系。
- 画布为白底，沿用现有笔记中的红、蓝、绿、紫配色和 Microsoft YaHei 字体回退。

## 文件

| 文件 | 说明 | 关键内容 |
| --- | --- | --- |
| `01_precision_compression.svg` | 位宽与权重体积 | 235B 参数在 BF16、INT8、INT4 下的理论体积 |
| `02_quantization_granularity.svg` | 量化粒度 | per-tensor、per-token、per-channel、per-group、per-block |
| `03_symmetric_quantization.svg` | 对称量化 | 浮点数轴到 signed INT8 `[-127,127]` 的映射 |
| `04_asymmetric_quantization.svg` | 非对称量化 | 浮点 0 映射到 zero-point `-23` |
| `05_quantization_error.svg` | 误差来源 | Round、Clip、Outlier |
| `06_smoothquant.svg` | SmoothQuant | 激活离群值向权重迁移的等价变换 |
| `07_microscaling.svg` | Microscaling | 全局 scale 与 block-wise scale 的差异 |

## 数值来源与校验

- 对称量化示例：`scale = 10.8 / 127 ≈ 0.0850`，`5.47 -> 64 -> 5.44`。
- 非对称量化示例：`scale = 18.39 / 255 ≈ 0.0721`，`zero-point = -23`，`5.47 -> 53 -> 5.48`。
- 所有 SVG 均已通过 XML 解析、viewBox 坐标边界和 Markdown 路径存在性检查。

## 参考

- 用户提供的量化课程截图用于确定内容顺序与视觉语气。
- 算法和格式定义以正文“参考资料”中的原论文和 OCP 规范为准。
