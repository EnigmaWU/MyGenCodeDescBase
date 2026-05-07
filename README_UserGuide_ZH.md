# 使用手册 —— `aggregateGenCodeDesc`

面向最终用户的 `aggregateGenCodeDesc` 工具手册。
讲清楚：**什么时候**跑、输入输出**放哪**、12 个（VCS × 访问方式 × 算法）组合**怎么**跑。

这个 BASE 的每个 fork 用各自选定的语言（Python / C++ / Rust）和 CLI 规范实现本合约。

- 相关文档：[README_ZH.md](README_ZH.md) · [README_UserStories_ZH.md](README_UserStories_ZH.md) · [README_AlgABC_ZH.md](README_AlgABC_ZH.md) · [README_Protocol_ZH.md](README_Protocol_ZH.md) · [README_TestGuide_ZH.md](README_TestGuide_ZH.md)

---

## 1. 概览

`aggregateGenCodeDesc` 先从 `startTime..endTime` 期间被新增或修改过的代码行开始，再只保留其中当前版本在 `endTime` 仍然存活的行，然后用三个度量来量化 AI 参与度。

聚合对象是下面两个集合的交集：

1. 由累计 `startTime..endTime` diff 新增或修改过的代码行；
2. 在 `endTime` 仓库快照中仍然存活的代码行。

`commitStart2EndTime.patch` 用来审计时间窗口 diff；JSON 指标只聚合这个 diff 里的存活子集。已删除、已 revert、或来源在窗口之前的行，都不能进入分母。

示例：如果 `startTime=2026-04-01`，`endTime=2026-04-30`，并且分支上对应的 commits 和 `genCodeDescV26.03` 记录分别在 `2026-04-03`、`2026-04-07`、`2026-04-15`，那么：

- `fromCommit` 是 `2026-04-03` 的 commit。
- `toCommit` 是 `2026-04-15` 的 commit。
- `commitStart2EndTime.patch` 覆盖这三个 commit 引入的累计变化。
- JSON 指标只统计这些 commit 新增或修改过、并且当前版本在 `endTime=2026-04-30` 仍然存活的行。

1. **加权模式**：`Σ(genRatio / 100) / totalLines`
2. **纯 AI 模式**：`count(genRatio == 100) / totalLines`
3. **主要 AI 模式**：`count(genRatio >= threshold) / totalLines`

工具由**代码库维护者**调用，用于衡量一段时间窗口内引入了多少 AI 生成的代码。

---

## 2. 输入

### 2.1 强制参数

每个 fork 必须接受以下强制 CLI 参数。参数名在 `--` 前缀之后使用 lower camel case。

| CLI 参数 | 含义 |
| --- | --- |
| `--repoUrl` | Git 或 SVN 仓库 URL。本地模式下传本地路径或 `file://` URL。映射到协议字段 `REPOSITORY.repoURL`。 |
| `--repoBranch` | Git 分支名，或 SVN 路径（如 `trunk`、`branches/rel-1.0`）。 |
| `--startTime` | 窗口开始时间，包含（ISO 8601：`2026-01-01T00:00:00Z`）。 |
| `--endTime` | 窗口结束时间，包含（ISO 8601）。 |
| `--genCodeDescDir` | 本次窗口对应的 genCodeDesc JSON 文件序列所在目录，见 §2.2。 |

### 2.2 `--genCodeDescDir` 约定

这个目录里放的是**一个 genCodeDesc JSON 文件序列**，每次版本一个文件，覆盖 `repoBranch` 上 `[startTime, endTime]` 这段窗口。

