---
layout: blog_post
title: "OPD 基础介绍：为什么 LLM RL 训练会悄悄变成 Off-Policy"
date: 2026-06-03
permalink: /blog/opd-basic-introduction/
excerpt: "一篇关于 Off-Policy Drift 的入门笔记：它是什么，为什么会出现在 LLM RL 中，以及我们可以如何诊断和缓解。"
---

在 LLM 的 post-training 中，我们常常会说某个 RL 算法是 “on-policy” 的：模型先生成回答，再根据这些回答计算 reward 和 policy gradient，然后更新自己。听起来数据来自当前模型，所以训练应该是 on-policy 的。

但在真实的大规模 LLM RL 系统里，事情没有这么干净。生成样本的 policy 和真正用于计算梯度的 policy 之间，经常会出现细微但持续累积的差异。这种差异可以理解为 **Off-Policy Drift，简称 OPD**。

## 1. 什么是 Off-Policy Drift？

最直观地说，OPD 指的是：

> 用来生成数据的模型分布，和用来训练更新的模型分布，不再完全一致。

在强化学习里，我们通常区分两个 policy：

- **Behavior policy**：实际采样数据的 policy，也就是 rollout 时生成 response 的模型。
- **Target / training policy**：训练时被优化的 policy，也就是计算 log probability、loss 和 gradient 的模型。

如果这两个 policy 完全一致，训练更接近 on-policy。  
如果它们不一致，训练就会带有 off-policy 成分。

对 LLM 来说，一条 response 是 token 序列。假设 rollout policy 生成了 token $y_t$，训练时当前 policy 也会给这个 token 一个概率。两者的比例可以写成：

$$
r_t = \frac{\pi_\theta(y_t \mid x, y_{<t})}{\mu(y_t \mid x, y_{<t})}
$$

其中 $\mu$ 是生成数据的 behavior policy，$\pi_\theta$ 是训练时的 policy。  
如果 $r_t$ 长期偏离 1，说明训练看到的数据和当前 policy 的真实分布之间出现了 mismatch。

## 2. 为什么 LLM RL 里会出现 OPD？

在小规模教科书式 RL 里，我们可能默认采样和训练发生在同一个模型、同一个环境、同一个时间点。但现代 LLM RL pipeline 通常更复杂。

常见来源包括：

### 2.1 rollout 和 training 使用不同引擎

生成 response 时，系统可能使用高吞吐 inference engine；训练时则使用另一个 training framework。即使它们加载的是“同一份权重”，由于精度、kernel、并行方式、attention 实现不同，log probability 也可能不完全一致。

这类差异看起来像工程细节，但它会直接影响 policy ratio，进而影响 gradient。

### 2.2 权重同步存在延迟

大规模训练为了提高吞吐，常常让 rollout worker 和 trainer 异步运行。rollout worker 可能还在用旧权重生成样本，而 trainer 已经更新了几步。

于是，一个 batch 里的 response 可能来自几步之前的 policy。等它被拿来训练时，当前 policy 已经变了。

### 2.3 同一个 batch 被训练多轮

PPO 类算法常会对同一批 rollout 数据做 multiple epochs。第一轮更新时，这些数据还比较“新鲜”；后面几轮时，policy 已经被更新过，数据就逐渐变旧。

这也是一种 staleness。

### 2.4 MoE、量化和非确定性也会放大差异

如果模型使用 MoE routing、FP8/INT8 量化、不同 batch size 下的非确定性 kernel，同一个 prefix 下的 log probability 都可能发生微小偏移。单个 token 的差异也许不大，但长序列上会累积。

## 3. OPD 为什么危险？

OPD 的核心问题不是“数据旧了一点”本身，而是：**gradient 可能不再准确反映当前 policy 应该如何改进。**

当训练 policy 和 rollout policy 差别较大时，模型可能会：

- 过度相信一些当前 policy 其实很少会生成的 token；
- 对 stale samples 给出过大的更新；
- 让 importance ratio 爆炸或塌缩；
- 使 PPO clipping 频繁触发，导致有效学习信号变弱；
- 在极端情况下造成 RL training collapse。

