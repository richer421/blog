---
title: Agent 开发经验总结：提示词工程与会话管理
date: 2026-02-20 22:00:00
tags:
  - AI
  - Agent
  - 提示词工程
  - 混沌工程
  - 大模型
categories:
  - AI
  - 开发实践
---

## 写在前面

最近在给自己的混沌工程平台开发内置 Agent，用的是 Eino 框架。开发过程中踩了一些坑，也总结出了一些实用的经验，记录在这里。

这个 Agent 的定位是**平台的自动化助手**——它具备平台的全量业务知识，能调用平台的内置工具，帮助用户辅助完成操作。常见场景比如：一句话创建演练、演练报告分析等等。

<!-- more -->

整个开发流程，我把它拆成了三大块：

1. **会话管理**：对话上下文怎么存、怎么管
2. **提示词工程**：怎么让大模型准确理解业务、正确调用工具
3. **Agent 编排**：工具链和执行流程怎么串起来

这篇文章主要聊前两个：**提示词工程**和**会话管理**。

## 提示词工程

提示词工程是 Agent 开发中最核心的部分。说白了，大模型能不能干好活，很大程度上取决于你的提示词写得好不好。

下面分三个方面来聊。

### 工具调用前的参数准备

B 端系统有一个很大的特点：**一个动作涉及的参数往往非常多**。

以"创建一个混沌演练"为例，可能需要的参数有：演练名称、目标集群、故障类型、注入范围、持续时间、调度策略、通知渠道……十几个字段是常态。

如果把这些参数全部抛给用户填，体验一定很差。用户找你的 Agent 就是图方便，结果你让他填一堆表单，那还不如直接去页面上点。

所以核心思路是：**尽可能少地让用户提供参数**。

怎么做？把工具函数需要的全量参数，按**来源**进行分类：

| 参数来源 | 说明 | 示例 |
|---------|------|------|
| 用户输入提炼 | 从用户的自然语言描述中提取 | "对 order 服务注入 CPU 满载" → 目标服务: order, 故障类型: CPU 满载 |
| 调用工具查询 | 通过调用其他工具获取 | 用户说了服务名，Agent 自动调接口查出对应的集群 ID、命名空间等 |
| 默认值 | 大多数场景下的合理默认 | 持续时间默认 30s，调度策略默认"立即执行" |
| 反问用户确认 | 必须用户决策、无法推断的参数 | "你希望演练在哪个环境执行？测试环境还是预发环境？" |

把这个分类写进工具的描述信息里，大模型拿到用户输入后，就会根据工具函数中的参数来源信息，**自动通过各种方式去补全参数**：能从用户话里提取的就提取，能查的就调工具去查，有默认值的就用默认值，实在缺的才反问用户。

这样一来，用户可能只需要说一句"帮我对 order 服务做个 CPU 满载演练"，Agent 就能把十几个参数都补齐，最后只确认一下关键信息就能执行。

**用户体验和参数完整性，就这样兼顾了。**

### 配置化的工具提示词：YAML + Embed 自生成

工具多了之后，每个工具都手写提示词，代码会变得很臃肿。而且不同人写的风格不一样，维护起来很头疼。

我的做法是：**用 YAML 配置文件定义工具提示词，通过 Go 的 embed 嵌入到代码中，运行时自动生成最终的提示词文本**。

大致结构如下：

```yaml
# tools/create_exercise.yaml
name: create_exercise
description: 创建一个混沌演练
parameters:
  - name: target_service
    type: string
    required: true
    source: user_input
    description: 目标服务名称

  - name: fault_type
    type: string
    required: true
    source: user_input
    description: 故障类型，如 CPU 满载、内存溢出、网络延迟等

  - name: cluster_id
    type: string
    required: true
    source: tool_query
    description: 集群 ID，通过目标服务名称调用 get_cluster_info 工具获取

  - name: duration
    type: integer
    required: false
    source: default
    default: 30
    description: 故障持续时间（秒）

  - name: environment
    type: string
    required: true
    source: ask_user
    description: 执行环境，需要用户确认
```

代码里通过 `embed` 统一加载：

```go
//go:embed tools/*.yaml
var toolPrompts embed.FS

func LoadToolPrompts() map[string]ToolPrompt {
    // 遍历 YAML 文件，解析为统一结构
    // 根据模板生成最终提示词
}
```

这么做的好处：

