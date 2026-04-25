# genCodeDesc 协议——是什么 & 为什么

## ======>>>genCodeDesc 是什么<<<======

- `genCodeDesc` 是一种**外挂的、版本级别的元数据**，用来描述一次提交里哪些行是 AI 生成的。
- 它**不是**仓库内容。它是 `codeAgent`（比如 HuayanCoder）在每次提交之后生产出来的。
- 一条 `genCodeDesc` 记录描述的是**一个具体的版本**。
- 协议名叫 `generatedTextDesc`——它既覆盖源代码（`codeLines`），也覆盖文档文本（`docLines`）。

### 身份标识

每条记录用一个组合键来定位：

- `repoURL` —— 规范的仓库 URL 或本地路径
- `repoBranch` —— 分支名（Git）或分支路径（SVN）
- `revisionId` —— 提交哈希（Git）或版本号（SVN）

### 一条记录里有什么

| 段落 | 干什么用的 |
|---|---|
| `SUMMARY` | 汇总行数：`totalCodeLines`、`fullGeneratedCodeLines`、`partialGeneratedCodeLines`（还有 Doc 对应的那几个） |
| `DETAIL` | 按文件的数组，每个文件有 `fileName` 和 `codeLines`/`docLines` 数组，带着逐行的 `genRatio` 和 `genMethod` |
| `REPOSITORY` | 仓库身份信息：`vcsType`、`repoURL`、`repoBranch`、`revisionId` |
| `CREDENTIAL` | 认证令牌——分析器把它当作信封元数据直接忽略 |

### 行级别的归属信息

- `genRatio` —— 整数 0–100，一行代码的 AI 归属百分比
  - `100` = 纯 AI 生成
  - `0` = 纯人写的
  - `30` = 30% 归功于 AI
- `genMethod` —— 这行是怎么生成的（比如 `codeCompletion`、`vibeCoding`、`Manual`）
- 行可以用 `lineLocation`（单行）或 `lineRange`（`{from, to}`）来定位

---

## ======>>>genCodeDesc 为什么存在<<<======

### 我们想回答的问题

> 在请求的时间段结束时，在指定分支的快照里，**那些在 `[startTime, endTime]` 期间被新增或修改的活代码行，有多大比例归功于 AI 生成？**

### 为什么需要外挂元数据

- 仓库自己知道**哪些行活着**，知道**哪个版本最后改了每一行**（通过 blame）。
- 但仓库**不知道**一行代码到底是人写的还是 AI 生成的。
- AI 归属的信息存在 `genCodeDesc` 记录里，是 `codeAgent` 在提交时生产的。
- 分析器需要**联合**这两套系统：
  - **仓库** → 哪些行活着，来源是哪个版本
  - **genCodeDesc** → 对于那个来源版本，每一行的 AI 比例是多少

### 核心度量公式

```
AI_Window_Live_Ratio = Sum(line.genRatio / 100，对所有来源版本在 [startTime, endTime] 内的活行)
                       / 窗口内活着的变更行总数
```

---

## ======>>>协议版本<<<======

### v26.03 —— 只记 AI 行（当前基线）

- **文件**：[Protocols/genCodeDescProtoV26.03.json](Protocols/genCodeDescProtoV26.03.json)
- **DETAIL 模型**：只记有 AI 归属的行（`genRatio > 0`）
- **SUMMARY 不变量**：`totalCodeLines >= fullGeneratedCodeLines + partialGeneratedCodeLines`；差值就是从 `DETAIL` 省略的人写/手写行，有效 `genRatio=0`。
- **用于**：算法 A（基于 blame）和算法 B（diff 回放）
- **Blame**：没有嵌进来——blame 在分析时从活的 VCS 里现查

### v26.04 —— 增量式 Add/Delete + 内嵌 Blame（计划中）

- **文件**：[Protocols/genCodeDescProtoV26.04.json](Protocols/genCodeDescProtoV26.04.json)
- **DETAIL 模型**：增量式——每次提交只记**新增**或**删除**的行，不是全量快照
- **用于**：算法 C（内嵌 blame，纯 genCodeDesc，运行时零 VCS 访问）
- **Blame**：内嵌的——codeAgent 在写入时抓取真实的 `git blame`/`svn blame` 输出
- **相比 v26.03 多了什么**：
  - `changeType` 字段：`add` 或 `delete`
  - 每行一个 `blame` 对象：`revisionId`、`originalFilePath`、`originalLine`、`timestamp`，可选 `author`
  - `REPOSITORY.revisionTimestamp` —— 算法 C 的处理顺序必须有
  - `genRatio=0, genMethod=Manual` 表示人写的行（显式记录，不是靠省略）
  - Delete 条目用 `blame.revisionId + originalFilePath + originalLine` 作为身份键，从累积集里移除
  - `blame.originalLineRange` 用于同一来源的连续行批量删除

