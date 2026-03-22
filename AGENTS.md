# meta-project

当前项目是一个智能体应用，使用 **Java** 语言实现，包含 **meta-ai** 和 **meta-agent** 两个核心模块。

- **meta-ai**：作为 meta-agent 的大脑，负责将用户自然语言解析为工具调用指令，基于概率模型实现语义理解
- **meta-agent**：作为用户入口，负责接收输入、调用 meta-ai、执行返回的工具指令、反馈结果

这是一个**动态智能交互循环**：用户输入 → meta-ai解析(概率推理) → 执行工具 → 循环提问或者结果反馈

---

# meta-ai 设计

meta-ai 是整个项目的大脑，负责解析用户输入，把用户输入转化成无歧义的指令让meta-agent执行（比如：某种编程语言）。

前提：从大语言模型的原理分析出 AI 能力，是根据概率来生产内容的。

这个模块也是根据概率来生成指令，要求：
不借助 AI 大模型，要实现自然语言对指令的转换，不是硬编码，不是录规则，要根据语义通过概率来智能处理。

## 核心目标

将用户自然语言输入转换为多步骤可执行的结构化指令，核心思路是**模拟大模型根据概率生成内容的机制**：
- 不硬编码规则
- 不录制规则
- 基于语义向量化 + 概率推理

## 整体流程

```
用户输入 → 预处理 → 语义向量化 → 概率推理 → 指令生成 → 歧义处理 → 返回
```

## 1. 文本预处理

**目标**：规范化输入，提取语义特征

```
原始输入 → 规范化 → 分词 → 词性标注 → 依存分析
```

### 1.1 规范化
- 全角转半角
- 去除多余标点
- Unicode标准化

### 1.2 分词
使用 **HanLP**（预训练模型，不自己训练）：
```java
List<Term> terms = StandardTokenizer.segment(input);
```

### 1.3 依存分析
用于指令拆分和参数提取：
```java
// 使用HanLP依存分析
List<Arc> arcs = parser.parse(words, postags);
```

## 2. 语义向量化

**目标**：将文本转为向量，实现语义级别的理解

### 2.1 句子向量

将句子转换为固定维度向量：

**方式一：TF-IDF加权**
- 词频 × 逆文档频率
- 突出关键词，抑制常见词

**方式二：词向量平均**
- 使用预训练中文词向量（搜狗词向量，不自己训练）
- 对句子中所有词向量加权平均
```java
// 加载预训练词向量
WordVectorModel model = new WordVectorModel("sgns.sogou.char");
// 计算句子向量 = 词向量加权平均
Vector sentenceVec = averageVectors(wordVectors, weights);
```

### 2.2 语义相似度计算

```java
// 余弦相似度
double similarity = cosine(vectorA, vectorB);
```

## 3. 概率推理

### 3.1 意图识别

将用户输入与所有工具的 capability 进行向量相似度计算：

```
用户输入向量 → 与工具capability向量计算相似度 → Softmax归一化 → 意图概率分布
```

```java
public Map<String, Double> computeIntentProbabilities(
    Vector inputVector, 
    List<Tool> tools
) {
    Map<String, Double> scores = new HashMap<>();
    for (Tool tool : tools) {
        Vector capVector = getToolCapabilityVector(tool);
        scores.put(tool.getName(), cosine(inputVector, capVector));
    }
    return softmax(scores);  // 归一化为概率
}
```

### 3.2 参数提取

基于依存分析结果，提取参数：

```
依存关系 → 语义角色 → 参数映射
```

- SBV（主谓关系）：执行者
- VOB（动宾关系）：受事者
- POB（介宾关系）：目标位置

### 3.3 置信度评估

```java
public double computeConfidence(String intent, Map<String, Double> probs) {
    double topProb = getTopProbability(probs);
    double secondProb = getSecondProbability(probs);
    double gap = topProb - secondProb;
    
    // 综合得分：top概率 + 区分度
    return topProb * (1 + gap);
}
```

## 4. 歧义处理

### 4.1 触发条件
- 最高概率 < 阈值（如 0.6）
- Top-1 与 Top-2 概率接近（如差值 < 0.1）
- 必填参数缺失

### 4.2 生成澄清问题

```java
public List<String> generateClarificationQuestions(
    ParseContext context,
    Map<String, Double> candidates
) {
    // 分析候选意图的关键差异点
    // 生成二选一或三选一问题
    // 例如："您是想查询北京还是上海的天气？"
}
```

## 5. 指令拆分

当输入包含多个指令时，基于依存分析拆分：

```
输入 → 核心动词检测 → 依存树遍历 → 独立指令提取 → 依赖关系分析
```

**示例**：`查询天气然后发邮件`

1. 检测核心动词：查询、发送
2. 依存分析确定主从关系
3. 拆分指令序列
4. 分析数据依赖：`weather.get.output` → `email.send.content`

## 6. 输出格式

```java
public class ParseResult {
    private ResolveStatus status;          // RESOLVED / AMBIGUOUS / FAILED
    private String intent;                 // 意图名称
    private Map<String, Object> params;    // 参数
    private double confidence;             // 置信度
    private List<String> questions;        // 澄清问题
    private List<Instruction> instructions; // 指令序列
}
```

---

# meta-agent 设计

## 核心目标

meta-agent 是整个项目的入口，主要负责接收用户的输入，调用 meta-ai 解析，记录用户的上下文窗口，提供执行工具（第三方 API 接口、编程环境），反馈结果。

Agent Loop 是 meta-agent 的核心机制，将用户请求转化为多步骤的自主执行流程。

```
用户输入 → meta-ai解析 → 工具调用 → 结果反馈 → 循环直到完成
```

