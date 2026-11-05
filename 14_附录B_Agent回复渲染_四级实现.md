# Agent 回复渲染：从 API 字段到四级展示 · 工程实现文档

> 面向产品经理的代码级拆解。主线：**先吃透模型 API 的入参/出参有哪些字段 → 再看这些字段如何逐级映射成前端展示**。所有代码以 Anthropic Messages API 为准（OpenAI 差异随文标注）。贯穿一句话：**协议不给内容贴标签，协议用结构表达语义；渲染器的本分是读结构，不是猜内容。**

---

## 第 0 部分：模型 API 的入参与出参（地基）

### 0.1 入参（request）

```python
resp = client.messages.create(
    model="claude-sonnet-4-5",       # 模型
    max_tokens=4096,                 # 本轮输出上限（撞顶会 stop_reason=max_tokens）
    system=SYSTEM_PROMPT,            # 通道一：系统提示词（模块7）
    tools=TOOLS,                     # 通道二：工具清单（模块2，独立于 system）
    messages=messages,              # 通道三：对话历史（模块1，只增不减）
    stream=True,                     # ★ 是否流式——决定前端是"逐字长出"还是"整段蹦出"
)
```

对渲染而言，`stream=True` 是分水岭：关了它，出参是一个完整对象，前端等全部生成完才拿到；开了它，出参变成一串**事件流**，前端边收边画——所有"打字机效果""工具卡片实时变状态"都建立在流式之上。

### 0.2 出参（非流式，先看完整结构）

```python
{
  "id": "msg_01…",
  "role": "assistant",
  "model": "claude-sonnet-4-5",
  "stop_reason": "tool_use",        # ★★ 全轮的控制信号+语义分类器（见 0.4）
  "stop_sequence": None,
  "usage": {"input_tokens": 2310, "output_tokens": 145,   # 计费与压缩阈值判断依据（模块6）
            "cache_read_input_tokens": 2048},             # 命中缓存的部分（模块7）
  "content": [                       # ★★★ 有序的 block 数组——渲染的主料
    {"type": "thinking", "thinking": "用户要竞品分析，我先看有哪些数据…"},
    {"type": "text", "text": "我先查一下现有的竞品资料。"},
    {"type": "tool_use", "id": "toolu_01A", "name": "glob",
     "input": {"pattern": "competitors/*.md"}}
  ]
}
```

三个字段撑起整个渲染，务必分清它们各自的角色：

- **content**：一个**有序数组**，不是三个并列字段。同类型块可多次出现、顺序有意义（模块1）。它是"这一轮 assistant 说了什么、想调什么"的全部内容。
- **stop_reason**：整轮生成结束的原因。它**不在 content 里**，是轮级元数据。
- **usage**：token 账单，喂给压缩管线判断阈值、喂给成本面板。

### 0.3 content 里的三种 block（渲染的原子）

| block 类型 | 关键字段 | 语义 | 注意 |
|---|---|---|---|
| `thinking` | `.thinking`（文本） | 模型的内部推理 | 可能没有；开了 extended thinking 才有 |
| `text` | `.text`（文本） | 模型说给人看的话 | **身份不固定**——是旁白还是答案，看 stop_reason（0.4） |
| `tool_use` | `.id` `.name` `.input` | 模型要调工具的**意图** | 只有意图，没有执行结果；结果在下一轮的 tool_result |

关键澄清：**tool_use 块里没有"执行状态"**。"正在执行/成功/失败"是 harness 跑 handler 时才知道的，不来自模型输出。前端那张会变状态的工具卡片，是靠 `tool_use.id`（即 `tool_use_id`）把"模型的意图块"和"harness 的执行状态"缝在一起的（0.5）。

### 0.3.1 ★ 纠偏：不是每轮都有三种块（重要，别被上面的示例误导）

0.2 的示例里三种块齐全，那只是**众多可能之一，不是常态**。真实规律：**content 数组至少有一个块（不会空回复），但装什么、装几个完全由模型这轮的行为决定。** 三种块按"确定性"从高到低：