- **一个目录只允许一种协议版本。** 要么全是 **v26.03**，要么全是 **v26.04**。混着放的话直接退出码 `2`。
- **发现方式**：目录里所有 `*.json` 文件全加载、解析、校验。
- **排序**：按 `revisionTimestamp` 升序（Alg C 用）。Alg A/B 用仓库自己的拓扑序；目录只用来查 `genRatio`。
- **身份校验**：每个文件的 `REPOSITORY.repoURL` + `repoBranch` + `revisionId` 必须和窗口对得上。对不上报错，对应 AC-006-2。
- **算法兼容性**：

  | 算法 | v26.03 | v26.04 |
  |---|---|---|
  | A（实时 blame） | ✅ | ❌ 拒绝 |
  | B（diff 重放） | ✅ | ❌ 拒绝 |
  | C（内嵌 blame） | ❌ 拒绝 | ✅ 必须 |

- **版本缺失策略**：窗口内某 revision 没有对应 genCodeDesc 文件时：
  - Alg A/B：将该 revision 的行视为 `genRatio=0`（未归因）。可配置：`abort` / `zero` / `skip`。
  - Alg C：链中断，可配置：`abort` / `ignore`。
- **重复 revisionId 策略**：可配置 `reject`（默认）或 `last-wins`。
- **时钟漂移策略**（仅 Alg C）：`revisionTimestamp` 非单调时，可配置 `abort` 或 `ignore`。

### 2.3 可选和按需参数

其他所有 CLI 参数都是可选参数，或仅在所选模式、算法、范围、输出行为、校验策略需要时才使用。

| CLI 参数 | 默认 / 何时需要 | 含义 |
| --- | --- | --- |
| `--threshold` | 按需 | 0..100 的整数，*主要 AI* 模式用。 |
| `--algorithm` | 按需 | 行来源发现策略：`A`、`B` 或 `C`（见 [README_AlgABC_ZH.md](README_AlgABC_ZH.md)）。 |
| `--scope` | 按需 | 文件/路径过滤：`A`、`B`、`C` 或 `D`（见 [README_Protocol_ZH.md](README_Protocol_ZH.md) — 范围定义）。 |
| `--outputDir` | 按需 | 两个输出产物写到的目录（不存在就新建），见 §3。 |
| `--repoPath` | 无 | **仅 Alg A。** 本地仓库工作副本路径。Alg A 跑 git/svn 时需要。若没提供且 `--repoUrl` 是远端，fork 可自动克隆。 |
| `--commitPatchDir` | 无 | **仅 Alg B。** 预先算好的每个 revision 的 unified diff 文件放这里，用于离线重放，见 §2.4。 |
| `--blameWhitespace` | `respect` | **仅 Alg A + Git。** `respect` 或 `ignore`（对应 `git blame -w`），见 AC-004-3。 |
| `--renameDetection` | `basic` | **仅 Git + Alg A/B。** `off` / `basic`（`-M`）/ `aggressive`（`-M -C -C`）。 |
| `--logLevel` | `Info` | stderr 日志详细程度：`Debug` / `Info` / `Warning` / `Error`，见 §2.5。 |
| `--timing` | `summary` | 耗时输出策略：`off`、`summary` 或 `detailed`，见 §2.6 和 §3.1 的 `TIMING`。 |
| `--onMissing` | 算法相关 | 缺失 genCodeDesc 的处理策略，见 §2.2。 |
| `--onDuplicate` | `reject` | 重复 revisionId 的处理策略，见 §2.2。 |
| `--onClockSkew` | `abort` | 仅 Alg C 的时钟漂移处理策略，见 §2.2。 |

协议版本从 `genCodeDescDir` 第一个文件自动检测；所有文件必须版本一致。

### 2.4 `--commitPatchDir` 约定（Alg B）