### v26.03 vs v26.04 —— 关键区别

| | **v26.03** | **v26.04** |
|---|---|---|
| **DETAIL 模型** | 快照式——只记 AI 相关的行（`genRatio > 0`） | 增量式——每次提交记 `add` 和 `delete`，包括人写的行（`genRatio=0, genMethod=Manual`） |
| **Blame** | 没嵌——分析时得跑活的 `git blame`/`svn blame` | 内嵌了——codeAgent 写入时抓的真实 VCS blame |
| **运行时要不要访问 VCS** | 要 | 不要 |
| **运行时要不要 diff 产物** | 不要（算法 A）或要（算法 B） | 不要 |
| **`revisionTimestamp`** | 没有 | 必须有——累积处理顺序靠它 |
| **删除追踪** | 没有——删除在记录里是看不见的 | 有——`changeType=delete` 带着 blame 身份键来从累积集里移除 |
| **人写的行** | 省略（不提 = 人写的） | 显式记录（`genRatio=0, genMethod=Manual`） |

**v26.04 最大的好处**：让 genCodeDesc 文件集**自给自足了**。消费者只要按时间戳顺序解析 v26.04 文件就能算出完全一样的度量——不用检出仓库、不用 VCS 命令行、不用 diff 补丁。这打开了断网、边缘设备和大规模批处理的大门——这些场景下访问仓库要么贵要么根本不行。

**代价**：正确性现在完全依赖 codeAgent 写入时的 blame 准不准。v26.03 的话分析器还能用活的 VCS blame 独立验证行来源；v26.04 这个验证没了。

### 版本兼容性

- v26.04 在协议层面是 v26.03 的**超集**。
- 新增的字段（`changeType`、`blame`）是前向兼容的，但算法 A 和 B 不消费它们。
- JSONC（带注释的 JSON）在加载时接受，方便写文档；分析器输出还是标准 JSON。

---

## ======>>>行所有权规则<<<======

这些规则定义了**哪个版本拥有一行代码**，所有算法和 fork 都得遵守。

| 场景 | 规则 |
|---|---|
| AI 行后来被人改了 | 归属到**更新的**提交（现在人拥有它了） |
| 人的行后来被 AI 重写了 | 归属到**更新的**提交（现在 AI 拥有它了） |
| 行被删了 | **不算**——从度量人口里移除了 |
| 文件改名或移动了 | 行所有权通过 blame 的改名检测**保持不变** |
| 行被复制到另一个文件 | 保守做法：当作复制那次提交**新增的**。血统模式（文件复制检测）是可选的。 |
| 写了 genCodeDesc 之后 force-push 或 amend 了 | 嵌进去的 blame **悄悄过时了**——没有自动纠正 |

核心原则：**blame 回答的永远是"哪个版本最后引入了这一行的当前文本？"**——是逐行的，不是逐提交的。它可以追到本次提交、父提交、一年前的提交、或者从合并分支过来的提交。

---

## ======>>>genMethod 词汇表<<<======

`genMethod` 字段描述一行是怎么生成的。定义了这些值：

| 值 | 含义 |
|---|---|
| `codeCompletion` | 通过行内代码补全生成的（比如 Copilot 的 tab 补全） |
| `vibeCoding` | 通过对话/代理驱动的编码生成的（比如聊天、agentic 流程） |
| `Manual` | 完全人写的——只在 v26.04 里用，因为 v26.04 显式记录人写的行 |

`genMethod` 是自由字符串——fork 可以引入新值（比如 `codeReview`、`refactoring`）而不破坏协议。消费者应该把不认识的值当作合法的。

---

## ======>>>CREDENTIAL 段落<<<======

`CREDENTIAL` 段落是**信封元数据**。规则：

- 分析器**忽略**这个段落——它不属于归属契约。
- fork **不应该**把真实凭据存在提交到任何仓库的 genCodeDesc 文件里。
- 如果元数据传输需要凭据，应该运行时注入，不要持久化到协议文件里。
- 协议模板里的 `accessToken` 只是个**占位符**。

---

## ======>>>校验规则<<<======

拿到一条 genCodeDesc 记录后，消费者**必须校验**它跟请求的目标匹配：

