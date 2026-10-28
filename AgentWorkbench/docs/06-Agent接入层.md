# 06 Agent 接入层

> **所属**：AgentWorkbench 开发文档 › 第三部分 智能体
> **读这篇之前**：[02 技术选型与总体架构](./02-技术选型与总体架构.md)
> **读完你能**：用一个可替换的客户端接上任意 OpenAI 兼容模型，写出系统提示词与上下文预算，实现一个带确认门的 Agent 循环，并知道七种失败该怎么处理
> **相关文件**：[返回目录](../README.md) ｜ [上一篇：差异引擎](./05-差异引擎.md) ｜ [下一篇：工具与权限门](./07-工具与权限门.md)

---

## 6.1 这一层的唯一目标

**把模型降级成一个可替换的零件。**

具体表现为三件事：换模型只改配置不改代码；模型挂了程序还能用（进 Mock）；模型说的话一律当"建议"，不当"事实"。做到这三件，AI 部分就不再是黑盒，而是一段能测试、能回滚、能计费的普通代码。

---

## 6.2 客户端：一个接口，多个 Provider

.NET 里的标准抽象是 `IChatClient`（`Microsoft.Extensions.AI`）。它只有两个方法有意义：`GetResponseAsync` 和 `GetStreamingResponseAsync`。所有 provider 的适配都收敛到 `AsIChatClient()` 这一句。

**`src/AgentWorkbench.Agents/Chat/ChatClientFactory.cs`**

```csharp
using Microsoft.Extensions.AI;
using Microsoft.Extensions.Options;
using OpenAI;
using System.ClientModel;

public interface IChatClientFactory
{
    IChatClient Create();          // 调用方负责 Dispose
    bool IsMock { get; }
}

public sealed class ChatClientFactory(IOptions<AgentOptions> options) : IChatClientFactory
{
    public bool IsMock => !options.Value.IsConfigured;

    public IChatClient Create()
    {
        var o = options.Value;
        if (!o.IsConfigured) return new MockChatClient();

        var client = new OpenAIClient(
            new ApiKeyCredential(o.ApiKey!),
            new OpenAIClientOptions { Endpoint = new Uri(o.BaseUrl) });

        return client.GetChatClient(o.Model).AsIChatClient();
    }
}
```

只要 provider 支持 OpenAI 兼容协议，改 `BaseUrl` 和 `Model` 就能换。常见几家：

| Provider | BaseUrl 示例 | 备注 |
| --- | --- | --- |
| OpenAI | `https://api.openai.com/v1` | 基准 |
| DeepSeek | `https://api.deepseek.com/v1` | 工具调用支持较好，中文强 |
| Moonshot | `https://api.moonshot.cn/v1` | 长上下文 |
| 阿里云百炼 | `https://dashscope.aliyuncs.com/compatible-mode/v1` | 需用兼容模式地址 |
| Ollama（本地） | `http://localhost:11434/v1` | 无需 Key，演示离线可用 |

地址以各家当期文档为准，这里只作形如示例。**上线前必须验证一件事**：目标模型是否稳定支持 tool calling。不支持 tool calling 的模型，这个 Demo 一行都跑不了。

---

## 6.3 Mock：让演示不依赖网络

Mock 不是"测试用的假东西"，它是**演示的保险**。现场网络抖动、API 欠费、模型限流，任何一件都能让演示翻车，而 Mock 永远在。

**`src/AgentWorkbench.Agents/Chat/MockChatClient.cs`（骨架）**

```csharp
public sealed class MockChatClient : IChatClient
{
    public ChatClientMetadata Metadata => new("mock", null, "mock-1");

    public Task<ChatResponse> GetResponseAsync(
        IEnumerable<ChatMessage> messages, ChatOptions? options = null,
        CancellationToken ct = default)
    {
        var lastUser = messages.LastOrDefault(m => m.Role == ChatRole.User)?.Text ?? "";
        var response = MockScripts.Respond(lastUser, messages);
        return Task.FromResult(response);
    }

    public async IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IEnumerable<ChatMessage> messages, ChatOptions? options = null,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        var response = await GetResponseAsync(messages, options, ct);
        foreach (var text in Chunk(response.Text, 8))
        {
            yield return new ChatResponseUpdate(ChatRole.Assistant, text);
            await Task.Delay(20, ct);        // 模拟打字，演示时更像真的
        }
    }

    public object? GetService(Type serviceType, object? serviceKey = null) => null;
    public void Dispose() { }
}
```

