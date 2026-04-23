# 使用手册 —— `aggregateGenCodeDesc`

面向最终用户的 `aggregateGenCodeDesc` 工具手册。
讲清楚：**什么时候**跑、输入输出**放哪**、12 个（VCS × 访问方式 × 算法）组合**怎么**跑。

这个 BASE 的每个 fork 用各自选定的语言（Python / C++ / Rust）和 CLI 规范实现本合约。

- 相关文档：[README_ZH.md](README_ZH.md) · [README_UserStories.md](README_UserStories.md) · [README_AlgABC_ZH.md](README_AlgABC_ZH.md) · [README_Protocol_ZH.md](README_Protocol_ZH.md)

---

## 1. 概览

`aggregateGenCodeDesc` 分析 `endTime` 时仓库快照中活着的代码，用三个度量来量化 AI 参与度：

1. **加权模式**：`Σ(genRatio / 100) / totalLines`
2. **纯 AI 模式**：`count(genRatio == 100) / totalLines`
3. **主要 AI 模式**：`count(genRatio >= threshold) / totalLines`

工具由**代码库维护者**调用，用于衡量一段时间窗口内引入了多少 AI 生成的代码。

---

## 2. 输入

### 2.1 必填参数

每个 fork 必须接受以下输入（具体 CLI 参数名由各 fork 自定义）：

| 参数 | 含义 |
|---|---|
| `repo-url` | Git 或 SVN 仓库 URL。本地模式下传本地路径或 `file://` URL。 |
| `repo-branch` | Git 分支名，或 SVN 路径（如 `trunk`、`branches/rel-1.0`）。 |
| `start-time` | 窗口开始时间，包含（ISO 8601：`2026-01-01T00:00:00Z`）。 |
| `end-time` | 窗口结束时间，包含（ISO 8601）。 |
| `threshold` | 0..100 的整数，*主要 AI* 模式用。 |
| `algorithm` | 行来源发现策略：`A`、`B` 或 `C`（见 [README_AlgABC_ZH.md](README_AlgABC_ZH.md)）。 |
| `scope` | 文件/路径过滤：`A`、`B`、`C` 或 `D`（见 [README_Protocol_ZH.md](README_Protocol_ZH.md) — 范围定义）。 |
| `gen-code-desc-dir` | 本次窗口对应的 genCodeDesc JSON 文件序列所在目录，见 §2.2。 |
| `output-dir` | 两个输出产物写到的目录（不存在就新建），见 §3。 |

### 2.2 `gen-code-desc-dir` 约定

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

### 2.3 可选参数

| 参数 | 默认 | 含义 |
|---|---|---|
| `repo-path` | 无 | **仅 Alg A。** 本地仓库工作副本路径。Alg A 跑 git/svn 时必须有。若没提供且 `repo-url` 是远端，fork 可自动克隆。 |
| `end-rev` | `HEAD` | **仅 Alg A。** 执行 blame 时的目标 revision。 |
| `commit-patch-dir` | 无 | **仅 Alg B。** 预先算好的每个 revision 的 unified diff 文件放这里，用于离线重放，见 §2.4。 |
| `blame-whitespace` | `respect` | **仅 Alg A + Git。** `respect` 或 `ignore`（对应 `git blame -w`），见 AC-004-3。 |
| `rename-detection` | `basic` | **仅 Git + Alg A/B。** `off` / `basic`（`-M`）/ `aggressive`（`-M -C -C`）。 |
| `log-level` | `Info` | stderr 日志详细程度：`Debug` / `Info` / `Warning` / `Error`，见 §2.5。 |
| `on-missing` | 算法相关 | 缺失 genCodeDesc 的处理策略，见 §2.2。 |
| `on-duplicate` | `reject` | 重复 revisionId 的处理策略，见 §2.2。 |
| `on-clock-skew` | `abort` | 仅 Alg C 的时钟漂移处理策略，见 §2.2。 |

协议版本从 `gen-code-desc-dir` 第一个文件自动检测；所有文件必须版本一致。

### 2.4 `commit-patch-dir` 约定（Alg B）

