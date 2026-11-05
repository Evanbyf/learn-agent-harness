# Agent Loop 主循环 · AI 产品经理学习文档

## 1. 最小可用的 Agent 循环（26 行 Python）

```python
import anthropic                                      # Anthropic 官方 SDK
client = anthropic.Anthropic()                        # 初始化客户端，属于 harness（脚手架）
messages = [{"role": "user", "content": "东京今天天气如何？"}]  # 对话历史，只增不减
while True:                                           # 主循环：跑到模型宣布"说完了"为止
    resp = client.messages.create(                    # 每一轮都把【完整】历史发给模型
        model="claude-sonnet-4-5",                    # 指定模型
        max_tokens=1024,                              # 单轮输出上限
        tools=TOOLS,                                  # 向模型【声明】可用工具，仅是说明书
        messages=messages,                            # 传入全部历史，模型本身无记忆
    )
    messages.append(                                  # 追加点①：harness 把模型回复存入历史
        {"role": "assistant", "content": resp.content})
    if resp.stop_reason != "tool_use":                # 分支：模型这轮没有请求用工具
        break                                         # → 说明任务完成，跳出循环
    results = []                                      # 否则收集本轮所有工具结果
    for block in resp.content:                        # 模型一轮可能同时请求多个工具
        if block.type == "tool_use":                  # 找出其中的工具调用块
            out = run_tool(block.name, block.input)   # ★ harness 真正执行工具（本地代码）
            results.append({                          # 把结果包装成 tool_result
                "type": "tool_result",
                "tool_use_id": block.id,              # 用 id 和请求一一配对
                "content": out})
    messages.append(                                  # 追加点②：以 user 角色"回灌"结果
        {"role": "user", "content": results})
print(resp.content[0].text)                           # 循环结束，打印最终答案
```

## 2. ASCII 流程图

```
        ┌──────────────────────────┐
        │ messages = [用户问题]     │
        └────────────┬─────────────┘
                     ▼
        ┌──────────────────────────┐
   ┌───▶│ 调 API：发送完整 messages │
   │    └────────────┬─────────────┘
   │                 ▼
   │    ┌──────────────────────────┐
   │    │ 追加 assistant 回复      │
   │    └────────────┬─────────────┘
   │                 ▼
   │        stop_reason == "tool_use"?
   │            │是            │否
   │            ▼              ▼
   │    ┌──────────────┐   ┌──────────┐
   │    │ harness 执行 │   │ 输出答案  │
   │    │ 工具，得结果  │   │ 循环结束  │
   │    └──────┬───────┘   └──────────┘
   │           ▼
   │    ┌──────────────────────────┐
   └────│ 追加 user(tool_result)   │
        └──────────────────────────┘
```

## 3. messages[] 如何只增地累积

`messages` 是整个 agent 的唯一"记忆"。模型是无状态的：每轮调用都要把从第一条到最新一条的完整历史重新发过去。历史只有两个追加点，各有明确的责任人和时机：

- 追加点①（代码第 11 行）：API 返回后，**harness** 立刻把模型的整段回复（含文字和 tool_use 块）以 `assistant` 角色追加；
- 追加点②（代码倒数第 2 行）：工具跑完后，**harness** 把 `tool_result` 以 `user` 角色追加。

于是历史永远是 `user → assistant → user → assistant …` 的交替只增序列，任何一条都不会被修改或删除。工具结果之所以标 `user` 角色，是因为对模型而言它是"外部世界传回的新输入"，和用户发言地位等同。多轮之后上下文越滚越长——这正是长任务会遇到 token 上限、需要"上下文压缩"的根源，也是 PM 评估 agent 成本时要盯住的变量。

## 4. 一句话点睛

**模型只在 `resp.content` 里"说"它想调什么工具（决策），真正跑代码的是 harness 里那行 `run_tool(...)`（执行）——模型永远碰不到执行环境，这条边界就是 agent 安全与权限设计的落点。**

## 5. 对照：ReAct 与 OpenAI 函数调用循环