- **只有 Alg B 会消费这个目录。** Alg A、Alg C 会直接忽略（带一条 warning）。
- 窗口 `[startTime, endTime]` 内每个 revision 一个文件，命名为 `<revisionId>.patch`，覆盖该 revision 相对于其在 `repoBranch` 上父提交的完整变更。
- 单个 `<revisionId>.patch` 可以包含多个文件 diff 区块；每个文件 diff 区块又可以包含多个 hunk（`@@ ... @@`）。Alg B 必须重放 patch 里的每个文件区块和每个 hunk。
- 目录里扫 `*.patch` 文件。
- 顺序：Alg B 从 VCS 历史元数据生成 patch 重放序列，不按目录遍历顺序、文件修改时间或文件名排序。
  - Git：在 `repoBranch` 上按父提交先于子提交的拓扑顺序重放。提交时间用于筛选 `[startTime, endTime]` 窗口，也可以作为确定性 tie-breaker，但不能覆盖父子拓扑顺序。
  - SVN：先用服务器时间戳筛选 `[startTime, endTime]` 内的 revision，再按服务器 revision number 升序重放。
- 缺 revision 走 `--onMissing`；重复走 `--onDuplicate`。
- 这个目录填好之后，Alg B 就可复现、不联网。

### 2.5 `--logLevel` 语义

| 档位 | 打什么 |
|---|---|
| `Debug` | 包含 `Info` 的所有内容，**再加内部调试细节**（解析器 token、原始 VCS 命令行、每文件耗时、哈希表统计、被拒绝的候选项）。非常啰嗦，给工具开发者排查问题用。 |
| `Info`（默认） | **文件加载**事件（每个 genCodeDesc / diff 文件的打开、解析、接受或拒绝）、**逐行状态流转**（每行来源查表、Alg C 的 add/delete 集合变化、Alg B 的 diff hunk 行号追踪）和**最终摘要**（分母、三个度量、诊断信息）。 |
| `Warning` | 只打 warning 和 error（缺 revision、revision 重复、时钟漂移、版本混用、降级结果）。不打任何按文件、按行的内容。 |
| `Error` | 只打导致处理中止的致命错误。 |

### 2.6 `--timing` 语义

Timing 使用单调 wall-clock 计时，并以非负小数秒输出。Timing 只用于可观测性：绝不能改变指标计算、过滤或行归因。

| 取值 | 行为 |
| --- | --- |
| `off` | 聚合 JSON 不输出 `TIMING`，日志也不输出 timing summary。 |
| `summary`（默认） | 在聚合 JSON 中输出稳定的顶层 `TIMING` object，并在最终日志里输出一条 timing summary。 |
| `detailed` | 包含 `summary` 的所有内容，并在 stderr 日志里输出 clone/fetch/checkout、blame、diff replay、genCodeDesc loading、aggregation、output writing 等阶段或命令级耗时。 |

标准 `TIMING` 阶段名如下：

| 字段 | 含义 |
| --- | --- |
| `totalSeconds` | 参数解析后到成功写完输出或报告 fatal error 的端到端耗时。 |
| `cloneRepoSeconds` | 自动 clone、fetch 或 checkout 远端仓库的耗时；未使用时为 `0`。 |
| `checkoutSeconds` | 准备干净 `endTime` 快照或隔离 worktree 的耗时。 |
| `loadGenCodeDescSeconds` | 发现、读取、解析、校验 genCodeDesc 文件的耗时。 |
| `blameSeconds` | 运行实时 `git blame` / `svn blame` 命令的耗时。仅 Alg A 使用；Alg B/C 为 `0`。 |
| `diffSeconds` | 生成或重放 diff/patch 的耗时。 |
| `aggregateSeconds` | 将行来源 join 到 genCodeDesc 条目并计算聚合指标的耗时。 |
| `writeOutputSeconds` | 写出 `aggregatedGenCodeDescV26.03.json` 和 `commitStart2EndTime.patch` 的耗时。 |
| `notRun` | 被所选算法或访问模式跳过的阶段名数组，例如 `cloneRepo`、`blame` 或 `diff`。 |

---

## 3. 输出

当传入 `--outputDir` 时，`outputDir` 里会放 **两个产物**：

| 文件（固定名称） | 是什么 |
|---|---|
| `aggregatedGenCodeDescV26.03.json` | 基于 genCodeDescProtoV26.03 JSON 形状的聚合结果（§3.1）。 |
| `commitStart2EndTime.patch` | `repoBranch` 上 `[startTime, endTime]` 的单个累积 unified diff（§3.2）。 |

