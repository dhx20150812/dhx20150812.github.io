---
layout:     post
title:      论文笔记
subtitle:   Sentence Level Content Planning and Style Specification for Neural Text Generation
date:       2020-11-18
author:     dhx20150812
header-img: img/post-bg-ios9-web.jpg
catalog: true
mathjax: true
tags:
    - NLP
    - Argument Generation
    - Content Planning
---

# Sentence-Level Content Planning and Style Specification for Neural Text Generation

> 来自EMNLP 2019，<https://www.aclweb.org/anthology/D19-1055.pdf>
>
> 代码已开源：<http://xinyuhua.github.io/Resources/emnlp19/>

##  Introduction

文本生成需要解决三个关键的问题：

（1）`content selection`——识别要呈现的相关信息

（2）`text planning`——将内容组织成有序的句子

（3）`surface realization`——决定能够提供连贯输出的单词和句法结构

传统的文本生成系统通常单独处理每个组件，因此需要在数据采集和系统工程方面进行大量的工作。端到端训练的神经网络模型可以生成流畅的文本，但受限于模型的结构和训练目标，他们往往缺乏可解释性，且生成的文本不连贯。

为了解决这个问题，作者认为，神经网络模型必须对 `content planning` 进行充分控制，以便产生一致的输出，尤其是对于长文本生成。同时，为了达到预期的语义目标，通过明确地建模和指定适当的语言风格，有助于实现风格控制的 `surface realization`。

<img src="https://note.youdao.com/yws/api/personal/file/WEB6e1dcad63165cd7dbb797c0fc3cfef43?method=download&shareKey=f638258f272c5d62e3cb0e11b3f18e81" alt="image-20201118195849944" style="zoom:50%;" />

例如，在“美国是否应该完全切断对外援助”这个话题上，上图展示了人类如何挑选一系列论点和适当的语言风格。辩论以“对外援助可以作为政治谈判筹码”为命题，然后是涵盖了几个关键概念的具体例子。最后是以辩论风格的语言结束。

因此，作者提出了一个端到端训练的神经文本生成框架，包括对传统生成组件的建模，以促进对生成文本内容和语言风格的控制。作者提出的模型使用了句子级别的 `content planning` 来进行信息的选择和排序，然后使用风格可控的 `surface realization` 来产生最终的输出。