- **tool_use：由 stop_reason 硬绑定，最确定。** `stop_reason == "tool_use"` ⟺ 这轮 content 里**至少有一个** tool_use 块（双向、协议保证，不是"通常"）。反过来 `end_turn` 那轮**一定没有** tool_use。且一轮可以有**多个** tool_use（并行，模块1 §8.1）。
- **text：最不稳定，经常整轮缺席。** 大量中间轮次是"只干活、不说话"——content 里只有 tool_use、一个 text 都没有。模型闷头连调五个工具、全程零解说，正常且常见。text 出不出取决于：模型自己想不想说，以及**你有没有在 prompt 里用纪律要求它说**（第2级那条"先说明再调用"就是人为拉高 text 出现率）。
- **thinking：默认根本不存在。** 普通调用 content 里没有 thinking 块，只有显式开启 extended thinking（`thinking:{type:"enabled", budget_tokens:…}`）才有。多数生产调用不开它（费 token、增延迟）。

真实的一轮的几种典型形态：

| 形态 | content | stop_reason | 场景 |
|---|---|---|---|
| 纯干活轮（最常见） | `[tool_use]` 或 `[tool_use, tool_use, …]` | tool_use | 中间轮，**无 text** |
| 边说边干轮 | `[text, tool_use]` | tool_use | 开了解说纪律 |
| 交卷轮 | `[text]` | end_turn | text 即最终答案 |
| 开 thinking 的干活轮 | `[thinking, tool_use]` | tool_use | 开了 extended thinking |
| "三件套"（特例） | `[thinking, text, tool_use]` | tool_use | 需同时满足：开 thinking + 模型恰好想说 + 又要调工具 |

**产品铁律：UI 绝不能假设每轮都有"给用户的话"。** 若渲染逻辑指望每轮都有 text 来推进度，大量沉默的干活轮会让界面"卡住不动"。这正是第 3 级用 todo 状态而非 text 驱动进度条的根本原因——**进度必须挂在必然发生的结构化信号（stop_reason、todo 状态变迁）上，不能挂在时有时无的 text 上。**

### 0.3.2 ★ 这套结构是训练来的，不是 prompt 教的（关键认知）

常见误解：以为要在 prompt 里写"请输出 text 和 tool_use 两种字段"。**不需要，而且有害。** 分两层：

**能力层（结构）= 训练 + API 参数，你管不着也不该管：**

- 模型被专门训练成会输出类型化的块、会按 tool schema 生成合法 tool_use、并在调工具时把 stop_reason 收成 `tool_use`（不调收 `end_turn`）。这是训练进权重的。
- 服务端还有**约束解码**兜底：模型进入生成 tool_use 的状态时，解码器强制输出符合你声明的 JSON schema——所以 input 一定合法、字段一定对得上（模块1 讲 ReAct→结构化的"API 层保证"就是它）。结构可靠性是训练+解码双保险。
- **前提**：这套能力由**传入 `tools` 参数**激活（模块2 的独立通道）。传了 tools 才等于"这局允许打 tool_use 这张牌"；不传，模型只会输出 text——不是不会，是没开这个功能。
- 若你偏要在 prompt 里描述 tool_use 格式，模型可能在 **text 里打印一段"长得像 tool_use 的文字"**，而不走真正的结构化通道——等于把它打回 ReAct 时代。

**内容层（行为）= prompt 的地盘，你能引导的是行为不是格式：**

- 调不调工具、调哪个 → 模型读 tool 的 description 决定（工具描述即 prompt，模块2）。
- 这轮说不说话 → 默认未必说；想让它边干边解说，prompt 写纪律拉高 text 出现率（第2级）。**拉高的是出现率，不是"教会它输出 text 字段"——字段它本来就会。**
- thinking 出不出 → 既不靠训练默认也不靠 prompt，靠 API 的 thinking 开关。