- **只有 Alg B 会消费这个目录。** Alg A、Alg C 会直接忽略（带一条 warning）。
- 窗口 `[startTime, endTime]` 内每个 revision 一个文件，命名为 `<revisionId>.patch`，覆盖该 revision 相对于其在 `repoBranch` 上父提交的完整变更。
- 目录里扫 `*.patch` 文件。
- 顺序：Alg B 按仓库自己的拓扑/父子顺序重放；文件名只是 revision → diff 的映射，不是排序键。
- 缺 revision 走 `on-missing`；重复走 `on-duplicate`。
- 这个目录填好之后，Alg B 就可复现、不联网。

### 2.5 `log-level` 语义

| 档位 | 打什么 |
|---|---|
| `Debug` | 包含 `Info` 的所有内容，**再加内部调试细节**（解析器 token、原始 VCS 命令行、每文件耗时、哈希表统计、被拒绝的候选项）。非常啰嗦，给工具开发者排查问题用。 |
| `Info`（默认） | **文件加载**事件（每个 genCodeDesc / diff 文件的打开、解析、接受或拒绝）、**逐行状态流转**（每行来源查表、Alg C 的 add/delete 集合变化、Alg B 的 diff hunk 行号追踪）和**最终摘要**（分母、三个度量、诊断信息）。 |
| `Warning` | 只打 warning 和 error（缺 revision、revision 重复、时钟漂移、版本混用、降级结果）。不打任何按文件、按行的内容。 |
| `Error` | 只打导致处理中止的致命错误。 |

---

## 3. 输出

`output-dir` 里会放 **两个产物**：

| 文件（固定名称） | 是什么 |
|---|---|
| `genCodeDescV26.03.json` | 聚合结果，形状跟 genCodeDescProtoV26.03 一样（§3.1）。 |
| `commitStart2EndTime.patch` | `repoBranch` 上 `[startTime, endTime]` 的单个累积 unified diff（§3.2）。 |

两个文件 **Alg A / B / C 都会生成**。

### 3.1 `genCodeDescV26.03.json`

形状跟 **[`Protocols/genCodeDescProtoV26.03.json`](Protocols/genCodeDescProtoV26.03.json)** 一样——字段名同、SUMMARY / DETAIL / REPOSITORY 结构同。聚合结果复用这个协议，这样已经能读版本级 genCodeDesc 的下游工具不用改就能读聚合结果。

度量 → 协议字段映射：

| 协议字段 | 聚合含义 |
|---|---|
| `protocolVersion` | `"26.03"`（输出信封的版本，跟输入 `gen-code-desc-dir` 的版本无关）。 |
| `codeAgent` | `"aggregateGenCodeDesc"`。 |
| `REPOSITORY.repoURL` / `repoBranch` | 原样抄自输入参数。 |
| `REPOSITORY.revisionId` | `"aggregate:<startTime>..<endTime>"` — 标识窗口的合成 id。 |
| `SUMMARY.totalCodeLines` | 分母——窗口内活代码行数。 |
| `SUMMARY.fullGeneratedCodeLines` | `genRatio == 100` 的行数（*纯 AI* 的分子）。 |
| `SUMMARY.partialGeneratedCodeLines` | `0 < genRatio < 100` 的行数。 |
| `SUMMARY.totalDocLines` / `fullGeneratedDocLines` / `partialGeneratedDocLines` | 同上，只统文档文件（Scope C/D）。 |
| `DETAIL[].fileName` | 窗口内每个活文件。 |
| `DETAIL[].codeLines[]` / `docLines[]` | 逐行的 `{lineLocation, genRatio, genMethod}`（或连续行用 `lineRange`），从来源 revision 的 genCodeDesc 拷过来。 |

**聚合专属扩展**（作为同级顶层 key；只认 v26.03 的老消费者会忽略）：