三条纪律：

1. **Mock 只覆盖演示脚本里的三句话**，其他输入回一句"当前为 MOCK 模式，仅支持演示脚本中的指令"。不要试图让它"什么都能答"——那会变成第二个模型，还得调它。
2. **界面上必须显眼标注 `MOCK`**，标题栏或状态栏常驻。绝不能让观看者以为这是真模型的效果。
3. **Mock 脚本针对 `samples/employees.v1.csv` 写死**（行号 E1003 之类）。换数据就不适用，这是明确边界，写进 README。

---

## 6.4 系统提示词

提示词是这一层唯一"没有类型检查"的代码，所以更要当代码对待：存文件、进版本库、改动要能 diff。

**`src/AgentWorkbench.Agents/Prompts/system.md`**

```markdown
你是 AgentWorkbench 的表格助理，只操作当前打开的这一张表。

## 能力边界
- 你只能通过工具访问当前文档。不能读其他文件、不能执行命令、不能联网。
- 涉及修改时，必须调用 propose_changes。不要声称"已经改好了"——是否生效由用户决定。

## 工作方式
1. 每轮开始，若尚未了解表结构，先调用 get_table_schema。
2. 需要具体数据时用 query_rows 或 get_row，不要凭印象回答。
3. 回答涉及数字时，注明依据的行（工号）。没有依据就说"表里没有这项数据"。
4. 用户要求修改时：
   - 先用 query_rows 确认目标行与范围，再提议；
   - 一次 propose_changes 只处理一个意图，ops 不超过 50 条；
   - 目标行不确定、或新值需要业务判断时，调用 ask_user 提问，不要猜。

## 规则（不可违反）
- 只读列（工号、入职日期）不可修改，遇到时直接说明原因，不要尝试绕过。
- 必填列不可置空。
- 不要修改用户没有提到的行。用户说"研发部"，就不要顺手改"平台研发"。
- 不要为了让数字好看而编造或"修正"数据。

## 输出
- 简体中文，简短。不要复述整张表。
- 提议修改后，用一句话说明："将修改 N 行共 M 处，请在下方确认。"
- 不要输出 JSON，除非用户明确要求。
```

四段结构（边界 / 工作方式 / 规则 / 输出）比"请你作为一名专业助理……"有效得多。原因很朴素：模型对**否定句和可判定条件**的遵循度，远高于对形容词的遵循度。"不要修改用户没有提到的行"可判定，"要谨慎"不可判定。

温度设 0（见 03.4）。改数据不需要创意，需要可复现。

---

## 6.5 上下文预算

模型的上下文不是免费的，也不是无限的。Demo 定的预算：

| 内容 | 预算 | 策略 |
| --- | --- | --- |
| 系统提示词 | ≈ 800 tokens | 固定，每轮都带 |
| 表结构摘要 | ≈ 300 tokens | 只给列名、类型、行数、主键，**不给全表** |
| 单条工具结果 | ≤ 2000 字符 | `query_rows` 默认 `limit=20`，只返回请求的列 |
| 对话历史 | 最近 8 轮 | 超了压缩 |
| 合计上限 | 24000 tokens | 超过触发压缩，压缩后仍超则拒绝本轮并提示 |

**永远不要把整张表塞进提示词。** 5000 行 × 7 列大约 30 万字符，既超限又贵，还会让模型开始"总结"而不是"查证"。正确姿势是给结构、给工具，让它自己查。

压缩规则，三条，按顺序执行：

1. 保留系统提示词与**第一条**用户消息（原始意图），永不压缩。
2. 保留最近 6 条消息原文。
3. 中间的工具结果各留前 200 字符，加一行 `…（已截断，原 1240 字符）`。

