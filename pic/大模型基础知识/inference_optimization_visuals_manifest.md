# 推理优化配图清单

生成方式：确定性 SVG 排版（未使用生成式图片模型）  
视觉风格：白底、讲义式布局、红/蓝/绿/橙强调色、中文无衬线字体  
画布：1440 px 宽，可无损缩放  

| 文件 | 内容 | 尺寸 |
| --- | --- | --- |
| `operator_fusion.svg` | 不融合、RMSNorm内部融合、Add+RMSNorm完全融合对比 | 1440 × 760 |
| `graph_mode.svg` | Eager模式与图模式执行流程对比 | 1440 × 690 |
| `kv_parallel_optimization.svg` | Paged Attention、KV Cache量化与多卡矩阵切分 | 1440 × 760 |

内容来源：用户提供的推理优化知识点截图；公式、标签和流程由对应Markdown正文校准。  