**与 ReAct 的关系**：骨架同源，都是"推理→行动→观察"循环。ReAct（2022 论文）靠 prompt 约定让模型输出 `Thought: / Action: / Observation:` 纯文本，harness 用正则去"抠"出动作，易解析失败；上面的循环用结构化的 `tool_use` / `tool_result` 块替代文本约定，`stop_reason` 替代"检测到 Final Answer 就停"。可以说 Messages API 循环是 ReAct 的工程化产品版。

**与 OpenAI 函数调用循环**：思想完全一致，仅字段名不同——OpenAI 用 `finish_reason == "tool_calls"` 对应这里的 `stop_reason == "tool_use"`；工具结果以独立的 `role: "tool"` 消息回灌，而 Anthropic 是放在 `user` 消息内的 `tool_result` 块；OpenAI 的函数参数是 JSON 字符串需自行 `json.loads`，Anthropic 的 `block.input` 已是解析好的对象。跨厂商迁移时改的只是"信封格式"，循环结构一行不用动。

## 6. 自测题

1. 把追加点②的 `"role": "user"` 误写成 `"assistant"`，API 会报错还是模型行为异常？为什么协议要求 tool_result 必须出现在 user 消息里？
2. 模型一轮同时请求了 3 个工具，但 harness 只回灌了 2 个 tool_result，下一轮调用会发生什么？
3. 如果去掉 `while True` 只调用一次 API，这个系统退化成了什么？据此用一句话向老板解释"chatbot 和 agent 的本质区别"。

---

## 7. 自测题参考答案（学习后补充）

**Q1：role 写错会怎样？**
API 直接报 400，轮不到模型行为异常。违反两条协议规则：① user/assistant 必须严格交替，追加点②前面刚追加了一条 assistant（模型的 tool_use 回复），再来一条 assistant 不合法；② `tool_result` 块只允许出现在 user 消息里（assistant 的合法块是 text / tool_use / thinking）。

为什么必须是 user：role 的本质是标注**信息的来源方向**——assistant = 模型说出去的，user = 外部世界喂进来的。工具结果是环境的反馈，因果上与用户发言同类。若允许标成 assistant，模型更容易学会"不等结果、自己编一个 Observation"——这正是 ReAct 纯文本时代的经典病（模型自问自答、伪造工具输出）。role 隔离从结构上堵死了这条路。

**Q2：3 个 tool_use 只回灌 2 个 tool_result？**
仍是 API 报 400（`invalid_request_error`），请求根本发不出去。协议硬校验：一条 assistant 消息里的每个 tool_use，都必须在紧跟的 user 消息里有 `tool_use_id` 配对的 tool_result。

产品含义：工具失败、超时、被用户拒绝，都**不能跳过不回**——失败回 `"is_error": true` 的 tool_result，拒绝回"用户拒绝了此操作"。这就是模块 2 要讲的"错误也要作为 tool_result 回去，而不是抛异常中断循环"的根源。

**Q3：去掉循环退化成什么？**
退化成普通 chatbot（一次问答）。模型即使返回 tool_use，也没有下一轮去执行和回灌——伸出手没人握。
给老板的一句话：**chatbot 只能告诉你该怎么做，agent 会自己动手做、看结果、不对就再来，直到做完——差别不在模型更聪明，而在外面那圈"没做完就再跑一轮"的循环。**

三题本质是同一条协议的三个侧面：正因为 tool_result 必须紧跟在 tool_use 之后的 user 消息里（Q2），两个追加点才必须严格成对（Q1），而这个"发出→执行→回灌"的节拍必须由循环驱动（Q3）。

---

## 8. 追问与深挖（学习中的延伸问题）

### 8.1 没有 id 配对，并行工具调用会怎样翻车？

场景：用户问"东京、大阪、京都哪个周末最凉快？订最凉快那个城市的酒店"。模型一轮并行发出 3 个 `get_weather`（id 分别为 A/B/C）。harness 并发执行，网络不讲顺序：完成顺序（大阪→京都→东京）≠ 发出顺序（东京→大阪→京都）。

无 id、按完成顺序拼字符串（ReAct 式）：三条 `Observation: 晴，24°C…` 没有身份标记，模型按"我发问的顺序"去猜对应关系 → 三个城市温度全配错 → 下一步自信地订了最热的东京。最阴险处在于**全程无报错**：拼接合法、推理通顺，错误要到线下才暴露——"静默的张冠李戴"比循环卡死难排查得多。