两个文件 **Alg A / B / C 都会生成**。

### 3.1 `aggregatedGenCodeDescV26.03.json`

这个输出基于 **[`Protocols/genCodeDescProtoV26.03.json`](Protocols/genCodeDescProtoV26.03.json)**——字段名同、SUMMARY / DETAIL / REPOSITORY 结构同。`aggregated...` 文件名把窗口级聚合产物和单 revision 的 genCodeDesc 记录区分开，同时保持下游工具对 v26.03 形状 JSON 的兼容性。

度量 → 协议字段映射：

| 协议字段 | 聚合含义 |
|---|---|
| `protocolVersion` | `"26.03"`（输出信封的版本，跟输入 `genCodeDescDir` 的版本无关）。 |
| `codeAgent` | `"aggregateGenCodeDesc"`。 |
| `REPOSITORY.repoURL` / `repoBranch` | 原样抄自输入参数。 |
| `REPOSITORY.revisionId` | `"aggregate:<startTime>..<endTime>"` — 标识窗口的合成 id。 |
| `SUMMARY.totalCodeLines` | 分母——`(startTime..endTime diff) ∩ (endTime 仍存活)` 的行数，包括可能从 `DETAIL` 省略的人写/未归因行。 |
| `SUMMARY.fullGeneratedCodeLines` | `genRatio == 100` 的行数（*纯 AI* 的分子）。 |
| `SUMMARY.partialGeneratedCodeLines` | `0 < genRatio < 100` 的行数。 |
| `SUMMARY.totalDocLines` / `fullGeneratedDocLines` / `partialGeneratedDocLines` | 同上，只统文档文件（Scope C/D）。代码和文档计数都满足 `total >= fullGenerated + partialGenerated`；差值就是有效 `genRatio=0` 的人写/未归因行。 |
| `DETAIL[].fileName` | 有归因条目的窗口内活文件。只有被省略的人写/未归因行的文件可以不出现。 |
| `DETAIL[].codeLines[]` / `docLines[]` | 稀疏的 v26.03 归因条目。`genRatio > 0` 的行用 `{lineLocation, genRatio, genMethod}`（或连续行用 `lineRange`）记录。人写/未归因行可以省略；省略即表示有效 `genRatio=0`。 |

**聚合专属扩展**（作为同级顶层 key；只认 v26.03 的老消费者会忽略）：

| 字段 | 含义 |
|---|---|
| `AGGREGATE.window` | `{startTime, endTime}`。 |
| `AGGREGATE.parameters` | `{algorithm, scope, threshold, inputProtocolVersion}`。 |
| `AGGREGATE.metrics.weighted` | `{value, numerator}` — `Σ(genRatio/100) / totalCodeLines`。 |
| `AGGREGATE.metrics.fullyAI` | `{value, numerator}` — `fullGeneratedCodeLines / totalCodeLines`。 |
| `AGGREGATE.metrics.mostlyAI` | `{value, numerator, threshold}` — `count(genRatio >= T) / totalCodeLines`。 |
| `AGGREGATE.diagnostics` | `{missingRevisions[], duplicateRevisions[], clockSkewDetected, warnings[]}`。 |
| `TIMING` | 当 `--timing != off` 时输出的可选顶层 timing summary：`{totalSeconds, cloneRepoSeconds, checkoutSeconds, loadGenCodeDescSeconds, blameSeconds, diffSeconds, aggregateSeconds, writeOutputSeconds, notRun[]}`。 |

例子（窗口内 10 行活代码，已生成行的 `genRatio = [100,100,100,100,100, 80,80,80, 30]`，另有 1 行省略的人写行，有效 `genRatio=0`，阈值 60）：