**一句话钉死：** 「能输出哪些类型的块」是训练 + API 参数决定的能力边界（你通过传 tools、开 thinking 激活）；「这一轮实际输出什么」是模型的内容决策（你通过 prompt 和工具描述引导，但引导的是行为，不是格式）。落回主线——**harness 不生产智能，它布置环境**：你布置好牌桌（tools 参数）和规矩（prompt），怎么出牌是模型的本事。

### 0.4 stop_reason：一个字段，两个消费者

同一个 `stop_reason`，循环控制器和渲染器各读各的——这是结构化协议的红利（一份信号多处受益）：

| stop_reason（OpenAI：finish_reason） | 循环控制器读到后 | 渲染器读到后：本轮 text 块的身份 |
|---|---|---|
| `tool_use`（`tool_calls`） | 执行工具，继续循环 | **过程旁白**（"我先查一下…"）→ 小字/进度气泡 |
| `end_turn`（`stop`） | 跳出循环，交卷 | **最终答案** → 正文气泡，完整 markdown |
| `max_tokens`（`length`） | 触发续写恢复（模块11） | **被截断的半成品**——别当答案渲染 |

深层原理：**text 块的"身份"不写在它自己身上，由它的上下文位置决定**——被 tool_use 跟着的是旁白，收在 end_turn 里的是答案。这和"tool_result 靠 id 认领""历史消息看时态"同一个哲学。渲染器读结构就永远分类对；靠猜内容分类的产品，迟早把旁白误当答案。

### 0.5 流式出参：真正驱动前端的事件序列

`stream=True` 时，上面那个完整对象被拆成按顺序推送的事件。渲染器实际监听的是这些（名字以 Anthropic 为准）：

```
message_start                    # 一轮开始，拿到 msg id、role
  content_block_start  index=0   type=thinking     # 第0块开始（宣告类型）
  content_block_delta  index=0   thinking_delta    # 逐 token 推 thinking 内容
  content_block_stop   index=0                      # 第0块结束
  content_block_start  index=1   type=text          # 第1块开始
  content_block_delta  index=1   text_delta         # 逐 token 推 text（打字机效果来源）
  content_block_stop   index=1
  content_block_start  index=2   type=tool_use  id=toolu_01A  name=glob
  content_block_delta  index=2   input_json_delta   # 逐 token 拼 input 的 JSON 字符串
  content_block_stop   index=2
message_delta   stop_reason=tool_use   usage=…      # ★ stop_reason 在最后才到！
message_stop
```

**这里藏着渲染工程的核心难点**：`stop_reason` 在 `message_delta`（全轮结束）才到，而 text 是 `content_block_delta` 阶段就在逐字流出的。也就是说——**渲染器开始画一段 text 时，还不知道这轮末尾会不会跟一个 tool_use，因此当下无法判定这段话是旁白还是答案**。四级方案里对这个难点的处理方式，是区分它们成熟度的关键（见 1.3）。

---

## 第 1 级：直通渲染（零智能，纯搬运）

### 原理
一张写死的「block 类型 → UI 组件」映射表，逐块渲染。不动模型、不动 harness，纯前端。

### 代码骨架

```javascript
// 监听流式事件，维护一个"当前轮"的块列表，边收边画
const blocks = [];                          // 本轮 assistant 的块
let stopReason = null;

stream.on("content_block_start", (e) => {
  blocks[e.index] = { type: e.content_block.type, ...e.content_block, text: "", done: false };
  mountComponent(e.index, blocks[e.index]); // 立刻挂一个空组件占位
});
stream.on("content_block_delta", (e) => {
  const b = blocks[e.index];
  if (e.delta.type === "thinking_delta") b.text += e.delta.thinking;
  if (e.delta.type === "text_delta")     b.text += e.delta.text;
  if (e.delta.type === "input_json_delta") b.partialJson += e.delta.partial_json;
  updateComponent(e.index, b);            // 逐 token 刷新（打字机）
});
stream.on("content_block_stop", (e) => { blocks[e.index].done = true; });
stream.on("message_delta", (e) => { stopReason = e.delta.stop_reason; });
```

