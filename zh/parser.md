# LRCX v1 解析指南

本文档面向解析器实现者。它把 RFC 中的思路整理为语言无关的实现指南，并把“规范要求”和“实现建议”分开叙述。

## 1. 解析目标

一个 LRCX 解析器的目标不是只读出文本，而是稳定生成一个结构化文档：

- 头部元数据
- 已声明的标签族
- 按时间排序的歌词组
- 每个歌词组中的主行、翻译、注音、时间轨、背景轨、自定义标记

## 2. 结果状态与解析模式

## 2.1 结果状态

- `Success`
  - 整个文件按当前模式成功完成解析
- `Partial`
  - 已得到可用的前半结果，但在 `Standard` 模式下因错误停止
- `Fragment`
  - `Loose` 模式跳过错误行后仍保留了若干可用片段
- `Abort`
  - 无法得到当前模式要求的最小结果

## 2.2 模式语义

### `Strict`

- 目标：最大一致性、最少歧义
- 建议行为：
  - 头部错误立即终止
  - 主体错误立即终止
  - 不容忍未声明标记、无效时间精度、主行冲突
- 常见结果：`Success` / `Abort`

### `Standard`

- 目标：符合规范，同时允许有限兼容
- 建议行为：
  - 头部致命错误终止
  - 主体遇到致命结构错误返回部分成功的流式结果 `Partial`
  - 非关键格式问题可记录警告
- 常见结果：`Success` / `Partial` / `Abort`

### `Loose`

- 目标：尽量取回可用信息
- 建议行为：
  - 可跳过坏行
  - 可记录被跳过的原始行和原因
  - 继续装配后续合法歌词组
- 常见结果：`Success` / `Fragment` / `Abort`

## 3. 规范化的数据模型

规范层不要求具体语言类型系统，但建议至少能表达下列逻辑结构：

```text
Document
  version
  head
    metadata
    declarations
      voices
      translations
      phonetics
      customMarks
      options
      easing
    credits
    comments
  body
    lineGroups[]

LineGroup
  timeMs
  durationMs?
  rawOrder[]
  main
  translations{}
  phonetics{}
  timing[]
  backgrounds[]
  marks[]
  attributes{}

MainTrack
  kind = text | timing | supermark | ref
  text
  voice?
  refTimeMs?

TimingToken
  tokens
  startMs
  durationMs
  easing?
  haltMs?
  haltScope = space | char
```

关键原则：

- `HEAD` 负责定义名字空间。
- `BODY` 负责消费这些名字空间并生成内容结构。
- 同一时间点的多行应先装配为 `LineGroup`，再做主行冲突检查。

## 4. 总体流程

推荐的实现顺序：

1. `GlobalScan`
2. `ResolveHead`
3. `ResolveBody`
4. `AssembleLineGroups`
5. `ResolveReferences`
6. `Finalize / Validate`

## 5. GlobalScan

`GlobalScan` 的职责：

- 找到文件中的第一条合法方括号行
- 切分原始行
- 识别 `HEAD` / `[---] v1.0` / `BODY`
- 提前判断是否根本不像 LRCX

### 规范要求

- 版本分隔行必须精确匹配 `[---] v1.0`
- 分隔行前的原始行属于 `HEAD`
- 分隔行后的原始行属于 `BODY`
- 若缺少分隔行，则该文本不是完整的 LRCX 文档

### 实现建议

- 使用游标扫描而不是用一个巨型正则整体匹配全文
- 保留原始行号与列号，便于报错
- 在 `Loose` 模式下可保留空行，但不要把空行解释为歌词行

## 6. ResolveHead

`ResolveHead` 负责解析头部声明和元数据。

### 6.1 头部分类

建议将 `HEAD` 行按用途分成四类：

- 标准元数据
- 声明行
- 贡献者/注释
- 未知扩展键

### 6.2 标准元数据

至少识别以下标准键：

- `#title`
- `#titlesort`
- `#artist`
- `#artistsort`
- `#album`
- `#lyricist`
- `#composer`
- `#date`
- `#releasetype`
- `#label`
- `#copyright`
- `#language`

额外键建议通过 `#more:*` 存储。

### 6.3 声明解析

解析器应建立可查询的声明表：

- `voice map`
- `translation map`
- `phonetic map`
- `custom mark map`
- `mark option map`
- `easing map`

### 6.4 校验

应至少做以下校验：

- 同一声明名不可重复
- `#OPTION` 引用的目标标签应存在
- `#phonetic` 左右两侧数量应一致
- `#voice` 左右两侧数量应尽量一致；不一致时保留已给出的说明文本

## 7. ResolveBody

`BODY` 解析建议拆成两个层面：

1. 单行解析
2. 同时间点装配

### 7.1 单行解析

对于每条 `BODY` 原始行：

1. 解析时间标签与可选持续时长
2. 解析一个或多个 `ContentTag`
3. 将剩余文本作为 `ContentText`
4. 根据首个“内容类型标签”判断该行属于哪种轨道

### 7.2 内容类型优先级

建议把首个内容类型标签视为该行的主内容类型，后续标签作为修饰信息。

主要内容类型包括：