- **统一格式**：所有工具的提示词风格一致，大模型更容易理解
- **减少代码量**：新增工具只需要加一个 YAML 文件，不用改代码
- **更适合 AI 开发**：让 AI 帮你写 YAML 配置比让它改业务代码安全得多
- **易于维护**：非开发人员也能理解和修改工具配置

### 系统提示词的结构化设计

系统提示词是 Agent 的"人格"和"能力边界"。设计得好，Agent 表现稳定；设计得差，它会各种"发挥"。

我采用的是 **知识 - 规则 - 行为 - 例子** 四段式结构：

```markdown
## 知识（Knowledge）
你是混沌工程平台的智能助手。
平台核心概念：
- 演练（Exercise）：一次完整的故障注入实验
- 场景（Scene）：预定义的故障模板
- 目标（Target）：故障注入的对象，通常是服务或 Pod
...

## 规则（Rules）
1. 执行任何破坏性操作前，必须向用户确认
2. 不得同时对同一个服务注入多种故障
3. 生产环境的演练必须经过用户二次确认
4. 参数不完整时，优先通过工具查询补全，其次使用默认值，最后才反问用户
...

## 行为（Behavior）
- 收到用户请求后，先分析意图，再规划执行步骤
- 调用工具前，先检查参数完整性
- 执行完成后，给出结构化的结果摘要
- 遇到错误时，给出可能的原因和建议
...

## 例子（Examples）
用户：帮我对 order 服务做个网络延迟演练
助手：<thinking>用户想对 order 服务注入网络延迟故障。
我需要：1. 查询 order 服务的集群信息 2. 确认演练环境 3. 创建演练</thinking>
<reply>好的，我来帮你创建网络延迟演练。先查一下 order 服务的信息...</reply>
...
```

这个结构有几个优点：

1. **知识部分**让模型理解"我是谁、我在什么领域"
2. **规则部分**划定了行为边界，避免做出危险操作
3. **行为部分**定义了工作模式，让输出更可预测
4. **例子部分**最直观，few-shot 是提升效果最立竿见影的方式

这四个部分各司其职，调试的时候也方便定位问题——如果 Agent 对某个概念理解有误，改知识部分；如果做了不该做的事，改规则部分；如果输出格式不对，改行为部分或加个例子。

## 会话管理

因为不是通用对话产品，我们的会话管理设计比市面上那些 Chat 应用要简单不少。核心就两个东西：**对话缓存**和 **Session 元数据**。

### 基于 Redis 双向链表的对话缓存

对话历史用 Redis 的 List 结构来存，本质上就是一个**双向链表**。

为什么用链表而不是直接存数组？

- **追加消息效率高**：`RPUSH` 直接往尾部追加，O(1)
- **按窗口截取方便**：`LRANGE` 可以取最近 N 条消息
- **淘汰旧消息简单**：`LTRIM` 直接截断

```go
// 追加一条消息
func (s *SessionStore) AppendMessage(sessionID string, msg Message) error {
    data, _ := json.Marshal(msg)
    pipe := s.rdb.Pipeline()
    pipe.RPush(ctx, chatKey(sessionID), data)
    pipe.LTrim(ctx, chatKey(sessionID), -MaxHistoryLen, -1) // 保留最近 N 条
    _, err := pipe.Exec(ctx)
    return err
}

// 获取对话历史
func (s *SessionStore) GetHistory(sessionID string) ([]Message, error) {
    results, err := s.rdb.LRange(ctx, chatKey(sessionID), 0, -1).Result()
    // 反序列化并返回
}
```

对话窗口的大小是个需要权衡的参数。太短了模型会忘记上文，太长了 Token 消耗大、响应也慢。我们目前设的是 **20 轮**（即 40 条消息），对于平台操作场景来说基本够用。

### 基于 HashMap 的 Session 元数据

除了对话内容，每个 Session 还有一些元数据需要存储，比如：

- 创建时间
- 最后活跃时间
- 用户 ID
- 当前状态（进行中 / 已结束）
- 关联的演练 ID（如果有的话）

这些用 Redis 的 Hash 结构来存：

```go
func (s *SessionStore) SetMeta(sessionID string, meta SessionMeta) error {
    fields := map[string]interface{}{
        "user_id":     meta.UserID,
        "created_at":  meta.CreatedAt.Unix(),
        "last_active": meta.LastActive.Unix(),
        "status":      meta.Status,
        "exercise_id": meta.ExerciseID,
    }
    return s.rdb.HMSet(ctx, metaKey(sessionID), fields).Err()
}
```