### 字段 → 前端映射表

| 出参字段 | 前端组件 | 原理 |
|---|---|---|
| block.type == `thinking` | 灰色可折叠"思考中"区 | 内部推理，展示但弱化 |
| block.type == `text` | markdown 气泡 | 模型说给人的话 |
| block.type == `tool_use`（name/input） | 工具卡片，显示"调用 glob(competitors/*.md)" | 展示模型意图 |
| `input_json_delta` 累积 | 卡片里参数逐字长出 | 流式拼 JSON 字符串 |
| tool_result（下一轮，按 id 匹配） | 找到对应卡片 → 更新 ✓/✗ + 折叠输出 | id 缝合意图与结果（0.5） |
| stop_reason | 决定最后一段 text 是否为正文 | 见 0.4 |

### 代表产品
Claude Code 终端界面。特点：所有内容都摊开显示，忠实但"吵"。

---

## 第 2 级：降噪折叠（纯前端策略，数据不变）

### 原理
和第 1 级**收到的数据一模一样**，只改 CSS/交互策略：把噪音压下去，把信号顶上来。一行模型代码、一行 harness 代码都不用改。

### 具体策略与映射

| 数据（同第1级） | 展示策略 | 目的 |
|---|---|---|
| thinking 块 | 默认折叠，标题只留"思考中…"，点击展开 | 推理是给需要的人看的，不占主视线 |
| 连续多个 tool_use | 合并成一行："读取了 3 个文件 ✓" | 消除机械噪音 |
| tool_result 内容 | 默认隐藏，卡片上只留成功/失败徽章 | 载荷太长，按需下钻 |
| text（旁白类） | 弱化为灰色小字 | 与最终答案拉开视觉层级 |
| text（答案类，end_turn） | 保持醒目正文 | 用户真正要读的 |

### 关键点
第 2 级证明了一件事：**很多"体验提升"根本不需要碰模型，只是前端如何解释同一批结构化数据**。产品上这是性价比最高的一档——改版快、无回归风险。

---

## 第 3 级：结构化进度信号（你问的"阶段性提示"的正解）

### 核心洞察
"我完成了第一阶段，开始第二阶段"这类提示，**最稳的实现不是去解析模型的散文，而是渲染 todo/task 的状态变迁**（模块3/9）。进度展示挂在**结构化状态**上，而非自然语言上——稳定、可靠，还反过来当防漂移锚点（一个机制赚两次）。

### 两条实现路径

**路径 A：渲染 todo 状态机（会话内进度条）**

模型调 `todo_write` 更新清单 → 这是一个普通 tool_use，但 harness 给它**专门的渲染器**（不走通用工具卡片）：

```javascript
// harness 侧：识别出是 todo 工具，走专用通道
if (toolUse.name === "todo_write") {
  const todos = toolUse.input.todos;
  renderProgressWidget(todos);   // 渲染成阶段卡片/进度条，而非工具卡片
  // 计算 "3/5 完成，当前：撰写竞品分析" 这类摘要
}
```

| 结构化字段 | 前端展示 | 原理 |
|---|---|---|
| todo[i].status: pending→in_progress | 该项高亮为"进行中" | 状态变迁 = 阶段推进的客观信号 |
| todo[i].status: in_progress→completed | 打勾、进度条+1 | "完成第一阶段"的可靠来源 |
| 全表统计 | "3/5 · 当前：写报告" | 一眼可扫的总览 |

**路径 B：专用"给用户捎话"工具**

模型中途要对用户说必须原样送达的话（不是旁白，是正式通知），调一个专用工具而非普通 text：

```python
{"name": "send_user_message",
 "description": "向用户发送一条需要原样呈现的消息（如阶段汇报、需要确认的问题）",
 "input_schema": {"type": "object", "properties": {"message": {"type": "string"}}}}
```

harness 把这个 tool_use 渲染成**醒目通知气泡**，和折叠的过程旁白区分开。好处：模型显式声明"这句要给用户看"，而不是靠渲染器猜 text 块的重要性。（Cowork 环境里就有这样一个工具。）

