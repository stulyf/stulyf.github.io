---
layout: post
title: Claude Code 源码关注的一些细节
date: 2026-04-01 00:00:00 +0800
description: 关于claude code源码中一些值得关注的设计细节
tags: [Claude, LLM, Agent]
categories: [Agent]
thumbnail: assets/img/claudecode_analysis.png
featured: true
toc:
  sidebar: left
---

{% include figure.liquid loading="eager" path="assets/img/claudecode_analysis.png" class="img-fluid rounded z-depth-1" max-width="600px" alt="Claude Code 源码分析" caption="Claude Code 源码分析" %}

## 1. 上下文管理和压缩机制

- 上下文管理解决的核心问题 ： 消除模型由于多轮对话积累导致的超出上下文限制的问题
- 上下文压缩的五个环节 ： 丢弃上摘要前的原始历史 -> 估计并且裁切工具输出 -> 删掉历史内容 -> 清除旧工具内容的结果 -> 自动摘要压缩
- 自动摘要压缩包含 ：

（1） 把整个对话发给claude ，让他写一份总结

用户要做什么 / 涉及了什么技术 / 操作了什么文件 / 遇到了什么错误，怎么修的 / 用户说了什么 / 还有什么没做完 / 当前正在做什么 / 下一步打算做什么
上述过程中，先让claude在< analysis >中打草稿思考，然后再在< summary >中写正式的总结 , 只保留 < summary >的方法

（2） 用总结替换原本的所有信息

（3）继续对话

- 补救压缩方式 ： 按照API轮次分组，从最旧的组开始丢弃，直到总的token数降到限制以下

## 2. 提示词

见 [Claude Code Prompts](https://github.com/stulyf/claude-code-prompts)

## 3. skill 管理 / 加载 / 运行 /生成

- skill 本质 ： 一个基于经验的可复用提示词 ，包含了明确的when-to-do;合适的allowed-tools;明确的实现步骤；边界和规则（不要做什么）；允许调整的参数
- skill 管理 ： 三级目录 ： 系统自带目录 / 用户个人目录 / 项目目录
- skill 加载 ： 先注册系统自带skill -> 扫描磁盘目录寻找skill -> 合并所有来源的 skill
- skill 执行 ： 两种执行方式 ： incline / fork，这两个是作为skill的属性被写死在创建中的

inline : 应用于较为简单的任务，会将skill的prompt直接注入到当前对话，模型在当前上下文下去运行，中间过程全部保留在主对话中

fork : 应用于较为复杂的任务，通过启动子代理运行，中间过程不进入主对话，主对话只收到最终结果的一句话总结

- skill 生成 ： 手动写 / 基于claude code 内部的 skillify的内置skill, 会引导你提炼出skill / 通过插件安装

## 4. agent memory （claude code 中的记忆系统设置）

- CLAUDE.md ： 项目规则的一些设置，分为 系统级/用户级/项目级/本地级
- Auto Memory ：长期记忆（不能从代码中分析得到的内容），记忆被分为四类 ： user用户的偏好 / feedback用户的反馈和纠正 / project项目特有知识 / reference项目某个内部api的调用方式
- 长期记忆的创建方式 ： 模型主动写入 / 自动提取 ：每轮对话结束后，系统会在后台启动一个子代理来提取记忆 / 自动整理 ： 满足一定的时间间隔或者是会话数之后，系统启动一个记忆整理流程，通过读取现有的所有记忆文件，进行合并重复，删除过时内容
- 短期记忆（上下文管理） ： 为每个对话窗口维护一个summary.md ，当token数超过阙值或者足够多的工具调用时，启动一个后台子代理分析当前对话并且更新summary.md内容；相当于边聊边更新summary，然后到达上下文限制时会调这个summary取代之前的上下文

## 5. 多智能体协作方式

三种多智能体协作方式 ：

- Fork 复制自己去干活 ：完整继承上下文，共享prompt cache
- Subagent 启动一个有独立身份的Agent : 全新开始 ；专家prompt
- **Coordinator模式 ： 有协调代理分配任务，启动多个worker 代理，直到worker通过 task-notification通知协调代理任务已经完成，协调代理才会根据结果向用户报告**

协调器模式要用户自己制定 ； Fork模式和Agent模式的选择是模型自己决定的