带 id 的结构化回灌：tool_result 块乱序排列无所谓，模型按 `tool_use_id` 回查自己的请求，配对靠标签不靠位置。且局部失败可精确定位：京都超时就回 `{tool_use_id:"C", is_error:true}`，模型只重试京都。

ReAct 没这个问题是因为它根本做不了并行（一轮只能写一个 Action，3 个查询 = 3 轮循环 = 3 倍延迟和 token）。所以 id 的本质不是修 bug，而是**用一个廉价标签换来"并行"这整个能力**。

### 8.2 同一轮的两个 tool_use 有依赖怎么办？

答案：协议层没有依赖字段、没有 DAG；模型也不会（不应该）同轮输出有依赖的调用——因为**物理上写不出来**。生成 tool_use 时 `input` 必须是具体完整的 JSON 值，没有占位符和变量引用。"找到 README 再读它"：发 `read_file` 时 path 该填什么？那个值只存在于还没执行的搜索结果里 → 写不出合法 input → 只能先发搜索。

**依赖的表达方式不是字段，而是轮次**：

```
并行（无依赖）：一轮搞定          串行（有依赖）：拆成两轮
┌────────────────────┐          ┌────────────────────┐
│ tool_use A + B     │          │ 第1轮：只发 A       │
└─────────┬──────────┘          └─────────┬──────────┘
          ▼                               ▼
   并行执行，乱序完成                 执行 A → 回灌结果
          ▼                               ▼  ← 同步点
   结果按 id 一起回灌            ┌────────────────────┐
                                │ 第2轮：发 B，input  │
                                │ 里填入 A 的输出     │
                                └────────────────────┘
```

并行由「同一轮的多个块」承载，串行由「循环的多轮」承载，每次 API 往返就是天然同步点——while 循环不只是"没做完再来"，还是**依赖调度器**。

失败模式是模型"猜"：不等搜索、同轮赌一个 `read_file(path="./README.md")`。harness 无从察觉（拿到的是合法 input），照常执行；猜错则错误作为 tool_result 回灌，模型下一轮自纠。系统靠**错误回灌自愈**，不靠 harness 做依赖分析——harness 串行执行也救不了，因为 B 的 input 在生成那刻已定死。

关联与对照：「上游依赖字段」这个直觉对应的是另外两层的东西——① 模块 9 的 Task 系统（blocks/blockedBy），那是 harness 层跨会话的持久任务图，粒度是任务而非单次调用；② LLMCompiler（2023）让模型一轮输出带 `$1` 占位符的 DAG 计划、harness 拓扑排序执行，省往返但偏离标准协议，属自定义 harness 深水区。

产品启示：每级依赖 = 多一次 API 往返。工具设计要**压扁依赖链**——提供 `glob` 这种一步到位的工具，而不是逼模型先 `ls` 再逐层进目录。

### 8.3 ReAct → 结构化循环：三个环节的升级（第 5 节展开）

| 环节 | ReAct（文本约定） | Messages API（结构化） |
|------|------------------|----------------------|
| 动作怎么说 | prompt 求模型遵守 `Action:` 格式，harness 正则抠，抠不出即卡死（20 步 × 97% 成功率 ≈ 整体仅 54%） | `tool_use` 是 API 层数据类型：训练 + 约束解码保证 input 为合法 JSON，harness 只读字段 |
| 什么时候停 | 正文里搜 `Final Answer:` 字样，会误杀/空转 | `stop_reason` 是独立元数据字段，正文内容不干扰控制流 |
| 结果怎么回 | 拼 `Observation:` 续在字符串尾部 | `tool_result` 块带 id 配对 → 解锁并行（见 8.1） |

一句话：逻辑没变，但把最易出错的"理解对方意图"环节，从 prompt 约定下沉为**模型训练 + API 协议双重保证的硬结构**。harness 的所有控制流（分发、判停、配对）都应挂在结构化字段上，永远不要依赖对模型自然语言输出的解析。