可以把它想象成：你根据一个旧版本模型的行为轨迹，去惩罚或奖励一个已经变化了的新版本模型。如果这两个版本差异不大，问题还可以接受；如果差异持续累积，训练信号就会越来越偏。

## 4. 如何诊断 OPD？

一个最直接的诊断方式是看 **log probability mismatch**。

比如，对于 rollout 中的每个 token，我们记录：

- rollout 时 behavior policy 给出的 log probability；
- training 时当前 policy 重新计算出的 log probability；
- 两者的差值或 ratio。

常见指标包括：

### 4.1 per-token log-ratio

$$
\log r_t = \log \pi_\theta(y_t \mid x, y_{<t}) - \log \mu(y_t \mid x, y_{<t})
$$

如果 $\log r_t$ 的分布很宽，说明 policy mismatch 比较严重。

### 4.2 sequence-level ratio

一整条 response 的 mismatch 可以粗略看作 token-level ratio 的累积：

$$
\log r = \sum_t \log r_t
$$

长序列里，即使每个 token 的漂移很小，累积后也可能很大。

### 4.3 clipping rate

如果 PPO 的 ratio 经常被 clip，说明 policy 更新已经频繁触碰 trust region。高 clipping rate 不一定全是 OPD 导致的，但它是一个很值得看的 warning signal。

### 4.4 stale step / policy lag

记录每个 sample 生成时的 policy step，以及它被训练消费时的 policy step。两者差值越大，sample 越 stale。

## 5. 如何缓解 OPD？

OPD 很难完全消除，但可以被监控和控制。

### 5.1 更频繁地同步权重

让 rollout worker 更快拿到最新 policy，可以减少 policy lag。代价是系统吞吐可能下降。

### 5.2 使用 importance sampling correction

如果我们知道 behavior policy 和当前 policy 的概率比例，就可以用 importance weight 修正 off-policy bias。简单说，就是让训练时意识到：“这个 token 是旧 policy 生成的，当前 policy 对它的概率已经变了。”

但 importance sampling 也有风险：ratio 太大时会引入高方差，所以通常还需要 clipping、normalization 或 rejection sampling。

### 5.3 控制同一批数据的重复训练次数

减少 multi-epoch update 可以降低 stale data 的使用程度。代价是 sample efficiency 可能下降。

### 5.4 监控训练和 inference 的 logprob 一致性

如果 rollout engine 和 training engine 对同一批 token 算出的 logprob 差异很大，说明系统层面已经引入 mismatch。这个问题应该先被当作 infrastructure bug 或 numerical mismatch 来排查，而不只是算法问题。

## 6. OPD 和 on-policy / off-policy 的关系

OPD 的一个重要启发是：LLM RL 里的 “on-policy” 不是一个二元标签，而是一个程度问题。

一个训练系统可以在算法设计上声称 on-policy，但在实际执行时因为异步、量化、重复训练、推理训练不一致等因素，表现出 off-policy 特征。

所以更有用的问题不是：

> 这个算法是不是 on-policy？

而是：

> rollout data 和 training policy 的 mismatch 有多大？这个 mismatch 是否被测量、限制和校正？

## 7. 小结

OPD 可以理解为 LLM RL 中的 policy 分布漂移问题。它来自 rollout policy 和 training policy 之间的差异，可能由异步训练、权重延迟、引擎差异、量化、multi-epoch update 等因素引入。

它之所以重要，是因为 policy gradient 对数据分布非常敏感。看似很小的 logprob mismatch，在长序列和大规模训练中可能累积成严重的优化偏差。

对我来说，OPD 最值得记住的一句话是：

> LLM RL 的稳定性不仅取决于 reward 和 loss，也取决于系统是否真的在用“当前 policy 的数据”训练当前 policy。

## References

- Chris Liu, [Off-Policy Drift in LLM RL](https://chrisliu298.ai/posts/off-policy-drift-in-llm-rl/), 2026.