Hash 的好处是可以**按字段更新**，不用每次都序列化整个对象。比如用户发了一条消息，只需要更新 `last_active` 字段就行。

整体结构很简单，但够用。没必要为了一个内置助手搞一套复杂的对话管理系统。

### 结构化输出：XML 标签驱动前端渲染

这个是我觉得比较有意思的一点。

传统的 Chat 界面，大模型的输出就是一段 Markdown 文本，前端直接渲染。但对于一个**平台内置助手**来说，纯文本的表现力太弱了。

我的做法是：**要求大模型完全按照 XML 标签格式输出**，前端根据不同的标签渲染不同的 UI 组件。

比如大模型的输出可能是这样的：

```xml
<thinking>
用户想创建一个 CPU 满载演练，目标是 order 服务。
我需要先查询 order 服务的集群信息，然后创建演练。
</thinking>

<reply>
好的，我来帮你创建演练。已查询到 order 服务部署在 cluster-prod-01 集群中。
</reply>

<action_card>
  <title>创建 CPU 满载演练</title>
  <status>待确认</status>
  <params>
    <param name="目标服务">order</param>
    <param name="故障类型">CPU 满载</param>
    <param name="集群">cluster-prod-01</param>
    <param name="持续时间">30s</param>
    <param name="环境">测试环境</param>
  </params>
  <actions>
    <button type="primary">确认执行</button>
    <button type="default">修改参数</button>
  </actions>
</action_card>
```

前端拿到这段输出后，按标签拆分，分别渲染：

- `<thinking>` → 折叠的"思考过程"面板，默认收起
- `<reply>` → 普通的文本消息气泡
- `<action_card>` → 一张操作卡片，带参数展示和操作按钮

```typescript
function renderBlock(block: ParsedBlock) {
  switch (block.tag) {
    case 'thinking':
      return <ThinkingPanel content={block.content} />
    case 'reply':
      return <MessageBubble content={block.content} />
    case 'action_card':
      return <ActionCard params={block.params} actions={block.actions} />
    case 'table':
      return <DataTable data={block.data} />
    case 'chart':
      return <Chart config={block.config} />
    default:
      return <MessageBubble content={block.content} />
  }
}
```

这样做的好处：

- **人机交互更生动**：不再只有冷冰冰的文本，有卡片、有按钮、有图表
- **用户体验更友好**：关键操作通过按钮确认，而不是让用户打字回复"确认"
- **信息展示更结构化**：演练结果、报告分析这类内容，用表格和图表比纯文本清晰得多
- **前后端解耦**：大模型只负责输出结构化数据，展示逻辑完全由前端控制

当然，这种方式对提示词的要求更高——你需要在系统提示词里明确定义每个标签的使用场景和格式规范，并且提供足够的 few-shot 示例，确保大模型严格遵循格式输出。如果格式不对，前端解析会出问题。

实践中，偶尔会遇到大模型输出格式不规范的情况（比如标签没闭合）。我的处理方式是在解析层做了**容错处理**，解析失败的内容 fallback 到普通文本渲染，保证至少信息不会丢。

## 总结

回顾一下整篇文章的核心要点：

### 提示词工程

1. **参数来源分类**：把工具参数按来源拆成用户输入、工具查询、默认值、反问用户四类，让大模型自动补全，减少用户负担
2. **YAML 配置化**：用 YAML 定义工具提示词，embed 嵌入代码，统一格式、减少代码量，也更适合 AI 辅助开发
3. **结构化系统提示词**：按知识、规则、行为、例子四段式设计，各司其职，方便调试定位问题

### 会话管理

1. **Redis 双向链表**：对话历史用 List 存，追加快、截取方便、淘汰简单
2. **HashMap 元数据**：Session 信息用 Hash 存，支持按字段更新
3. **XML 标签驱动渲染**：大模型输出结构化 XML，前端按标签渲染不同 UI 组件，让人机交互更丰富

这些实践不一定适用于所有场景，但如果你也在做**B 端平台的内置 Agent**，希望能给你一些参考。

Agent 编排的部分，下次再聊。

## 参考资料

- [Eino 框架文档](https://github.com/cloudwego/eino)
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)
- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