压缩会丢信息，所以要记日志：`context.compressed` 事件写清压缩前后 token 数。看到这个事件频繁出现，说明该缩小查询范围或换长上下文模型了。

---

## 6.6 Agent 循环

状态机只有五个状态，别搞复杂：

```
Idle ──用户发送──▶ Thinking ──有工具调用──▶ ToolRunning
  ▲                   │                        │
  │                   └──无工具调用──▶ Done     ├─只读工具──▶ Thinking
  │                                            └─变更提议──▶ AwaitingConfirm
  │                                                                 │
  └──────────用户接受/拒绝（应用或丢弃补丁）──────────────────────────┘
```

三个要点：

- **`AwaitingConfirm` 是一个真实状态**，不是弹窗。它会中断循环、把控制权交回界面。用户不点，循环就停在那儿，不消耗 token。
- **只读工具不需要确认**，直接执行、结果回灌、继续下一轮。这是体验流畅的关键：查数据不该被打断。
- **变更提议一出，本轮结束**。不要在同一轮里既提议又继续查——用户看到两处待确认会不知道先看哪个。

**`src/AgentWorkbench.Agents/Runtime/AgentSession.cs`（核心循环）**

```csharp
public sealed class AgentSession(
    IChatClient chat,
    IToolDispatcher dispatcher,
    AgentOptions options,
    ILogger<AgentSession> logger)
{
    private readonly List<ChatMessage> _messages = [new(ChatRole.System, PromptLoader.System)];
    private ChatOptions? _chatOptions;

    public IReadOnlyList<ChatMessage> Transcript => _messages;
    public AgentState State { get; private set; } = AgentState.Idle;

    public void ConfigureTools(IList<AITool> tools) =>
        _chatOptions = new ChatOptions
        {
            Tools = tools,
            Temperature = (float)options.Temperature,
            MaxOutputTokens = options.MaxOutputTokens
        };

    public async Task<AgentTurn> SendAsync(string userText, CancellationToken ct)
    {
        _messages.Add(new ChatMessage(ChatRole.User, userText));
        State = AgentState.Thinking;

        for (var turn = 0; turn < options.MaxTurns; turn++)
        {
            var response = await chat.GetResponseAsync(_messages, _chatOptions, ct);
            _messages.AddMessages(response);

            var calls = response.Messages
                .SelectMany(m => m.Contents)
                .OfType<FunctionCallContent>()
                .ToList();

            if (calls.Count == 0)
            {
                State = AgentState.Idle;
                return AgentTurn.Completed(response.Text ?? string.Empty);
            }

            foreach (var call in calls)
            {
                State = AgentState.ToolRunning;
                var outcome = await dispatcher.DispatchAsync(call, ct);

                _messages.Add(new ChatMessage(ChatRole.Tool,
                    [new FunctionResultContent(call.CallId, outcome.Payload)]));

                if (outcome.Kind == ToolOutcomeKind.AwaitingConfirmation)
                {
                    State = AgentState.AwaitingConfirm;
                    return AgentTurn.AwaitingConfirmation(outcome.ProposalId!, response.Text);
                }
            }

            State = AgentState.Thinking;
        }

        State = AgentState.Idle;
        logger.LogWarning("达到最大轮次 {MaxTurns}，中止本轮", options.MaxTurns);
        return AgentTurn.Failed("模型在限定轮次内没有完成，请把问题拆小一点再试。");
    }
}
```

两个容易写错的地方：

- `_messages.AddMessages(response)` 而不是只加 `response.Text`。工具调用的 `CallId` 必须原样保留在历史里，否则下一轮的工具结果对不上号，provider 会直接报 400。
- `FunctionResultContent` 的第一个参数是 `call.CallId`，不是工具名。写错的表现是所有工具调用都返回"找不到对应的调用"。

---

## 6.7 工具结果怎么回灌

模型看不到你的 `Exception`，只看得到你回给它的那段文本。所以回灌格式要设计，不能 `ex.ToString()` 一贴了事。

**成功（只读）**