#### 核心循环流程（TAOR）

```
┌─────────────────────────────────────────┐
│              Agent Loop                 │
│                                         │
│  ┌──────────┐                           │
│  │  Prompt  │◄──── 用户消息              │
│  └────┬─────┘                           │
│       ▼                                 │
│  ┌──────────┐                           │
│  │  meta-ai │◄──── 系统提示 + 历史消息    │
│  └────┬─────┘                           │
│       ▼                                 │
│  ┌────────────────┐                     │
│  │  stop_reason?  │                     │
│     │       │                           │
│   Done    ┌──────────┐                  │
│           │  工具调用 │                  │
│                │                        │
│                │ 返回结果到消息           │
│                └──────► 循环 ◄──────────│
└─────────────────────────────────────────┘
```

#### 循环终止条件

| stop_reason | 动作 | 含义 |
|-------------|------|------|
| `end_turn` | 停止循环 | 模型已完成任务，返回结果给用户 |
| `tool_use` | 执行工具，继续循环 | 模型需要调用工具 |

### 2. 四个执行阶段

在每次循环中，meta-agent会经历以下四个阶段：

```
Gather → Plan → Act → Verify
  ↑                      │
  └──────────────────────┘
```

- **Gather（收集）**：使用 Read、Glob、Grep、Bash 了解当前状态
- **Plan（规划）**：使用 TodoWrite 创建任务清单，制定执行计划
- **Act（执行）**：使用 Edit、Bash 等工具执行具体操作
- **Verify（验证）**：检查执行结果，必要时循环修正

### 3. 消息管理

```
消息数组 = [
  { role: "user", content: "用户请求" },
  { role: "assistant", content: [工具调用块] },
  { role: "user", content: 工具执行结果 },
  ...
]
```

- 消息数组随循环累积增长
- 工具结果自动追加到消息历史
- 支持并行工具调用（单次可调用多个工具）

### 4. 上下文管理

```
新输入 → 上下文窗口检查 → 历史裁剪 → 新内容追加 → 上下文快照
```

- **窗口大小**：定义最大保留历史条目数（如 100 条）
- **重要性评估**：根据意图类型标记内容权重
- **历史裁剪**：超出窗口时，按权重和时效裁剪旧内容
- **快照机制**：定期保存上下文状态，支持恢复
- **自动压缩**：上下文使用约 90% 时自动触发压缩

### 5. 子智能体机制

对于复杂任务，启用子智能体进行探索：

```
主循环 → Task(子智能体) → 独立上下文 → 结果汇总 → 主循环继续
```

- 子智能体拥有独立的上下文空间
- 探索结果作为工具输出返回主循环
- 避免探索过程污染主会话上下文

### 6. 工具调用机制

```
任务对象 → 工具匹配 → 参数映射 → 工具执行 → 结果标准化 → 反馈生成
```

- **工具注册表**：维护可用工具列表及元信息
- **工具匹配**：根据意图类型选择对应工具
- **参数映射**：将任务参数转换为工具所需格式
- **执行与重试**：调用工具，失败时自动重试（N次）
- **结果标准化**：统一不同工具的返回格式
- **批量执行**：支持多个指令批量执行
- **并行调度**：根据指令依赖关系和并行分组进行并行执行

### 7. 内置工具列表

参照Claude Code 提供以下内置工具，参考设计：

#### 文件操作
| 工具 | 功能 |
|------|------|
| Read | 读取文件内容，支持偏移和行数限制 |
| Write | 写入或覆盖文件内容 |
| Edit | 对文件进行精确的字符串替换 |
| MultiEdit | 批量执行多个编辑操作 |
| Glob | 通过 glob 模式匹配文件路径 |
| Grep | 在文件中搜索正则表达式内容 |

#### 命令执行
| 工具 | 功能 |
|------|------|
| Bash | 执行 shell 命令 |
| BashOutput | 获取后台命令的输出 |
| KillShell | 终止正在运行的 shell 会话 |

#### 网络请求
| 工具 | 功能 |
|------|------|
| WebFetch | 获取 URL 内容，支持格式转换 |
| WebSearch | 执行网络搜索 |

#### 任务管理
| 工具 | 功能 |
|------|------|
| Task | 启动子任务/子智能体 |
| TodoWrite | 创建和管理任务列表 |
| ExitPlanMode | 退出计划模式 |

#### 高级工具
| 工具 | 功能 |
|------|------|
| Tool Search | MCP 工具动态发现 |
| Explore | 代码库探索智能体 |

### 8. 工具权限管理

- **allow**：自动允许执行的操作
- **ask**：执行前需用户确认
- **deny**：禁止执行的操作

示例配置：
```json
{
  "permissions": {
    "allow": ["Bash(npm run *)", "Bash(git status)"],
    "ask": ["Bash(git push:*)", "Bash(npm install *)"],
    "deny": ["Read(./.env*)", "Read(./secrets/**)"]
  }
}
```

### 9. 钩子机制（Hooks）

在工具调用的关键节点注入自定义逻辑：

| 钩子点 | 时机 |
|--------|------|
| `UserPromptSubmit` | 用户提交输入后处理前 |
| `PreToolUse` | 工具执行前（可拦截/修改） |
| `PostToolUse` | 工具执行后 |
| `Pause` | 模型暂停时 |
| `Resume` | 模型恢复时 |

### 10. 结果反馈

```
执行结果 → 结果评估 → 格式化输出 → 流式/批量推送 → 用户展示
```

- **结果评估**：检查执行是否成功，生成状态码
- **格式化**：将结果转为人类可读格式（文本、表格、图表）
- **推送方式**：支持流式输出或批量返回