| 字段 | 含义 |
|---|---|
| `AGGREGATE.window` | `{startTime, endTime}`。 |
| `AGGREGATE.parameters` | `{algorithm, scope, threshold, inputProtocolVersion}`。 |
| `AGGREGATE.metrics.weighted` | `{value, numerator}` — `Σ(genRatio/100) / totalCodeLines`。 |
| `AGGREGATE.metrics.fullyAI` | `{value, numerator}` — `fullGeneratedCodeLines / totalCodeLines`。 |
| `AGGREGATE.metrics.mostlyAI` | `{value, numerator, threshold}` — `count(genRatio >= T) / totalCodeLines`。 |
| `AGGREGATE.diagnostics` | `{missingRevisions[], duplicateRevisions[], clockSkewDetected, warnings[]}`。 |

例子（窗口内 10 行活代码，`genRatio = [100,100,100,100,100, 80,80,80, 30, 0]`，阈值 60）：

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
        {"lineLocation": 9,                 "genRatio":  30, "genMethod": "vibeCoding"},
        {"lineLocation": 10,                "genRatio":   0, "genMethod": "Manual"}
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
  }
}
```

*所有运行时日志写到 stderr，不会干扰 JSON 输出的捕获或管道传递。*

### 3.2 `commitStart2EndTime.patch`

从窗口开始前的父提交到窗口末尾 revision 的 **单个累积 unified diff**，作用在 `repoBranch` 上。等价于：

```text
git diff <revJustBeforeStartTime>..<revAtEndTime> -- <scope 路径>
```

- **格式**：标准 unified diff（`diff --git ...` / `---` / `+++` / `@@` hunks）。用 `git apply` 或 `patch -p1` 可应用。
- **范围**：按 `scope` 参数过滤（和 JSON 分母的文件范围一致）。
- **三个算法都生成**：

  | 算法 | patch 怎么得来 |
  |---|---|
  | A | 在工作副本或远端调 `git diff` / `svn diff`。 |
  | B | 把 `commit-patch-dir` 里的单次 revision diff 按拓扑顺序拼成一个累积 diff，revision 之间加 `# --- commit <rev> ---` 分隔符。 |
  | C | 从 v26.04 的内嵌 add/delete 记录合成：把 `[startTime, endTime]` 上累积的 add/delete 状态序列化为 unified diff。不访问 VCS。 |

- **用途**：和 JSON 配对——JSON 回答*"比例多少"*，patch 回答*"到底算了哪些行"*；两者一起就能审计、复现，不需要再访问仓库。
- **文件头**：patch 首部是一段注释，记录 `repoURL`、`repoBranch`、`startTime`、`endTime`、`algorithm`、`scope`，以及合成 id `aggregate:<start>..<end>`。

---

## 4. 场景矩阵 —— 12 个组合

轴：**VCS** = `git` | `svn` · **访问** = `local` | `remote` · **算法** = `A` | `B` | `C`。

每个组合都消费同一个 §2.2 里说的 `gen-code-desc-dir` 序列。

### 4.1 git × local

#### git · local · A（实时 blame）

- **前置**：本地有工作副本；`git` 在 PATH；`gen-code-desc-dir` 是 v26.03。
- **示例形状**：
  ```
  aggregateGenCodeDesc \
    --repo-url file:///srv/repos/foo \
    --repo-branch main \
    --start-time 2026-01-01T00:00:00Z --end-time 2026-04-01T00:00:00Z \
    --threshold 60 --algorithm A --scope A \
    --gen-code-desc-dir ./gcd/ \
    --output-dir ./out/
  ```
- **局限**：浅克隆会让 blame 失真（AC-005-4）。

#### git · local · B（离线 diff 重放）

- **前置**：工作副本带窗口内完整对象库；`commit-patch-dir` 已填充。
- **局限**：重命名链深时状态会炸（见 README 规模表）。

#### git · local · C（内嵌 blame，仅 v26.04）

- **前置**：**完全不需要 VCS**——`repo-url` / `repo-branch` 只用来做校验。`gen-code-desc-dir` 必须是 v26.04。
- **局限**：正确性完全依赖 codeAgent 写入时的 blame。

### 4.2 git × remote

#### git · remote · A

- **前置**：调用方须事先克隆远端（通过 `repo-path` 传路径），或 fork 自动克隆到工作目录。
- **局限**：浅克隆会让 blame 失真（AC-005-4）。