| 字段 | 必须匹配 |
|---|---|
| `REPOSITORY.vcsType` | 活仓库的类型（`git` 或 `svn`） |
| `REPOSITORY.repoURL` | 请求的逻辑仓库身份 |
| `REPOSITORY.repoBranch` | 请求的分支或路径 |
| `REPOSITORY.revisionId` | 请求的版本 |

任何字段不匹配的话，这条记录**必须拒绝**，防止悄悄地把错误的元数据关联到真实的仓库版本上。

---

## ======>>>已知协议歧义<<<======

### v26.04 `lineRange` + blame 的 originalLine 映射

在当前的 v26.04 示例里，一个 `lineRange: {from: 2, to: 14}` 的 add 条目带着 `blame.originalLine: 2`。设计意图是：第 2 行映射到 originalLine 2，第 3 行映射到 originalLine 3，以此类推（连续的一一对应）。这能行是因为 `_detailSemantics.lineRange_constraint` 要求范围内所有行共享相同的 blame 来源，原始行号也是连续的。不过理想情况下协议应该用 `blame.originalLineRange: {from: 2, to: 14}` 来完全消除歧义。fork 应该把 lineRange 条目里的 `blame.originalLine` 当作**连续原始范围的起始位置**。

### v26.04 改名示例

**纯改名**（`old.py` → `new.py`，内容没变）：

改名那次提交的 v26.04 记录**没有 DETAIL 条目**。没有行被增删——增量模型只记内容变化。累积存活集里还是引用着 `originalFilePath: "old.py"`，度量不受影响（genRatio 跟路径无关）。

```jsonc
// 提交 bbb 的 v26.04（纯改名：old.py → new.py）
// DETAIL 是空的——没有行内容变化
{
    "DETAIL": [],
    "REPOSITORY": {
        "vcsType": "git",
        "revisionId": "bbb",
        "revisionTimestamp": "2026-03-16T10:00:00Z"
    }
}
```

**改名 + 改了一行**（`old.py` → `new.py`，第 2 行被人重写了）：

```jsonc
// 提交 bbb 的 v26.04（改名 + 改了第 2 行）
{
    "DETAIL": [
        {
            "fileName": "new.py",
            "codeLines": [
                // 删掉第 2 行的旧版本（用它的 blame 来源做 key）
                {
                    "changeType": "delete",
                    "blame": { "revisionId": "aaa", "originalFilePath": "old.py", "originalLine": 2 }
                },
                // 加上第 2 行的新版本（blame 指向这次提交，路径是新路径）
                {
                    "changeType": "add",
                    "lineLocation": 2, "genRatio": 0, "genMethod": "Manual",
                    "blame": { "revisionId": "bbb", "originalFilePath": "new.py", "originalLine": 2,
                               "timestamp": "2026-03-16T10:00:00Z" }
                }
            ]
        }
    ]
}
// 第 1 行和第 3 行：没变，没条目——blame 还是追到 revision aaa 的 old.py
```

---

## ======>>>版本演进策略<<<======

- **加性变更**（新的可选字段、新的 `_semantics` 文档）：小版本号递增（比如 `26.03` → `26.04`）。
- **破坏性变更**（删字段、改字段语义、改段落名）：需要**大版本号跳升**（比如 `26.xx` → `27.xx`）。
- 不认识的顶级字段当作信封元数据**忽略掉**——这保证了前向兼容。
- 目前还没有正式的 JSON Schema。需要严格校验的 fork 应该自己定义一个。

---

## ======>>>作用域定义<<<======

协议支持多个作用域级别。作用域决定了**什么算作被度量的行**。

| 作用域 | 名称 | 算什么 | 协议字段 |
|---|---|---|---|
| A | 纯源代码 | 源文件里的非空非注释行（`.c`、`.cc`、`.cpp`、`.cxx`、`.go`、`.h`、`.hpp`、`.java`、`.js`、`.py`、`.rs`、`.ts`） | `codeLines` |
| B | 源码 + 注释 | 源文件里所有非空行（包括注释） | `codeLines` |
| C | 文档文本 | 文档文件里的非空行（`.md`、`.rst`、`.txt`） | `docLines` |
| D | 所有文本 | 源文件和文档文件的并集 | `codeLines` + `docLines` |

---

## ======>>>出处<<<======

协议定义引自 [AggregateGenCodeDesc](https://github.com/EnigmaWU/MyLLM_Arena/tree/main/MyStartups/AggregateGenCodeDesc)。
行所有权规则提炼自 README §4 "Line ownership rules" 和 README_UbiLang.md。