```json
{
  "protocolName": "generatedTextDesc",
  "protocolVersion": "26.03",
  "codeAgent": "aggregateGenCodeDesc",

  "SUMMARY": {
    "totalCodeLines": 10,
    "fullGeneratedCodeLines": 5,
    "partialGeneratedCodeLines": 4,
    "totalDocLines": 0,
    "fullGeneratedDocLines": 0,
    "partialGeneratedDocLines": 0
  },

  "DETAIL": [
    {
      "fileName": "src/auth.py",
      "codeLines": [
        {"lineRange": {"from": 1, "to": 5}, "genRatio": 100, "genMethod": "codeCompletion"},
        {"lineRange": {"from": 6, "to": 8}, "genRatio":  80, "genMethod": "vibeCoding"},
        {"lineLocation": 9,                 "genRatio":  30, "genMethod": "vibeCoding"}
      ]
    }
  ],

  "REPOSITORY": {
    "vcsType": "git",
    "repoURL": "https://github.com/acme/foo",
    "repoBranch": "main",
    "revisionId": "aggregate:2026-01-01T00:00:00Z..2026-04-01T00:00:00Z"
  },

  "AGGREGATE": {
    "window": {
      "startTime": "2026-01-01T00:00:00Z",
      "endTime":   "2026-04-01T00:00:00Z"
    },
    "parameters": {
      "algorithm": "C",
      "scope": "A",
      "threshold": 60,
      "inputProtocolVersion": "26.04"
    },
    "metrics": {
      "weighted":  {"value": 0.77, "numerator": 7.7},
      "fullyAI":   {"value": 0.50, "numerator": 5},
      "mostlyAI":  {"value": 0.80, "numerator": 8, "threshold": 60}
    },
    "diagnostics": {
      "missingRevisions": [],
      "duplicateRevisions": [],
      "clockSkewDetected": false,
      "warnings": []
    }
  },
  "TIMING": {
    "totalSeconds": 12.384,
    "cloneRepoSeconds": 0.0,
    "checkoutSeconds": 0.0,
    "loadGenCodeDescSeconds": 1.122,
    "blameSeconds": 0.0,
    "diffSeconds": 0.0,
    "aggregateSeconds": 10.734,
    "writeOutputSeconds": 0.082,
    "notRun": ["cloneRepo", "checkout", "blame", "diff"]
  }
}
```

*所有运行时日志写到 stderr，不会干扰 JSON 输出的捕获或管道传递。*

### 3.2 `commitStart2EndTime.patch`

`repoBranch` 上所选 revision 范围的 **单个累积 unified diff**：

- `fromCommit` = `repoBranch` 上时间戳 `>= startTime` 的第一个 commit/revision。
- `toCommit` = `repoBranch` 上时间戳 `<= endTime` 的最后一个 commit/revision。

这个 patch 表示 `fromCommit..toCommit` 引入的净变化。对 Git 来说，为了把 `fromCommit` 自身的改动也包含进去，diff 命令会用 `fromCommit` 的父提交作为左端点：

```text
git diff <parentOfFromCommit>..<toCommit> -- <scope 路径>
```

- **格式**：标准 unified diff（`diff --git ...` / `---` / `+++` / `@@` hunks）。用 `git apply` 或 `patch -p1` 可应用。
- **范围**：按 `scope` 参数过滤（和 JSON 分母的文件范围一致）。
- **三个算法都生成**：

  | 算法 | patch 怎么得来 |
  |---|---|
  | A | 在工作副本或远端调 `git diff` / `svn diff`。 |
  | B | 把 `commitPatchDir` 里的单次 revision diff 按拓扑顺序拼成一个累积 diff，revision 之间加 `# --- commit <rev> ---` 分隔符。 |
  | C | 从 v26.04 的内嵌 add/delete 记录合成：把 `[startTime, endTime]` 上累积的 add/delete 状态序列化为 unified diff。不访问 VCS。 |

- **用途**：和 JSON 配对——JSON 回答*"比例多少"*，patch 回答*"到底算了哪些行"*；两者一起就能审计、复现，不需要再访问仓库。
- **文件头**：patch 首部是一段注释，记录 `repoURL`、`repoBranch`、`startTime`、`endTime`、`algorithm`、`scope`，以及合成 id `aggregate:<start>..<end>`。