![image-20201118195833139](https://note.youdao.com/yws/api/personal/file/WEB692f57f44765614bf437edfda740ff33?method=download&shareKey=846ddcfefc142200b0a34322df1af374)

如上图所示，模型的输入包括一个主题语句和一组关键短语。输出是一个相关且连贯的段落，以反映输入中的要点。作者使用了两个独立的解码器，对于每个句子：（1）`text planning` 解码器根据先前的选择来选择相关的关键短语和所需的风格（2）`surface realization` 解码器以指定的样式生成文本。

作者在三个任务上进行了实验：（1）`Reddit CMV` 的论点生成任务（2）维基百科的引言段落生成（3）`AGENDA` 数据集上的科研论文摘要生成。实验结果表明，在三个数据集上，作者提出的模型比非检索的方式取得了更高的 `BLEU`、`ROUGH` 和 `METEOR`。

## Model

模型的输入包括两部分：（1）主题陈述 $\mathbf{x}=\\{x_{i}\\}$。它可以是论点，可以是维基百科的文章标题，也可以是论文标题。（2）关键词记忆单元，$\mathcal{M}$，它包含话题要点列表，在 `content planning` 和风格选择中起着关键作用。目标输出是一个序列 $\mathbf{y}=\\{y_{t}\\}$，它可以是一个反驳的论点，维基百科文章中的一段话，或者一篇论文摘要。

### 输入编码

输入的文本 $\mathbf{x}$ 通过双向LSTM，它最后一层的隐藏层用于 `content planning` 解码器和  `surface realization` 解码器的初始状态。为了编码记忆单元 $\mathcal{M}$ 中的关键词组，每个关键词首先将它的所有单词的GLOVE词向量中求和，转换成一个向量 $e_{k}$。基于双向LSTM的关键词读取器采用隐藏状态 $\boldsymbol{h}_{k}^{e}$ 对 $\mathcal{M}$ 中的所有关键短语进行编码。作者还在 $\mathcal{M}$ 中插入`<START>`和`<END>`标记，以便于学习何时开始和完成选择。

### Content Planning

`Content planning` 基于前面的句子中已选择的关键词短语，从 $\mathcal{M}$ 中为每个句子（以 $j$ 为索引）选择一组关键词短语，从而实现主题连贯性和避免内容重复。这个选择器表示为一个选择向量 $\boldsymbol{v}\_{j} \in \mathbb{R}^{\|\mathcal{M}\|}$，它的每一维 $\boldsymbol{v}\_{j, k} \in\{0,1\}$ 表示第 $k$ 个短语是否被选择为生成第 $j$ 个句子。

作者使用了句子级别的 LSTM 网络 $f$，根据选择的短语词向量之和 $\boldsymbol{m}_{j}$，产生第 $j$ 个句子的隐层状态：

$$
\begin{array}{l}
\boldsymbol{s}_{j}=f\left(\boldsymbol{s}_{j-1}, \boldsymbol{m}_{j}\right) \\
\boldsymbol{m}_{j}=\sum_{k=1}^{|\mathcal{M}|} \boldsymbol{v}_{j, k} \boldsymbol{h}_{k}^{e}
\end{array}
$$

在实际中，多次使用的短语应该避免被再次选中，因此，作者提出了一个向量 $\boldsymbol{q}_{j}$，它用来追踪到第 $j$ 个句子时哪些短语已经被选择过：

$$
\boldsymbol{q}_{j}=\left(\sum_{r=0}^{j} \boldsymbol{v}_{r}\right)^{T} \times \mathbb{E}
$$

其中，$\mathbb{E}=\left[\boldsymbol{h}\_{1}^{e}, \boldsymbol{h}\_{2}^{e}, \ldots \boldsymbol{h}\_{\|\mathcal{M}\|}^{e}\right]^{T} \in \mathbb{R}^{\|\mathcal{M}\| \times H}$ 是短语表示的矩阵。$H$ 是 LSTM 的隐层维度。因此，下一时刻的短语选择向量 $\boldsymbol{v}\_{j+1}$ 有下面的式子得到：

$$
P\left(\boldsymbol{v}_{j+1, k}=1 \mid \boldsymbol{v}_{1: j}\right)=\sigma\left(\mathbf{w}_{v}^{T} \boldsymbol{s}_{j}+\boldsymbol{q}_{j} \mathbf{W}^{c} \boldsymbol{h}_{k}^{e}\right)
$$

作为学习目标的一部分，作者采用了交叉熵来作为评价准则：

$$
\mathcal{L}_{\mathrm{sel}}=-\sum_{(\mathbf{x}, \mathbf{y}) \in D} \sum_{j=1}^{J}\left(\sum_{k=1}^{|\mathcal{M}|} \log \left(P\left(\boldsymbol{v}_{j, k}^{*}\right)\right)\right)
$$

### Style Specification

人们通常会为不同的语言目标选择使用不同的语言风格。因此，作者为每个句子使用了一个风格变量 $\boldsymbol{t}\_{j}$，它由如下的公式预测得到：

$$
\hat{\boldsymbol{t}}_{j}=\operatorname{softmax}\left(\mathbf{w}_{s}^{T}\left(\tanh \left(\mathbf{W}^{s}\left[\boldsymbol{m}_{j} ; \boldsymbol{s}_{j}\right]\right)\right)\right.
$$

$\hat{\boldsymbol{t}}\_{j}$ 是所有类别上估计的概率分布。作者选择其中最大概率的风格，并用其 one-hot 表示 $\boldsymbol{t}\_{j}$ 作为` realization decoder` 的输入。然后将其与真实标签 $\boldsymbol{t}\_{j}^{*}$ 一起计算交叉熵损失：

$$
\mathcal{L}_{\text {style }}=-\sum_{(\mathbf{x}, \mathbf{y}) \in D} \sum_{j=1}^{J} \boldsymbol{t}_{j}^{*} \log \hat{\boldsymbol{t}}_{j}
$$


### Surface Realization

作者用 LSTM 网络实现了 `surface realization` 解码器，为了计算得到第 $t$ 个词的隐层状态 $\boldsymbol{z}\_{t}$，LSTM 结合了 `content planning` 解码器的隐层状态 $\boldsymbol{s}\_{J(t)}$ 和上一个词$\boldsymbol{y}\_{t-1}$ ，公式如下：

$$
\boldsymbol{z}_{t}=g\left(\boldsymbol{z}_{t-1}, \tanh \left(\mathbf{W}^{w s} \boldsymbol{s}_{J(t)}+\mathbf{W}^{w w} \boldsymbol{y}_{t-1}\right)\right)
$$

对于单词预测，作者考虑了两种注意力。一个关于输入的陈述 $\mathbf{x}$，它产生了上下文向量 $c_{t}^{w}$；另一个关于 `keyphase memory bank` $\mathcal{M}$，它生成 $\boldsymbol{c}\_{t}^{e}$。同时为了更好地发挥语言风格对选词的控制作用，作者直接将预测的句子风格拼接到隐层状态 $\boldsymbol{z}\_{t}$ 上，然后计算在词典上的概率分布：

$$
\begin{array}{l}
P\left(y_{t} \mid y_{1: t-1}\right)=\operatorname{softmax}\left(\tanh \left(\mathbf{W}^{o}\left[\boldsymbol{z}_{t} ; \boldsymbol{c}_{t}^{w} ; \boldsymbol{c}_{t}^{e} ; \boldsymbol{t}_{J(t)}\right]\right)\right) 
\end{array}
$$

$$
\begin{aligned}
\boldsymbol{c}_{t}^{w} &=\sum_{i=1}^{L} \alpha_{i}^{w} \boldsymbol{h}_{i}, \quad \alpha_{i}^{w}=\operatorname{softmax}\left(\boldsymbol{z}_{t} \mathbf{W}^{w a} \boldsymbol{h}_{i}\right) \\
\boldsymbol{c}_{t}^{e} &=\sum_{k=1}^{|\mathcal{M}|} \alpha_{k} \boldsymbol{h}_{k}^{e}, \quad \alpha_{k}^{e}=\operatorname{softmax}\left(\boldsymbol{z}_{t} \mathbf{W}^{w e} \boldsymbol{h}_{k}^{e}\right)
\end{aligned}
$$


### Training Objective

作者将三个损失函数整合到一起：

（1）单词预测 $\mathcal{L}\_{\mathrm{gen}} = -\sum_{D} \sum_{t=1}^{T} \log P(y_{t}^{*} \mid \mathbf{x} ; \theta)$

（2）关键词选择 $\mathcal{L}_{\mathrm{sel}}$

（3）风格预测 $\mathcal{L}_{\text {style }}$

因此，总的损失函数是：

$$
\mathcal{L}(\theta)=\mathcal{L}_{\mathrm{gen}}(\theta)+\gamma \cdot \mathcal{L}_{\text {style }}(\theta)+\eta \cdot \mathcal{L}_{\operatorname{sel}}(\theta)
$$

为了简单起见，在实验中作者将 $\gamma$ 和 $\eta$ 设置为1.0。

## Experiment Setup

### Tasks and Datasets

作者在三个任务上进行了实验，三个任务的描述如下：

（1）Argument Generation：这个任务的目标是为有争议的问题陈述生成反驳的观点。数据收集自`Reddit ChangeMyView` 数据集，每个帖子的原贴作为争议的问题，回帖中支持数大于反对数的作为目标的反驳观点。

在训练时，作者使用了目标观点来构建查询词检索维基百科的文章；在测试时，从输入的问题中构建查询词。然后基于 `topic signature words` 从检索的文章中抽取关键短语。

同时，作者为目标句子定义了三个风格：`CLAIM` 、`PREMISE` 和 `FUNCTIONAL` ，然后使用了规则对每个句子进行分类标注。

（2）Paragraph Generation：为维基百科的文章生成引言。输入包含了一个标题、用户指定的整体风格和一系列短语。作者从文章中抽取动名词短语作为关键词，同时根据句子长度对句子风格进行分类。


（3）Abstract Generation：为科技文章生成摘要。

### Baselines and Comparisons

对于三个任务，作者都实现了一个SEQ2SEQ的baseline，它将输入的文本和关键词集合一起编码，然后产生输出。

对观点生成的任务，作者实现了一个 `RETRIEVAL` 的baseline，它返回了将原贴作为输入时概率最高的文章。同时，作者还与之前的工作进行了比较。

对维基百科的生成任务，作者也实现了一个`RETRIEVAL` 的baseline，它基于余弦相似度返回了与输入标题和关键词最相似的文章。

对摘要生成的任务，作者将其与 SOTA 的 `GRAPHWRITER` 进行比较。

同时，作者还与模型的变体进行了比较：（1）对每个句子使用gold-standard的关键词选择（Oracle Plan）（2）没有指定风格

### Results and Analysis

（1）Argument Generation：由下图可见，作者提出的模型与其余的baseline相比都取得了更高的BLEU和ROUGE分数，同时也比生成式的方法产生了更长的文本。同时，在模型变体中，oracle 设置的模型提升了效果，这证明了 `content selection` 的重要性。当移除风格控制时，模型的分数降低，这证明了风格控制在生成时的作用。

![image-20201119155619494](https://note.youdao.com/yws/api/personal/file/WEB9dc0598486cfab3dc1fedb4f879e7fcb?method=download&shareKey=d3b0868bd56d75a7181f06d135c9f757)

（2）Wikipedia Generation：在维基百科数据集上的结果如下图所示。可见，当没有风格控制时，模型的效果显著降低。同时，oracle设置的模型变体取得了最高的分数。

![image-20201119160136658](https://note.youdao.com/yws/api/personal/file/WEB8a1008d9a0ea4901c1150789a44a4ee6?method=download&shareKey=82dea09b6a31e59ce239bca6841556cb)

同时为了展示 `content selection` 对生成质量的影响，作者绘制了关键词选择时的F1分数与生成效果之间的关系图。可见，两者有强烈的相关性。

![image-20201119160504622](https://note.youdao.com/yws/api/personal/file/WEB0e79fcc612842459e2472109baad1389?method=download&shareKey=6eb2f6e6e6fa5551c1aba8af4a10a031)

（3）Abstract Generation：

最后，作者将自己的模型与 `GRAPHWRITER` 进行了比较，实验结果表明自己的方法与其相比有不弱的表现。

![image-20201119160558288](https://note.youdao.com/yws/api/personal/file/WEB7c171c763d05e82aeb8375f1adb1a18e?method=download&shareKey=6964a29567994d3ca1d6d87227abbc51)
