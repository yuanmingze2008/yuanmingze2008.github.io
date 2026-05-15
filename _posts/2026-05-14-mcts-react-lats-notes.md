---
title: "MCTS ReAct LATS"
date: 2026-05-14 23:53:08 +08:00
permalink: /posts/mcts-react-lats-notes/
tags:
  - agent
  - llm
  - search
  - notes
note_group: "Agent Research"
summary: "围绕 MCTS、ReAct 与 LATS，记录搜索、行动、反馈和语言模型推理结合的基本框架。"
source_note: "local source file, not uploaded"
---

一种强大的**决策**算法，用于**在特定决策过程中做出最优选择**的启发式搜索算法。

The loop of Selection, Expansion, Simulation, Backpropagation, and final Decision.

每次 iteration 从 root 出发。

1. 在 Selection 阶段，如果当前节点已经完全展开且非终止，就用 UCT 在已有子节点中选择一个继续向下。
    一旦遇到尚未完全展开的非终止节点，就停止 Selection，进入 Expansion：选择一个尚未尝试的 action，创建新子节点。

2. UCT 在 Selection 阶段反复使用，直到到达可扩展节点或终止节点。
   $$
   \text{UCT}(v_i)=\frac{w_i}{n_i}+c_i\sqrt{\frac{\ln n_p}{n_i}}
   $$
   其中：

   - $v_i$ 是待评估的子节点。
   - $\frac{w_i}{n_i}$ 是该子节点的**平均奖励**（或平均价值），代表“**利用**”项。这部分鼓励算法选择历史表现好的节点。
   - $c\sqrt{\frac{\ln n_p}{n_i}}$ 是**探索项**。$c$ 是一个可调的探索常数，通常取值为 $\sqrt2$，$n_p$ 是其父节点的总访问次数，$n_i$ 是该子节点自身的访问次数。当 $n_i$ 很小时，这一项的值会很大，从而鼓励算法去探索访问次数少的节点。随着 $n_i$ 的增加，这一项的值会减小。
   - $c$ 为**探索常数**，用于调节探索与利用之间的权重。$c$ 越大，算法越倾向于探索。

3. 然后从该新子节点开始 Simulation，并将结果 Backpropagation 回 root。Simulation 是从节点向下做完一次随机选择的价值估计，Backpropagation 是基于该节点 reward 更新搜索树中该节点向 root 路径上的点。

这样做直到达到 run-time limit 或者 tries limit。

搜索预算耗尽后，退出循环，做一次 decision：从 root 的所有子节点中选择最终 action。注意：回忆 MCTS 的定义，MCTS 是决定落子，决定一次 action。然后根据环境变化（如对手落子或者新的状态）再次进行 MCTS，所以不会递归选择 arg max。

```python
def mcts_search(root_state, budget):
    root = Node(root_state)

    for _ in range(budget):

        # 1. Selection
        node = root

        while not node.is_terminal() and node.is_fully_expanded():
            node = select_child_by_uct(node)

        # 2. Expansion
        if not node.is_terminal():
            node = expand_one_untried_action(node)

        # 3. Simulation / Evaluation
        reward = rollout_or_evaluate(node.state)

        # 4. Backpropagation
        backpropagate(node, reward)

    # 5. Decision / Final Action Selection
    best_child = argmax(root.children, key=lambda child: child.visits)

    return best_child.action_from_parent
```