- 纯文本主行
- `#timing`
- 翻译轨
- 注音轨
- `#back`
- `#ignore`
- `#ref`

修饰标签包括：

- 声部标签，例如 `#A`
- 自定义标签，例如 `#chorus`
- 自定义属性标签

## 8. AssembleLineGroups

解析器应把时间值相同的行装配到同一个歌词组中。

### 8.1 装配步骤

1. 按时间顺序读取行
2. 将相同 `timeMs` 的行挂入同一个 `LineGroup`
3. 保留原始行顺序，便于冲突诊断
4. 汇总主行、翻译、注音、背景、标签

### 8.2 主行归一化

同一个 `LineGroup` 最终必须只有一个主行语义：

- 若纯文本主行存在，它通常是优先主行
- 若没有纯文本主行，可由 `#timing` 重建文本
- 若没有纯文本主行，也可由 `supermark` 重建文本
- 若使用 `#ref`，则主行内容来自被引用组

如果同一时间点出现多个主行候选且文本不一致：

- `Strict`：应报错并终止
- `Standard`：应返回 `Partial`
- `Loose`：可保留首个合法主行并记录冲突

## 9. ResolveLineContent

不同轨道应有各自的子解析器。

## 9.1 文本主行

- 读取纯文本
- 处理转义
- 记录声部与自定义标签

## 9.2 翻译轨

- 由标签映射到翻译语言键
- `[#zh]`、`[#en]` 映射到具名语言
- `[#trans]` 映射到未指定语言的默认槽位

## 9.3 注音轨

解析前先从 `HEAD` 查询其声明模式：

- `brief`
- `supermark`
- `full`

然后分发给对应子解析器。

## 10. Timing 解析

## 10.1 基础规则

`#timing` 和 `full` 注音共享同一套时间片段语法。

每个时间标记都作用于其前一个文本片段。

```text
text<duration>
text<duration:easing>
text<duration:easing.halt>
text<duration:.halt>
```

### 规范检查

- 标记不可嵌套
- `duration` 必须是非负整数
- `easing` 若出现，必须能在声明表中找到
- 若出现 `.halt`，其值必须可解析为数值

### 推荐输出

对子解析器而言，最稳定的输出不是“字符数组”，而是时间片段数组：

```text
[
  { tokens: "今更", startMs: 0, durationMs: 1912, easing: "eo1" },
  { tokens: " ", startMs: 1912, durationMs: 424 },
  { tokens: "思い", startMs: 2336, durationMs: 1785, easing: "eo1" }
]
```

这样更容易：

- 重建文本
- 计算持续时长
- 做播放器高亮
- 做调试输出

## 11. `brief` / `supermark` / `full` 注音解析

### `brief`

- 读取整行注音文本
- 可解析基础 `<duration>`
- 推荐输出：`text + timing[]`

### `supermark`

- 识别 `<^...>` 开头的注音片段
- 识别后续以 `^` 结束的目标歌词片段
- 推荐输出：若干“注音片段 -> 目标文本区间”的映射

### `full`

- 直接复用 `#timing` 子解析器
- 不同点仅在于输出被记录到某个注音轨，而不是主时间轨

## 12. 背景人声 `#back`

`#back:offset_ms` 行附着在当前时间点的歌词组上。

解析器应：

- 读取偏移毫秒值
- 解析其后时间片段
- 将其存储为附加背景轨

注意：

- `offset_ms` 可以为负
- 背景轨不应替代主行

## 13. 引用 `#ref`

`#ref:time` 的实现建议：

1. 在第一遍 `BODY` 解析中记录引用关系
2. 在全部歌词组装配后再做引用解析
3. 要求只能引用已经出现过的时间点

引用解析后的结果至少应保留：

- 当前组自己的 `timeMs`
- 当前组自己的 `durationMs`
- 被引用组的来源时间点
- 实际展开后的主行与可复用附加内容

## 14. 错误类别建议

语言无关的错误类别至少建议包括：

- `InvalidFile`
- `VersionUnmatch`
- `InvalidLine`
- `BlankLine`
- `SyntaxError`
- `UnsupportedAccuracy`
- `Nonsequential`
- `NonIdenticalLine`
- `UndeclaredMark`
- `UnknownEasing`
- `InvalidReference`

## 15. 实现建议（非规范）

这些建议来自 RFC 思路整理，不是强制规定：

- 优先使用游标式、自顶向下的状态解析，而不是依赖一条复杂正则
- 保留 `line / column / rawLine`，方便调试与转换回写
- 把“识别行类型”和“解析轨道内容”分层实现
- 先得到稳定的逻辑模型，再考虑播放器专用结构
- 对 `Loose` 模式，单独记录 skipped lines，避免污染主结果

## 16. 验证清单

实现完成后，至少要覆盖这些场景：

- 合法头部与合法版本分隔
- 经典 LRC 风格主行
- 翻译轨
- 声部标记
- `brief`
- `supermark`
- `full`
- 基础 `#timing`
- 带缓动的高级 `#timing`
- 背景人声
- 仅向前的 `#ref`
- 未声明标签报错
- 同时间点主行冲突
- 毫秒精度超限
- 非法嵌套标记