```json
{ "ok": true, "rows": [ { "工号": "E1003", "姓名": "李娜", "月薪": "18000" } ], "total": 1 }
```

**校验失败（可自修）**

```json
{
  "ok": false,
  "error": "validation_failed",
  "issues": [
    { "code": "TYPE_MISMATCH", "opIndex": 0, "column": "月薪", "message": "「两万」不是数字，请给出数值" },
    { "code": "READONLY_COLUMN", "opIndex": 2, "column": "工号", "message": "该列只读" }
  ],
  "hint": "修正 issues 后重新调用 propose_changes；不要逐条重试。"
}
```

**被用户拒绝**

```json
{ "ok": false, "error": "rejected_by_user", "reason": "只同意前三条", "proposalId": "p-20261105-003" }
```

**不可恢复**

```json
{ "ok": false, "error": "row_not_found", "rowId": "E9999", "candidates": ["E1009", "E1099"] }
```

`candidates` 是给模型的台阶：它编了一个不存在的工号，你回一个相近的真实工号，下一轮它大概率能改对。这比单纯报错有效得多。

---

## 6.8 七种失败与处理

| 症状 | 真实原因 | 处理 | 用户看到什么 |
| --- | --- | --- | --- |
| 不调工具，直接编数据 | 提示词没约束住 / 模型弱 | 强化 6.4 的"没有依据就说没有"；换支持 tool calling 的模型 | 回答下方标"未经验证" |
| 工具参数 JSON 非法 | 模型生成能力不足 | 捕获解析异常，回灌 schema 摘要，重试 1 次 | "正在重试" |
| 编造 rowId | 上下文里没有真实行 | 回 `row_not_found` + `candidates` | "未找到该行，已给出相近选项" |
| 反复调同一个工具 | 陷入循环 | 连续 3 次相同（工具+参数）即中断 | "模型在原地打转，已停止" |
| 超过最大轮次 | 任务太大 | 中断但保留已生成的提议 | "请把问题拆小" |
| 429 / 超时 | 限流或网络 | 指数退避 3 次（1s、2s、4s） | "网络不稳定，重试中" |
| 流式中断 | 连接断 | 保留已收文本，标记不完整 | 消息末尾显示"（回复中断）" |

一张表里最该重视的是第一行和第四行：前者意味着这个 Demo 失去了意义（模型在编数据），后者意味着你的提示词或工具设计有问题。

---

## 6.9 成本与延迟预算

不写具体金额（每家单价一直在变），只写**用量**，你按自己的模型单价换算。

| 场景 | 输入 tokens | 输出 tokens | 工具调用次数 |
| --- | --- | --- | --- |
| 问一个事实（"研发部几个人"） | ≈ 2.5k | ≈ 200 | 2 |
| 一次修改（"研发部涨 5%"） | ≈ 5k | ≈ 600 | 3–4 |
| 一次多轮修正 | ≈ 9k | ≈ 1.2k | 6–8 |

延迟目标（演示标准，不是性能指标）：

- 首字 < 1.5 秒
- 纯问答完整回答 < 8 秒
- 一次改数据从发送到出现预览 < 12 秒

超了先看是不是工具返回的数据太多。`query_rows` 一次返回 500 行，模型光读完就要好几秒，还容易"总结"出错。

---

## 本篇要点

- 客户端只是一个 `IChatClient`，换 provider 改 `BaseUrl` + `Model`，代码不动。
- Mock 是演示的保险，只覆盖脚本里的三句话，界面上必须标 `MOCK`。
- 系统提示词按"边界 / 工作方式 / 规则 / 输出"四段写，多用可判定的否定句。
- 上下文只给结构和工具，**永不给全表**；超预算先压缩工具结果。
- 循环五个状态，`AwaitingConfirm` 是真实状态，变更提议一出本轮就结束。
- 工具结果按 `ok/issues/hint` 结构化回灌，让模型能自己改对。

---

[返回目录](../README.md) ｜ [上一篇：差异引擎](./05-差异引擎.md) ｜ [下一篇：工具与权限门 →](./07-工具与权限门.md)
