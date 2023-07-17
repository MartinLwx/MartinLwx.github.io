# 用 MPNN 框架解读 GAT


## 什么是 MPNN 框架

Justin Gilmer 提出了 MPNN（Message Passing Neural Network）框架[^1] ，用于描述被用来做图上的监督学习的图神经网络模型。我发现这是一个很好用的框架，可以很好理解不同的 GNN 模型是如何工作的，方便快速弄清楚不同的 GNN 模型之间的差别。我们考虑图 $G$ 上的一个节点 $v$，它的向量表示 $h_v$ 的更新方式如下：
$$m_v^{t+1}=\sum_{u\in \mathcal{N}(v)}M_t(h_v^t,h_u^t,e_{vu})$$
$$h_v^{t+1}=U_t(h_v^t,m_v^{t+1})$$

其中
- $u$ 为 $v$ 的邻居节点，$\mathcal{N}(v)$ 则表示节点 $v$ 的所有邻居
- $e_{vu}$ 是可选项，表示边上的特征（若有）
- $M_t$ 是消息函数，$m_v^{t+1}$ 就是所有邻居节点的消息聚合之后的结果
- $U_t$ 是节点更新函数

在更新完图上所有节点的向量表示之后，我们可能需要要做图级别的分类任务，在 MPNN 框架中对应为：
$$\hat y=R(\{h_v^T|v\in G\})$$

其中：
- $R$ 为图读出函数，输入是图上所有节点的向量表示，输出为一个特征向量用于图级别的分类任务