### 为什么比"解析散文"强
散文式进度（靠 NLP 从 text 里抠"第一阶段完成"）脆、易误判；结构化信号是模型主动写入的、机器可读的状态，天生适合驱动 UI。这就是模块 3 那句"todo widget 是最廉价的可解释性来源"在渲染层的兑现。

---

## 第 4 级：harness 侧摘要压缩（成本换整洁）

### 何时需要
任务跑几十轮、产生海量中间内容，连第 2 级的折叠解说流都太吵。此时对**展示流**做压缩。

> ⚠️ 关键区分：这是压缩**展示流**（给用户看的），**不是**压缩上下文（给模型看的，那是模块6）。两者独立——上下文可以完整保留，展示可以大幅精简；反之亦然。

### 两种实现

**规则版（便宜、确定性）**

```javascript
// 按 todo 项给活动分组，每组只显示一行滚动状态
function summarizeActivity(blocksThisPhase) {
  const toolCount = blocksThisPhase.filter(b => b.type === "tool_use").length;
  return `处理"${currentTodo}"中（已调用 ${toolCount} 次工具）…`;  // 只留最新一行
}
```

**模型版（贵、语义强）**

```python
# harness 异步雇一个便宜小模型，把一段活动压成一句人话
summary = cheap_llm(f"用一句话概括 agent 刚才这段操作，给用户看：\n{render(activity)}")
# → "正在对比三家竞品的定价页"
```

### 字段 → 展示映射

| 数据 | 展示 | 代价 |
|---|---|---|
| 一批 tool_use + text（中间轮） | 摘要成一行进度（规则或小模型） | 保真度下降 |
| 原始 blocks | 折叠在"展开查看详细过程"入口后 | —— |
| end_turn 轮的 text | 仍按正文完整渲染 | 最终答案永不摘要 |

### 铁律
第 4 级必须配一个**"展开原始过程"的下钻入口**——摘要有损，信任靠可回溯兜底。这和模块6"截断必留指针"、模块9"结论必附证据"是同一条原则在展示层的复现。

### 代表形态
Claude Cowork 的对话流：工具调用间的叙述被摘要呈现，只有需要用户原样读的内容（走 send_user_message）和最终答案是原文——分类依据正是 0.4 的 stop_reason 规则。

---

## 总表：四级对比

| 级别 | 改动层 | 数据来源 | 核心机制 | 代价 | 代表 |
|---|---|---|---|---|---|
| 0/1 直通 | 前端 | 原始 blocks | 类型→组件映射 | 吵 | Claude Code 终端 |
| 2 降噪 | 前端 | 同上 | 折叠/合并策略 | 几乎无 | 多数 chat UI |
| 3 结构化进度 | 模型+harness | todo/task 状态、专用工具 | 渲染结构化状态而非散文 | 修路(prompt) | Cowork 任务卡 |
| 4 摘要压缩 | harness | 展示流（非上下文） | 规则/小模型摘要+下钻 | 保真度 | Cowork 对话流 |

**成熟产品四层叠用**：结构化任务卡打底（扫一眼）＋ 模型解说穿插（跟着看）＋ 原始工具卡折叠可查（出事要查）＋ 超长过程摘要滚动——分别服务三种用户姿态。

---

## 附：给 Tanka 的落地建议

1. **先做 0→2 级**：一周内可上，性价比最高，纯前端。
2. **第 3 级是差异化重点**：Tanka 有 IM/任务体系，把 agent 进度渲染成团队可见的任务卡，是别人给不了的（呼应模块9 Q3 的人机同图）。
3. **第 4 级按需**：只有长任务多的场景才需要，且务必先建好"下钻看原文"，否则用户不信任摘要。
4. **stop_reason 的分类逻辑要写对**：这是最容易被"猜内容"实现方式坑到的地方——务必读结构（block 顺序 + stop_reason），不要用关键词猜 text 是不是最终答案。
