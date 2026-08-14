# Notes

记录大模型基础、推理优化与量化等方向的学习笔记，内容持续更新。

## 内容目录

1. [大模型基础知识](./2.大模型基础知识/大模型基础知识.md)
   - Transformer、Attention、MLP与MoE
   - Prefill、Decode与KV Cache
2. [MLP运行示例](./2.大模型基础知识/MLP运行示例.md)
   - 3个输入token生成至5个token的完整计算示例
3. [大模型推理优化](./3.大模型推理优化/大模型推理优化基础.md)
   - 融合算子与图模式
   - Paged Attention与KV Cache量化
   - 多卡切分与多流并行
4. [大模型量化](./4.大模型量化/大模型量化基础.md)
   - 对称与非对称量化、量化粒度和误差来源
   - W8A8、SmoothQuant、PTQ与QAT
   - GPTQ、AWQ、QuaRot、MXFP8与MXFP4

## 文件夹结构

```text
Notes/
├── 2.大模型基础知识/
│   ├── 大模型基础知识.md
│   └── MLP运行示例.md
├── 3.大模型推理优化/
│   └── 大模型推理优化基础.md
├── 4.大模型量化/
│   └── 大模型量化基础.md
├── pic/
│   ├── 大模型基础知识/
│   │   ├── transformer_block.png
│   │   ├── attention_01.png
│   │   ├── operator_fusion.svg
│   │   ├── graph_mode.svg
│   │   ├── kv_parallel_optimization.svg
│   │   └── inference_optimization_visuals_manifest.md
│   └── 大模型量化/
│       ├── 01_precision_compression.svg
│       ├── 02_quantization_granularity.svg
│       ├── 03_symmetric_quantization.svg
│       ├── 04_asymmetric_quantization.svg
│       ├── 05_quantization_error.svg
│       ├── 06_smoothquant.svg
│       ├── 07_microscaling.svg
│       └── quantization_visuals_manifest.md
├── .gitignore
└── README.md
```

## 目录约定

- 编号目录用于区分学习主题及阅读顺序。
- Markdown笔记保存在对应主题目录中。
- 图片和其他可视化资源统一保存在`pic/对应主题/`目录中。
