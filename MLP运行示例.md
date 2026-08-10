# 3 Token 生成 5 Token 的完整计算示例

下面用一个极简 Transformer 演示：输入 3 个 token，模型再生成 2 个 token，最终序列长度为 5。

为了让数字容易计算，示例只使用：

- 1 层 Transformer
- 1 个 Attention Head
- 隐藏维度 $d=2$
- 贪心解码，每次选择概率最大的 token
- 省略 LayerNorm
- token 向量中已包含位置信息

最终目标：

```text
输入：我  喜欢  AI
生成：因为  强大
最终：我  喜欢  AI  因为  强大
```

## 一、输入向量

将前三个 token 转换为二维向量：

$$
x_1=x_{\text{我}}=[1,0]
$$

$$
x_2=x_{\text{喜欢}}=[0,1]
$$

$$
x_3=x_{\text{AI}}=[1,1]
$$

组成输入矩阵：

$$
X=
\begin{bmatrix}
1&0\\
0&1\\
1&1
\end{bmatrix}
\in\mathbb{R}^{3\times2}
$$

其中，3 表示 token 数量，2 表示每个 token 的特征维度。

## 二、计算 Q、K、V

为了简化，假设：

$$
W_Q=W_K=W_V=I
$$

因此：

$$
Q=XW_Q=X,\qquad K=XW_K=X,\qquad V=XW_V=X
$$

所以：

$$
Q=K=V=
\begin{bmatrix}
1&0\\
0&1\\
1&1
\end{bmatrix}
$$

此时 KV Cache 的维度为：

$$
K_{\text{cache}},V_{\text{cache}}\in\mathbb{R}^{3\times2}
$$

## 三、对三个输入 token 计算 Attention

使用因果掩码：

- “我”只能看“我”
- “喜欢”能看“我、喜欢”
- “AI”能看“我、喜欢、AI”

公式为：

$$
\operatorname{Attention}(Q,K,V)
=\operatorname{Softmax}\left(\frac{QK^T}{\sqrt2}\right)V
$$

### 1. token“我”

Query 为：

$$
q_1=[1,0]
$$

它只能看到自己：

$$
\frac{q_1k_1^T}{\sqrt2}=\frac{1}{\sqrt2}=0.707
$$

只有一个分数，因此 Softmax 权重为 $[1]$，Attention 输出为：

$$
c_1=1\times v_1=[1,0]
$$

### 2. token“喜欢”

Query 为：

$$
q_2=[0,1]
$$

它可以看到前两个 token：

$$
\frac{q_2K^T}{\sqrt2}=\frac{[0,1]}{\sqrt2}=[0,0.707]
$$

Softmax 后：

$$
\operatorname{Softmax}([0,0.707])\approx[0.330,0.670]
$$

因此：

$$
c_2=0.330v_1+0.670v_2
$$

$$
c_2=0.330[1,0]+0.670[0,1]=[0.330,0.670]
$$

### 3. token“AI”

Query 为：

$$
q_3=[1,1]
$$

它可以看到全部三个 token：

$$
q_3K^T=[1,1]
\begin{bmatrix}
1&0\\
0&1\\
1&1
\end{bmatrix}^{T}
=[1,1,2]
$$

缩放：

$$
\frac{q_3K^T}{\sqrt2}=[0.707,0.707,1.414]
$$

Softmax 后：

$$
\alpha_3\approx[0.248,0.248,0.504]
$$

含义是：对“我”的权重约为 24.8%，对“喜欢”的权重约为 24.8%，对“AI”的权重约为 50.4%。

加权汇总 Value：

$$
c_3=0.248v_1+0.248v_2+0.504v_3
$$

$$
c_3=0.248[1,0]+0.248[0,1]+0.504[1,1]
\approx[0.752,0.752]
$$

## 四、残差连接和 MLP

先进行 Attention 残差连接：

$$
u_i=x_i+c_i
$$

对于最后一个 token“AI”：

$$
u_3=[1,1]+[0.752,0.752]=[1.752,1.752]
$$

为了简化，假设 MLP 为：

$$
\operatorname{MLP}(u)=0.2\operatorname{ReLU}(u)
$$

因此：

$$
\operatorname{MLP}(u_3)=0.2[1.752,1.752]=[0.350,0.350]
$$

再进行一次残差连接：