---

## 4. 场景矩阵 —— 12 个组合

轴：**VCS** = `git` | `svn` · **访问** = `local` | `remote` · **算法** = `A` | `B` | `C`。

每个组合都消费同一个 §2.2 里说的 `genCodeDescDir` 序列。

### 4.1 git × local

#### git · local · A（实时 blame）

- **前置**：本地有工作副本；`git` 在 PATH；`genCodeDescDir` 是 v26.03。
- **示例形状**：
  ```
  aggregateGenCodeDesc \
    --repoUrl file:///srv/repos/foo \
    --repoBranch main \
    --startTime 2026-01-01T00:00:00Z --endTime 2026-04-01T00:00:00Z \
    --threshold 60 --algorithm A --scope A \
    --genCodeDescDir ./gcd/ \
    --outputDir ./out/
  ```
- **局限**：浅克隆会让 blame 失真（AC-005-4）。

#### git · local · B（离线 diff 重放）

- **前置**：工作副本带窗口内完整对象库；`commitPatchDir` 已填充。
- **局限**：重命名链深时状态会炸（见 README 规模表）。

#### git · local · C（内嵌 blame，仅 v26.04）

- **前置**：**完全不需要 VCS**——`repoUrl` / `repoBranch` 只用来做校验。`genCodeDescDir` 必须是 v26.04。
- **局限**：正确性完全依赖 codeAgent 写入时的 blame。

### 4.2 git × remote

#### git · remote · A

- **前置**：调用方须事先克隆远端（通过 `--repoPath` 传路径），或 fork 自动克隆到工作目录。
- **局限**：浅克隆会让 blame 失真（AC-005-4）。

#### git · remote · B

- **前置**：调用方在 `commitPatchDir` 里事先备好每个 revision 的 patch。工具本身不联网。
- **局限**：patch 的准备是调用方的责任。

#### git · remote · C

- **前置**：**完全不访问远端仓库**。传 `repoUrl`/`repoBranch` 只是为了让结果 JSON 自我标识。
- **推荐**：内网隔离 / 大规模批处理场景。

### 4.3 svn × local

#### svn · local · A

- **前置**：工作副本；`svn` 在 PATH。
- **局限**：`svn blame` 对 merge 过来的行会不精确（见 README Git vs SVN 表）。

#### svn · local · B

- **前置**：工作副本；`commitPatchDir` 通过 `svn diff -rN:M` 填充。
- **局限**：不支持跨文件 move 检测。

#### svn · local · C

- 和 `git · local · C` 一样——不访问 VCS。

### 4.4 svn × remote

#### svn · remote · A

- **前置**：调用方须事先签出 SVN 工作副本。`repoBranch` 要传 SVN 路径（如 `trunk`）。
- **局限**：工具本身不应直接联 SVN 服务器。

#### svn · remote · B

- **前置**：调用方在 `commitPatchDir` 里事先备好每个 revision 的 patch。
- **局限**：patch 的准备是调用方的责任。

#### svn · remote · C

- 和 `git · remote · C` 一样——不访问 VCS。

### 4.5 组合汇总

| # | VCS | 访问 | 算法 | 跑的时候要访问仓库吗？ | 最适合 |
|---|---|---|---|---|---|
| 1 | git | local | A | 要 | 开发迭代 |
| 2 | git | local | B | 要（patch） | 可复现的离线重放 |
| 3 | git | local | C | **不要** | 封闭式 CI |
| 4 | git | remote | A | 要（工作副本） | 按需审计 |
| 5 | git | remote | B | 不要（只要 patch） | 离线批处理 |
| 6 | git | remote | C | **不要** | 内网隔离 / 大规模 |
| 7 | svn | local | A | 要 | svn 开发迭代 |
| 8 | svn | local | B | 要（patch） | svn 离线重放 |
| 9 | svn | local | C | **不要** | 封闭式 CI（svn） |
| 10 | svn | remote | A | 要（工作副本） | svn 按需审计 |
| 11 | svn | remote | B | 不要（只要 patch） | svn 离线批处理 |
| 12 | svn | remote | C | **不要** | 内网隔离（svn） |