## 什么是 GAT
> 🧐 我发现将论文的公式和具体的代码联系起来总是能够帮助理解，因此下面关于 GAT 公式的讲解我会在用 MPNN 框架的基础上，同时附上相关的代码（用 `...` 表示其他省略的部分）。代码来自于 DGL 官方提供的 GAT 模块的[源码](https://docs.dgl.ai/en/latest/_modules/dgl/nn/pytorch/conv/gatconv.html#GATConv)

> 🧐 GAT 可以方便堆叠多层，下面的讨论都是从**某一层 $l$**的**某个**节点 $v$ 的角度来谈的

#### Step 1. 对节点做线性变换
$$h_v^{l}=W^lh_v^{l}$$

设每个节点的向量表示的长度为 $F$，在第一步中，**对图上每个结点**的向量先做一个线性变换，其中 $W\in\mathcal{R}^{F'\times F}$, 因此每个节点的向量都更新长度为 $F'$。为了区分不同层的向量，用上角标 $l$ 表示这是第 $l$ 层的。注意**在第 $l$ 层中，所有节点共用同一个权重矩阵 $W$**

> 📒 **注意后面的 $h_v^l$ 或者 $h_u^l$ 都是经过线性变换之后的**

```python
class GATConv(nn.Module):
    def __init__(self, ...):
        ...
        self.fc = nn.Linear(
            self._in_src_feats, out_feats * num_heads, bias=False
        )
        ...
    def forward(self, graph, feat, ...):
        """
        Args
        ----
            feat: (N, *, D_in) where D_in is the size of input feature

        Returns
        -------
            torch.Tensor
                (N, *, num_heads, D_out)

        """
        ...
        src_prefix_shape = dst_prefix_shape = feat.shape[:-1]
        h_src = h_dst = self.feat_drop(feat)
        # h_src: (N, *, D_in)
        feat_src = feat_dst = self.fc(h_src).view(
            *src_prefix_shape, self._num_heads, self._out_feats
        )
        # feat_src/feat_dst: (N, *, num_heads, out_feat)
        ...
```

注意，上面代码中的 `h_src` 会出来两个一样的 `feat_src` 和 `feat_dst` 是因为 DGL 采用了数学上等价但是计算效率会更高的实现。后面会解释

#### Step 2. 计算注意力
$$e_{vu}^l=LeakyReLU\Big((a^l)^T[h_v^{l}||h_u^{l}]\Big)$$

$$\alpha_{vu}^l=Softmax_u(e_{vu}^l)$$

第二步就是要计算中心节点 $v$ 和它的所有邻居节点之间的注意力了。在上面的公式中：
- $e_{vu}^l$ 表示注意力系数（attention coefficient）。论文中提到完全可以采用不同的注意力计算方式，在 GAT 的论文中，作者选择用一层前馈神经网络计算注意力[^2]。**注意这里的 $e_{vu}^l$ 跟 MPNN 的 $e_{vu}$ 是没有关系的，只是恰好记号一样了而已**
- $||$ 表示拼接操作，即我们会将中心节点和它对应的邻居结点的向量表示拼接起来，得到一个长度为 $2F'$ 的向量，对应公式中的 $[h_v^{l}||h_u^{l}]$，然后把它送入到前面提到的一层前馈神经网络中，对应公式中的 $(a^l)^T[h_v^{l}||h_u^{l}]$，其中 $(a^l)^T$ 指的是第 $l$ 层的单层前馈神经网络的参数
- 激活函数选用的是 $LeakyReLU$
- 最后在节点 $v$ 的所有邻居节点之间用 Softmax 做归一化

> 🤔️ 步骤一和步骤二对应 MPNN 框架的 $m_v^{t+1}$ 的计算
##### 多头注意力
正如 Transformer 中有多头注意力一样，GAT 的作者同样在节点更新的时候采用了多头注意力的机制：
$$h_v^{l+1}= ||^{K^l} \sigma(\sum_{u\in\mathcal{N}(i)}\alpha_{vu}^{(k,l)}W^{(k,l)}h_u^{l})$$

记号越来越复杂了，但是仔细思索一番还是可以理清楚的，上角标 $(k,l)$ 的意思是第 $l$ 层的第 $k$ 个头。其中 $K^l$ 为第 $l$ 层“头”的数量。上面的公式的意思就是每个“头”会计算出一个向量表示，然后不同“头”之间的会拼接起来


```python
class GATConv(nn.Module):
    def __init__(self, ...):
        ...
        self.attn_l = nn.Parameter(
            th.FloatTensor(size=(1, num_heads, out_feats))
        )
        self.attn_r = nn.Parameter(
            th.FloatTensor(size=(1, num_heads, out_feats))
        )
        self.leaky_relu = nn.LeakyReLU(negative_slope)
        ...

    def forward(self, graph, feat, ...):
        """
        Args
        ----
            feat: (N, *, D_in) where D_in is the size of input feature

        Returns
        -------
            torch.Tensor
                (N, *, num_heads, D_out)

        """
        # feat_src/feat_dst: (N, *, num_heads, out_feat)
        el = (feat_src * self.attn_l).sum(dim=-1).unsqueeze(-1)
        er = (feat_dst * self.attn_r).sum(dim=-1).unsqueeze(-1)
        # el/er: (N, *, num_heads, 1)
        graph.srcdata.update({"ft": feat_src, "el": el})
        graph.dstdata.update({"er": er})
        graph.apply_edges(fn.u_add_v("el", "er", "e"))
        e = self.leaky_relu(graph.edata.pop("e"))
        # e: (N, *, num_heads, 1)

        # normalization
        graph.edata["a"] = self.attn_drop(edge_softmax(graph, e))
        # a: (N, *, num_heads, 1)

        # weighted sum
        graph.update_all(
            # ft: (N, *, num_heads, out_feat)
            # a: (N, *, num_heads, 1)
            # m: (N, *, num_heads, out_feat)
            fn.u_mul_e("ft", "a", "m"), 
            fn.sum("m", "ft")
        )
        rst = graph.dstdata["ft"]
        # rst: (N, *, num_heads, out_feat)
        ...
```
DGL 的实现基于下面这个事实：
$$a^T[h_v||h_u]=a_l^Th_v+a_r^Th_u$$
为什么会更为高效呢？
- 不需要存储 $[h_v||h_u]$ 这个中间变量了（DGL 会将消息存储为边的属性）
- 加法可以用 DGL 优化过的 `fn.u_add_v` 函数
#### Step 3. 聚合邻居消息

最后节点更新的方式就是计算邻居的向量表示的加权和（基于前面算好的注意力）：
$$h_v^{l+1}=\sigma(\sum_{u\in \mathcal{N}(i)}\alpha_{vu}W^{l}h_u^{l})$$

> 🤔️ 上面就对应 MPNN 框架中的 $h_v^{t+1}$ 的计算，注意 GAT 算 $h_t^{l+1}$ 的时候并不会用到自己上一层表示 $h_t^l$。同时 GAT 的提出是用于解决图上的节点分类问题，因此也没有图读出的操作。

在 GAT 的论文中，作者是要将 GAT 用于节点级别的分类任务[^2]。假设我们堆叠了 $L$ 层的 GAT 之后，在最后一层如果还采用拼接的方式显然是不合理的。因此作者在最后一层 GAT 中，是取多个头的平均值**之后才**应用了激活函数，这里的激活函数如果采用 Softmax，就可以直接做节点分类了[^2]。公式如下所示:
$$final\ embedding\ of\ h_v= \sigma\Big(\frac{1}{K^L}\sum_{k=1}^{K^L}\sum_{u\in\mathcal{N}(i)}\alpha_{vu}^{(k,L)}W^{(k,L)}h_u^{L}\Big)$$


## 实现

> 🤔️ 图神经网络的流行框架之一 [DGL](https://www.dgl.ai/) 的设计就是基于 MPNN 框架，不过他们的公式会稍有不同，他们还有一个聚合函数 $\rho$，用于决定一个节点如何聚合从邻居那边收到的所有信息。**我认为他们的公式更具有泛化性，能够适用于更多种情况**。他们还贴心地写了关于如何使用 DGL 的 MPNN 相关函数的[教程](https://docs.dgl.ai/en/latest/guide/message.html)，推荐一看👍👍👍

至于 GAT 的实现，DGL 不仅提供了 [GAT 模块](https://docs.dgl.ai/en/latest/generated/dgl.nn.pytorch.conv.GATConv.html#dgl.nn.pytorch.conv.GATConv)，而且还写了一篇不错的用自带的 `message_func` 和 `reduce_func` 手动实现 GAT 的[教程](https://docs.dgl.ai/en/latest/tutorials/models/1_gnn/9_gat.html)。一个完整的 GAT 任务训练脚本可以看[这里](https://github.com/dmlc/dgl/blob/master/examples/core/gat/train.py)

## 总结
以上就是如何用 MPNN 框架解释 GAT 的方法，其中加上了 DGL 的源码分析进行解释，用注意力计算节点之间的关系看起来是一个很自然的思路，可以看成是对 GCN 的一种泛化。GAT 能够很好学习到图的局部结构表示，而且计算注意力的方式可以并行，是十分高效的🍻🍻🍻

## 参考

[^1]: Gilmer J, Schoenholz S S, Riley P F, et al. Neural message passing for quantum chemistry[C]//International conference on machine learning. PMLR, 2017: 1263-1272. [arXiv](https://arxiv.org/abs/1704.01212)
[^2]: Veličković P, Cucurull G, Casanova A, et al. Graph attention networks[J]. arXiv preprint arXiv:1710.10903, 2017. [arXiv](https://arxiv.org/abs/1710.10903)