#### git · remote · B

- **前置**：调用方在 `commit-patch-dir` 里事先备好每个 revision 的 patch。工具本身不联网。
- **局限**：patch 的准备是调用方的责任。

#### git · remote · C

- **前置**：**完全不访问远端仓库**。传 `repo-url`/`repo-branch` 只是为了让结果 JSON 自我标识。
- **推荐**：内网隔离 / 大规模批处理场景。

### 4.3 svn × local

#### svn · local · A

- **前置**：工作副本；`svn` 在 PATH。
- **局限**：`svn blame` 对 merge 过来的行会不精确（见 README Git vs SVN 表）。

#### svn · local · B

- **前置**：工作副本；`commit-patch-dir` 通过 `svn diff -rN:M` 填充。
- **局限**：不支持跨文件 move 检测。

#### svn · local · C

- 和 `git · local · C` 一样——不访问 VCS。

### 4.4 svn × remote

#### svn · remote · A

- **前置**：调用方须事先签出 SVN 工作副本。`repo-branch` 要传 SVN 路径（如 `trunk`）。
- **局限**：工具本身不应直接联 SVN 服务器。

#### svn · remote · B

- **前置**：调用方在 `commit-patch-dir` 里事先备好每个 revision 的 patch。
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

对应 [README_UserStories.md](README_UserStories.md) US-006：

| 情况 | 参数 | 默认 | 退出码 |
|---|---|---|---|
| 窗口内某 revision 的 genCodeDesc 缺失 | `on-missing` | `zero`（A/B）/ `abort`（C） | 0 / 2 |
| 文件里 `REPOSITORY` 对不上 | — | 一律拒绝 | 2 |
| `revisionId` 重复 | `on-duplicate` | `reject` | 2 / 0 |
| `revisionTimestamp` 非单调（Alg C） | `on-clock-skew` | `abort` | 2 / 0 |
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
  --repo-url https://github.com/acme/myrepo \
  --repo-branch main \
  --start-time 2026-01-01T00:00:00Z \
  --end-time 2026-04-01T00:00:00Z \
  --threshold 60 \
  --algorithm A --scope A \
  --gen-code-desc-dir ./gcd-v26.03/ \
  --output-dir ./out/
```

### 算法 B：离线 Diff 重放（Git 或 SVN）

```text
aggregateGenCodeDesc \
  --repo-url https://svn.example.com/repo \
  --repo-branch /trunk \
  --start-time 2026-02-01T00:00:00Z \
  --end-time 2026-02-28T23:59:59Z \
  --threshold 60 \
  --algorithm B --scope A \
  --gen-code-desc-dir ./gcd-v26.03/ \
  --commit-patch-dir ./diffs/ \
  --output-dir ./out/
```

### 算法 C：内嵌 Blame（仅 v26.04，不需访问 VCS）

```text
aggregateGenCodeDesc \
  --repo-url https://github.com/acme/myrepo \
  --repo-branch main \
  --start-time 2026-01-01T00:00:00Z \
  --end-time 2026-04-01T00:00:00Z \
  --threshold 60 \
  --algorithm C --scope A \
  --gen-code-desc-dir ./gcd-v26.04/ \
  --output-dir ./out/
```

---

## 8. Fork 如何实现本手册

1. 从 `MyGenCodeDescBase` fork 出来，针对特定 CodeAgent & LLM 组合。
2. 用你选定的语言（Python / C++ / Rust）实现 `aggregateGenCodeDesc`。
3. 你的 fork 的 `README_ZH.md` 引用本手册，并说明：
   - 具体的 CLI 参数名（可以和上面的参数名不同）。
   - 12 个组合中哪些已支持（目标是全部 12 个；仅支持 Alg C 的 fork 可以跳过第 1、2、7、8 格）。
   - 每个组合的已知局限（如"Alg B 尚未实现"）。
   - `on-missing`、`on-duplicate`、`on-clock-skew` 的默认策略。
4. [README_UserStories.md](README_UserStories.md) 里的全部 57 条验收标准都是测试目标。
