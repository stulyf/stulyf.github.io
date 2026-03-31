---
layout: post
title: LLM 造数据的经验
date: 2026-03-31 16:02:00
description: 关于llm实践中造数据的一些经验
tags: [data, LLM]
categories: [data]
thumbnail: assets/img/data_construct.png
featured: true
toc:
  sidebar: left
---

{% include figure.liquid loading="eager" path="assets/img/data_construct.png" class="img-fluid rounded z-depth-1" alt="LLM 造数据流程" caption="LLM 造数据的整体流程" %}

1. 造业务数据 :  self-instruct

种子数据集 ：人工先生成少量精炼数据 (seed instructions)

问题生成 ：llm根据种子问题 生成问题 然后进行质量过滤 

问题格式去重 ： 基于长度，字符合法性，正则规则（重复性），去重

问题语义去重 ： 基于embedding相似度去重

生成答案 ： llm基于生成的问题给出回答

答案格式去重 ： 基于长度，字符合法性，正则规则（重复性），去重

答案上llm as judge (不必要） ： 根据相关性，完整性，正确性，清楚性进行打分

合并数据集 ：

继续重复上述过程 

2. 思维链数据 ： cot蒸馏

给问题，模型采样回答，用问答对直接作为sft数据 ： 可能存在幻觉，回答不正确

给问题，和答案，模型补充推理过程 ： 不符合人类流程 ，同时也违反类似于GPT的生成模式

给问题，模型进行多次采样，选择其中回答正确的加入数据集 ： 成本较高，但是质量较好

3. 造DPO数据集

直接拿api 通过instruct分别生成好坏数据集  ：模型能力得不到真的提升

对于有ground truth 的任务，调高温度，基于self-consistency的方式采样多次，然后过滤掉模型不能回答正确的问题或者是模型全部都回答对的问题，对剩下的数据集进行规则过滤，尽量构建多维度差异较大的数据集

对于无ground truth 的模型，可以用参数量较大和当前的模型分别采样，然后用embedding 模型分别编码，基于embedding对数据集进行相似度阙值过滤，最后还可以上一层 llm as judge 去进一步提炼数据

4. 一些其他的业务数据处理经验 

sft 会比 pretrain 阶段 引入未见过的 special token , 如 system, user ,assistant 等，还可以针对业务引入新的special token，来区分不同角色

一般sft的数据是 prompt 不算 los，因为prompt 存在大量重复性语句，会导致模型反复学习一段话。多轮对话中的有两种计算loss的方式，对每个answer都计算 loss ,对最后一轮的 answer 计算loss，

sft要区分task_type, 根据任务难度去分配sft数据量，难的任务可以长一些，不要平均注意

数据的多样性非常重要，数据分布不能让模型找到规律，避免模型学到特定的位置分布或者token分布，从而避免模型在训练时退化

5. 造 rag 的 sft数据

关注检索为空的时候模型怎么回复 ， 避免模型自由发挥直接虚空回答

关注检索内容矛盾的情况，回答轨迹中不要只看到其中一条，而是真的对每个内容进行了分析

检索内容和query无关的场景，也需要让模型见过

检索内容错误的场景，需要让模型还是根据检索内容，让模型遵守检索到的证据，这是rag的核心能力。

6. 如何判断基座模型是否有某个task_type的能力

少量特定task_type 数据训多个epoch,然后观察是否正确率显著上升，如果还是没怎么变，说明基座模型就不具备这个能力。