---

## 5. 校验和错误分类

对应 [README_UserStories_ZH.md](README_UserStories_ZH.md) US-006：

| 情况 | 参数 | 默认 | 退出码 |
|---|---|---|---|
| 窗口内某 revision 的 genCodeDesc 缺失 | `--onMissing` | `zero`（A/B）/ `abort`（C） | 0 / 2 |
| 文件里 `REPOSITORY` 对不上 | — | 一律拒绝 | 2 |
| `revisionId` 重复 | `--onDuplicate` | `reject` | 2 / 0 |
| `revisionTimestamp` 非单调（Alg C） | `--onClockSkew` | `abort` | 2 / 0 |
| `genRatio` 超出 0..100 | — | 一律拒绝 | 2 |
| 目录里协议版本混用 | — | 一律拒绝 | 2 |
| Alg C 给了 v26.03，或 Alg A/B 给了 v26.04 | — | 一律拒绝 | 2 |

---

## 6. 退出码

| 码 | 含义 |
|---|---|
| 0 | 成功 |
| 1 | 运行时/IO 错误（网络、磁盘、VCS CLI 失败、未捕获异常） |
| 2 | 输入/校验错误（参数错、genCodeDesc 错、版本混用、算法与版本不匹配） |

---

## 7. 各算法快速示例

### 算法 A：实时 Blame（Git，本地工作副本）

```text
aggregateGenCodeDesc \
  --repoUrl https://github.com/acme/myrepo \
  --repoBranch main \
  --startTime 2026-01-01T00:00:00Z \
  --endTime 2026-04-01T00:00:00Z \
  --threshold 60 \
  --algorithm A --scope A \
  --genCodeDescDir ./gcd-v26.03/ \
  --outputDir ./out/
```

### 算法 B：离线 Diff 重放（Git 或 SVN）

```text
aggregateGenCodeDesc \
  --repoUrl https://svn.example.com/repo \
  --repoBranch /trunk \
  --startTime 2026-02-01T00:00:00Z \
  --endTime 2026-02-28T23:59:59Z \
  --threshold 60 \
  --algorithm B --scope A \
  --genCodeDescDir ./gcd-v26.03/ \
  --commitPatchDir ./diffs/ \
  --outputDir ./out/
```

### 算法 C：内嵌 Blame（仅 v26.04，不需访问 VCS）

```text
aggregateGenCodeDesc \
  --repoUrl https://github.com/acme/myrepo \
  --repoBranch main \
  --startTime 2026-01-01T00:00:00Z \
  --endTime 2026-04-01T00:00:00Z \
  --threshold 60 \
  --algorithm C --scope A \
  --genCodeDescDir ./gcd-v26.04/ \
  --outputDir ./out/
```

---

## 8. Fork 如何实现本手册

1. 从 `MyGenCodeDescBase` fork 出来，针对特定 CodeAgent & LLM 组合。
2. 用你选定的语言（Python / C++ / Rust）实现 `aggregateGenCodeDesc`。
3. 你的 fork 的 `README_ZH.md` 引用本手册，并说明：
    - 除上面强制 lower-camel-case CLI 参数之外，是否还支持额外兼容参数。
    - 12 个组合中哪些已支持（目标是全部 12 个；仅支持 Alg C 的 fork 可以跳过第 1、2、7、8 格）。
    - 每个组合的已知局限（如"Alg B 尚未实现"）。
    - `--onMissing`、`--onDuplicate`、`--onClockSkew` 的默认策略。
4. [README_UserStories_ZH.md](README_UserStories_ZH.md) 里的全部 66 条验收标准都是测试目标。