![Monte Carlo Tree Search | Wiki.js](https://wiki.neuro.tu-berlin.de/teaching/2025ss-algodat-se/algorithms/mcts/mcts-phasen.jpeg)

![repeated_x_times](https://media.geeksforgeeks.org/wp-content/uploads/20251108150801887143/repeated_x_times.webp)



# ReAct

> 2022 arxiv, 2023 ICLR, Prompt Engineering.

在面对稍有逻辑的问题，单纯的 CoT (Reason Only) 或者 Act-Only (Search, Observation) 很容易逻辑混乱，把某些相关元素的权重调大导致答案错误。

ReAct 将 Reasoning 和 Acting 结合，使 LLMs 以交错的方式生成**推理轨迹**和**任务特定操作** 。

- 生成推理轨迹使模型能够诱导、跟踪和更新操作计划，甚至处理异常情况。

- 操作步骤允许与外部源（如知识库或环境）进行交互并且收集信息，ReAct 框架也允许 LLMs 与外部工具交互来获取额外信息，从而给出更可靠和实际的回应。

结果表明，作者将 Reasoning, Acting, CoT 结合起来提出的 ReAct 除了在语言和决策任务上的表现高于当时先进水准，还提高了 LLMs 的人类可解释性和可信度。

通过下图可以看出，总体 workflow 就是链式多次的 Thought + Action + Observation，Thought 对任务进行指导和推理，Observation 对 Action 结果进行针对性 Reflection 并为下次 Thought 提供线索。



```
问题 科罗拉多造山带东部区域延伸到的区域的海拔范围是多少?
思考 1 我需要搜索科罗拉多造山带，找到科罗拉多造山带东段延伸的区域，然后找到该区域的海拔范围。
操作 1 搜索 [科罗拉多造山带]
观察 1 科罗拉多造山带是科罗拉多及其周边地区造山运动的一段。
思考 2 它没有提到东部地区。所以我需要查一下东部地区。
操作 2 查找 [东部地区]
观察 2 (结果 1 / 1) 东部区域延伸至高平原，称为中原造山带。
思考 3 科罗拉多造山运动的东部区域延伸到高平原。所以我需要搜索高平原，找到它的海拔范围。
操作 3 搜索 [高平原]
观察 3 高平原指的是两个截然不同的陆地区域之一。
思考 4 我需要搜索高平原（美国）
操作 4 搜索 [高平原（美国）]
观察 4 高平原是大平原的一个分区。从东到西，高平原的海拔从 1800 到 7000 英尺 (550 到 2130 米) 不等。[3]
思考 5 高平原海拔高度从 1800 到 7000 英尺，所以答案是 1800 到 7000 英尺。
操作 5 结束 [1800 到 7000 英尺]
...
```



![image-20260514165925237](/images/notes/mcts-react-lats-notes/image-20260514165925237.png)



# LATS

> 2023 arxiv, 2024 ICLR, Agentic Architecture.

观察 ReAct 的结构发现，其针对优化的是 LLMs 解决强逻辑问题的能力，本质还是链式的 CoT，一旦中间某一步产生幻觉就会全盘皆输。

而回顾广思慎行的 ToT，它的优势能弥补 ReAct 的劣势，但是它的劣势（BFS 和 DFS 在遇到复杂任务时，搜索空间过大无法承受）依然存在。

可以发现，LLMs 的输出其本质还是决策问题，面对搜索空间非常大的情况，我们考虑针对 ToT 使用蒙特卡洛树搜索。

- Selection：Agent 利用 UCB 平衡探索和利用。
- Expansion：在选定节点后，调用 LLM 生成 $k$ 个 Next Actions 以及对应的 Thoughts，也就是给出 $k$ 种方案（子节点）以供选择。
- Evaluation：LATS 将 LLM 作为一个价值函数，当 Agent 执行了已选的 Next Action，它会把当前环境状态打包喂给 LLM，让它输出一个分数。
- Backpropagation：基于打分反向传播更新该节点到 root 的 $V(s),N(s)$。

> 几个补充：
>
> 1. 在 LLM 介入的情况下，无法做到真正的“扩展完毕”，我们人为设定超参数 $k$ 作为扩展阈值。
>
> 2. 这里和传统 Monte Carlo 的 Simulation 有些出入，更应该称为 Evaluation.
>
>    为什么不在 next action 后随机 act 指导 finish，获得一个 answer，然后将 question 和 answer 合成一个陈述句喂给 LLM 去 evaluation 呢？
>
>    - Token 开销过高：如果你每次评估一个中间节点，都要让 LLM 一路模拟到任务结束，意味着树上的每一个节点都会引发一次完整对话。
>
>    - LLMs 幻觉导致的“噪声过大”：这是关键，假设当前在第 2 步，这一步走得非常正确。但如果你让 LLM 模拟到最后（比如需要 10 步），由于 LLM 存在幻觉，它可能在第 8 步的时候突然犯蠢导致任务失败。
>
>    这里引入 Value Network：**通过 Value Network 进行 Evaluation。**不需要完成游戏，只需要知晓当前状态，就能输出一个胜率估值。
>
>    在 LATS 中，LLM 就扮演了这个价值网络。系统通过精心设计的 Prompt（比如：“请评估当前已有的推理步骤是否走在正确的方向上，是否存在逻辑漏洞，给出 1-10 的评分”），让 LLM 运用其庞大的预训练世界知识，直接对局部状态进行静态分析。同样，AlphaGo 也是引入 Value Network。可以发现，这本质是一个可训练的网络，LLM 本质也是一个可训练的网络。
>
>    然而，这种 Rollout + 合并 question 与 answer 的贴合 MCTS 的想法在 Math & Coding 领域似乎是 SOTA 的，当然还需要加入一些中间过程性奖励模型和大小核架构等方法去减少算力消耗。



# Prompt Engineering & Agent Architecture

Agent Research 中强化 LLM 推理并且不改变模型内部权值的部分可以分为 Prompt Engineering 和 Agent Architecture，而强化 LLM 推理能力的另一种方式就行面向 agent 的训练与微调，比如 agentic rl，当然目前这三类逐渐融合使用。

- Prompt Engineering 本质是通过改变模型的输入来改变 LLM 在推理时的概率分布。

  - Reasoning Patterns：有许多的推理范式比如 CoT，Self-Consistency 通过多数投票来提高准确率，Least-to-Most 作为 CoT 的进阶通过构造“拆解并按顺序解决”的提示词辅助完成复杂长链条推理任务。
  - In-context Learning：一种 Few-shot Learning，研究如何通过少量示例让模型在不更新权重的条件下，快速习得特定的输入输出映射。本质上是在超大规模与训练后表现出的一种行为能力。
  - Role-Playing & Constraints：通过 Prompt 去定义模型的身份和回答输出格式等。

  特点是，由于完全发生在模型内，没有外部干扰，所以呈现随机性（**prompt 只是一种建议而非命令**）和高敏感度。

- Agent Architecture 研究的是如何在 LLM 之外建立一套**确定性的逻辑框架**，将模型封装成一个有分析、记忆、规划、行动能力的智能体。

  - Planning：任务分解（ReAct, Plan-and-Solve），模型对结果的自我修正（Reflexion 框架）。
  - Memory：Short-term Memory（利用 Context Window 作为 memory），Long-term Memory（通过外部向量数据库实现 RAG 检索，通过缩进技术实现信息召回）。
  - Tools：针对 Action，研究如何调用（API 接口相关），调用哪个，如何解析执行结果，当然这部分相关的 Toolformer 更倾向于 SFT。
  - Control Flow：如 LATS 的 agent 底层，Multi-Agent Systems 之间的协作等等。

  特点是，在模型外增加外部代码和结构对 agent 进行控制，使 agent 的行为确定性增加，另外通过外部储存如数据库来增强 agent 的记忆管理。