$$
z_3=u_3+\operatorname{MLP}(u_3)
$$

$$
z_3=[1.752,1.752]+[0.350,0.350]=[2.102,2.102]
$$

这个向量用于预测第 4 个 token。

## 五、预测第 4 个 token

假设候选 token 为“因为、强大、。”，输出映射矩阵为：

$$
W_{\text{out}}=
\begin{bmatrix}
0.8&1&0\\
0.8&0&1
\end{bmatrix}
$$

每一列依次对应“因为、强大、。”。

计算 logits：

$$
\operatorname{logits}_4=z_3W_{\text{out}}
$$

$$
[2.102,2.102]
\begin{bmatrix}
0.8&1&0\\
0.8&0&1
\end{bmatrix}
=[3.364,2.102,2.102]
$$

Softmax 后大约为：

| 候选 token | 概率 |
| --- | ---: |
| 因为 | 63.8% |
| 强大 | 18.1% |
| 。 | 18.1% |

因此选择第 4 个 token“因为”，当前序列变为：

```text
我  喜欢  AI  因为
```

## 六、追加第 4 个 token 的 KV Cache

假设“因为”的输入向量为：

$$
x_4=[2,0]
$$

由于 $W_Q=W_K=W_V=I$，所以：

$$
q_4=k_4=v_4=[2,0]
$$

将新的 $k_4、v_4$ 追加到缓存：

$$
K_{\text{cache}}=V_{\text{cache}}=
\begin{bmatrix}
1&0\\
0&1\\
1&1\\
2&0
\end{bmatrix}
$$

矩阵维度从：

$$
3\times2\rightarrow4\times2
$$

发生变化的是 token 数量这一维，每个 Value 的特征维度仍然是 2。

## 七、第 4 个 token 计算 Attention

“因为”的 Query 为：

$$
q_4=[2,0]
$$

与 4 个 Key 计算点积：

$$
q_4K_{\text{cache}}^T=[2,0,2,4]
$$

缩放：

$$
\frac{q_4K_{\text{cache}}^T}{\sqrt2}
=[1.414,0,1.414,2.828]
$$

Softmax 后：

$$
\alpha_4\approx[0.157,0.038,0.157,0.647]
$$

然后加权汇总 Value：

$$
c_4=0.157v_1+0.038v_2+0.157v_3+0.647v_4
$$

$$
c_4=0.157[1,0]+0.038[0,1]+0.157[1,1]+0.647[2,0]
\approx[1.609,0.196]
$$

Attention 残差：

$$
u_4=x_4+c_4=[2,0]+[1.609,0.196]=[3.609,0.196]
$$

MLP：

$$
\operatorname{MLP}(u_4)=0.2\operatorname{ReLU}(u_4)
=[0.722,0.039]
$$

MLP 残差：

$$
z_4=u_4+\operatorname{MLP}(u_4)
=[4.331,0.235]
$$

## 八、预测第 5 个 token

继续使用同一个输出矩阵：

$$
\operatorname{logits}_5=z_4W_{\text{out}}
$$

$$
[4.331,0.235]
\begin{bmatrix}
0.8&1&0\\
0.8&0&1
\end{bmatrix}
=[3.653,4.331,0.235]
$$

Softmax 后大约为：

| 候选 token | 概率 |
| --- | ---: |
| 因为 | 33.3% |
| 强大 | 65.6% |
| 。 | 1.1% |

因此选择第 5 个 token“强大”，最终序列为：

```text
我  喜欢  AI  因为  强大
```

## 完整流程总结

```text
输入3个token
我  喜欢  AI
       ↓
Prefill：并行计算3个token的Q、K、V
       ↓
KV Cache长度 = 3
       ↓
使用“AI”位置的输出预测下一个token
       ↓
生成“因为”
       ↓
计算并追加“因为”的K、V
       ↓
KV Cache长度 = 4
       ↓
使用“因为”位置的输出预测下一个token
       ↓
生成“强大”
       ↓
最终序列长度 = 5
```

当“强大”刚被选出来时，它还没有经过下一轮 Transformer 计算。因此，如果生成到这里就停止，KV Cache 通常只有 4 个位置；只有继续预测第 6 个 token 时，才会计算并追加“强大”的 $K、V$。

真实大模型的维度、层数和注意力头更多，但核心生成循环与这个例子相同。
