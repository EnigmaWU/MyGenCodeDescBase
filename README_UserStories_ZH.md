# 用户故事 —— 每个 Fork 必须满足的 WHAT

本文档定义 MyGenCodeDescBase 每个 fork 都必须满足的**可测试验收标准**。
如果你的 fork 通过下面所有 AC 场景，那么你的 `aggregateGenCodeDesc` 实现就符合本 BASE 规范。

本文是 [README_UserStories.md](README_UserStories.md) 的中文镜像。为保持可追踪性，US/AC 编号、分类标签和 `GIVEN / WHEN / THEN` 关键字保持稳定。

格式遵循 [create-user-story](/.github/skills/create-user-story/SKILL.md) skill：
`AS A / I WANT / SO THAT` + `GIVEN / WHEN / THEN`，并带 AC 分类。

---

## 角色

| 角色 | 定义 | 来源 |
| --- | --- | --- |
| **代码库维护者** | 使用 `aggregateGenCodeDesc` 跨 revision 度量 AI 生成代码比例的人。 | [README_ZH.md](README_ZH.md) — “我们想要什么” |
| **工具开发者** | 构建、调试、维护 `aggregateGenCodeDesc` fork 的开发者。 | [README_ZH.md](README_ZH.md) — “我们想要什么” |

---

## US-000: 所有 Fork 的安全 Agent 沙箱

AS A 代码库维护者或工具开发者,
I WANT 每个 fork 都用可复现、隔离的容器环境承载所有代码代理工作,
SO THAT 实验、构建和 agent 操作不会污染或损坏宿主系统，并且所有贡献者都有一致、安全的环境。

### AC-000-1: [Typical] Fork 使用 Dev Container 初始化

```gherkin
Scenario: [Typical] Fork initializes with Dev Container
  GIVEN 开发者 fork 了 MyGenCodeDescBase
  WHEN 他们在 Docker Desktop 运行时用 VS Code 打开该 fork
  THEN VS Code 会提示在 Dev Container 中重新打开
  AND 容器可以成功构建
```

### AC-000-2: [Typical] 所有 agent / 代码工作都发生在容器内

```gherkin
Scenario: [Typical] All agent/code work happens inside container
  GIVEN 该 fork 已在 Dev Container 中打开
  WHEN 代码代理生成代码、运行测试或安装依赖
  THEN 所有动作都被隔离在容器中
  AND 除 workspace 挂载外，宿主系统不会被修改
```

### AC-000-3: [Safety] 非 root 用户且默认不挂载宿主 secrets

```gherkin
Scenario: [Safety] Container avoids privileged host access by default
  GIVEN 容器正在运行
  THEN 默认用户是非 root 用户
  AND 默认不挂载宿主 secrets
  AND 默认不挂载 Docker socket
```

### AC-000-4: [Typical] 安全工作流文档可执行

```gherkin
Scenario: [Typical] Contributor follows safe workflow docs
  GIVEN 新贡献者阅读仓库文档
  WHEN 他们按照 fork 和 Dev Container 指南操作
  THEN 除 Docker 和 VS Code Dev Containers 外，不需要额外准备即可复现安全 agent 工作流
```

---

## US-001: 核心度量计算

AS A 代码库维护者,
I WANT 计算时间窗口内被修改且在 endTime 仍存活的代码行的 AI 生成比例,
SO THAT 我可以量化代码库中有多少代码归因于 AI 生成。

### AC-001-1: [Typical] 加权模式计算 genRatio 之和

```gherkin
Scenario: [Typical] Weighted AI ratio for mixed-authorship lines
  GIVEN 10 行窗口内存活行的 genRatio 为 [100,100,100,100,100,80,80,80,30,0]
  WHEN aggregateGenCodeDesc 计算 Weighted 指标
  THEN 结果是 77.0%
```

### AC-001-2: [Typical] 纯 AI 模式只统计 genRatio==100

```gherkin
Scenario: [Typical] Fully AI ratio counts only fully-generated lines
  GIVEN 10 行窗口内存活行的 genRatio 为 [100,100,100,100,100,80,80,80,30,0]
  WHEN aggregateGenCodeDesc 计算 Fully AI 指标
  THEN 结果是 50.0%
```

### AC-001-3: [Typical] 主要 AI 模式统计 genRatio >= threshold

```gherkin
Scenario: [Typical] Mostly AI ratio with threshold 60
  GIVEN 10 行窗口内存活行的 genRatio 为 [100,100,100,100,100,80,80,80,30,0]
  AND threshold 为 60
  WHEN aggregateGenCodeDesc 计算 Mostly AI 指标
  THEN 结果是 80.0%
```

### AC-001-4: [Edge] 所有行都是人工编写

```gherkin
Scenario: [Edge] Zero AI ratio when all lines are human-written
  GIVEN 50 行窗口内存活行的 genRatio 全部为 0
  WHEN aggregateGenCodeDesc 计算三个指标
  THEN Weighted 是 0.0%
  AND Fully AI 是 0.0%
  AND Mostly AI 是 0.0%
```

### AC-001-5: [Edge] 所有行都是纯 AI 生成

```gherkin
Scenario: [Edge] 100% ratio when all lines are AI-generated
  GIVEN 50 行窗口内存活行的 genRatio 全部为 100
  WHEN aggregateGenCodeDesc 计算三个指标
  THEN Weighted 是 100.0%
  AND Fully AI 是 100.0%
  AND Mostly AI 是 100.0%
```

### AC-001-6: [Edge] 时间窗口内没有行发生变化

```gherkin
Scenario: [Edge] No in-window lines yields zero denominator
  GIVEN 仓库有 1000 行存活代码
  AND 没有任何行在 [startTime, endTime] 内被新增或修改
  WHEN aggregateGenCodeDesc 计算指标
  THEN 所有模式的结果都是 0.0%
  AND totalLines 是 0
```

### AC-001-7: [Typical] 稀疏 DETAIL 将省略行视为人工行

```gherkin
Scenario: [Typical] v26.03 DETAIL omits manual lines while SUMMARY counts them
  GIVEN SUMMARY.totalCodeLines 是 10
  AND SUMMARY.fullGeneratedCodeLines 是 5
  AND SUMMARY.partialGeneratedCodeLines 是 4
  AND DETAIL 只包含 9 行生成代码的条目
  WHEN aggregateGenCodeDesc 校验并计算聚合指标
  THEN 该记录被接受，因为 totalCodeLines >= fullGeneratedCodeLines + partialGeneratedCodeLines
  AND 被省略的窗口内存活行被视为 genRatio 0 且为人工/未归因
  AND Weighted 是 77.0%
  AND Fully AI 是 50.0%
  AND Mostly AI 在 threshold 为 60 时是 80.0%
```

### AC-001-8: [Typical] 聚合集是窗口 diff 与存活代码的交集

```gherkin
Scenario: [Typical] Metrics aggregate only the alive subset of the window diff
  GIVEN 从 fromCommit >= startTime 到 toCommit <= endTime 的提交范围累计 diff 中包含新增、修改和删除行
  AND endTime 的仓库快照只包含这些行中仍存活的当前版本
  WHEN aggregateGenCodeDesc 计算聚合指标
  THEN 分母是 (startTime..endTime diff) 与 (alive at endTime) 的交集行数
  AND diff 中已删除或已 revert 的行不计入分母
  AND 在 endTime 存活但最后修改时间早于 startTime 的行不计入分母
```

### AC-001-9: [Typical] outputDir 写出独立命名的聚合 genCodeDesc 产物

```gherkin
Scenario: [Typical] Aggregate JSON output uses the aggregated filename
  GIVEN aggregateGenCodeDesc 以 --outputDir ./out 调用
  WHEN 工具写出 aggregate JSON 结果
  THEN JSON 产物命名为 aggregatedGenCodeDescV26.03.json
  AND 该产物基于 genCodeDescProtoV26.03 JSON 形状
  AND 工具不会把聚合产物命名为 genCodeDescV26.03.json
```

---

## US-002: 文件级条件

AS A 代码库维护者,
I WANT 正确处理文件 rename、delete、copy 和 move,
SO THAT AI 比例不会被文件级操作扭曲。

### AC-002-1: [Typical] 纯 rename 保留行归因

```gherkin
Scenario: [Typical] File renamed without content change
  GIVEN 文件 "old.py" 有 100 行，并在 [startTime, endTime] 内的某个 commit 中被重命名为 "new.py"
  AND 内容没有变化
  WHEN aggregateGenCodeDesc 计算指标
  THEN 全部 100 行保留写入它们的原 commit 的 genRatio
  AND 没有行被重复计数
```

### AC-002-2: [Typical] Rename + 修改将变化行归因到新 commit

```gherkin
Scenario: [Typical] File renamed and partially modified
  GIVEN 文件 "old.py" 被重命名为 "new.py"
  AND 100 行中有 20 行在同一个 commit 中被修改
  WHEN aggregateGenCodeDesc 计算指标
  THEN 80 行未变化行保留原 genRatio
  AND 20 行修改行使用 rename commit 的 genCodeDesc 中的 genRatio
```

### AC-002-3: [Typical] 已删除文件对指标贡献为零

```gherkin
Scenario: [Typical] File deleted within the window
  GIVEN 文件 "removed.py" 在 startTime 前存在，且有 50 行 AI 生成代码（genRatio 100）
  AND 该文件在 [startTime, endTime] 内的 commit 中被删除
  WHEN aggregateGenCodeDesc 在 endTime 计算指标
  THEN "removed.py" 对指标贡献 0 行
  AND 它的任何行都不出现在存活快照中
```

### AC-002-4: [Edge] 文件复制到新路径

```gherkin
Scenario: [Edge] File duplicated via copy
  GIVEN 文件 "lib.py" 有 100 行
  AND "lib.py" 在 [startTime, endTime] 内的 commit 中被复制为 "lib_v2.py"
  WHEN aggregateGenCodeDesc 计算指标
  THEN "lib_v2.py" 中所有行都归因到 copy commit
  AND "lib.py" 中的行保留原归因
```

---

## US-003: 提交级条件

AS A 代码库维护者,
I WANT merge、squash、cherry-pick、revert、amend 和 rebase 操作都能产生正确归因,
SO THAT 无论分支工作流如何，指标都保持准确。

### AC-003-1: [Typical] Merge commit 追踪到原始 revision

```gherkin
Scenario: [Typical] Merge commit preserves original line origins
  GIVEN 分支 "feature" 中有 50 行来自 commit F1 的 AI 生成代码（genRatio 100）
  AND 分支 "main" 在 [startTime, endTime] 内通过 merge commit M1 合并了 "feature"
  WHEN aggregateGenCodeDesc 计算指标
  THEN 这 50 行归因到 commit F1（不是 M1）
  AND genRatio 来自 F1 的 genCodeDesc
```

### AC-003-2: [Typical] Squash merge 将所有行归因到 squash commit

```gherkin
Scenario: [Typical] Squash merge collapses attribution
  GIVEN 3 个 commit（C1、C2、C3）具有不同 genRatio，并被 squash-merge 为 commit S1
  AND S1 位于 [startTime, endTime] 内
  WHEN aggregateGenCodeDesc 计算指标
  THEN 所有存活行都归因到 S1
  AND genRatio 来自 S1 的 genCodeDesc
```

### AC-003-3: [Typical] Cherry-pick 形成独立归因

```gherkin
Scenario: [Typical] Cherry-picked commit has its own genCodeDesc
  GIVEN 分支 "feature" 上的 commit C1 有 30 行 AI 生成代码
  AND C1 被 cherry-pick 到 "main"，形成 [startTime, endTime] 内的 commit C2
  WHEN aggregateGenCodeDesc 为 "main" 计算指标
  THEN 行归因到 C2（不是 C1）
  AND genRatio 来自 C2 的 genCodeDesc
```

### AC-003-4: [Typical] Revert commit 移除 AI 归因

```gherkin
Scenario: [Typical] Reverted AI lines are gone
  GIVEN commit C1 新增了 20 行 AI 生成代码（genRatio 100）
  AND commit R1 在 [startTime, endTime] 内 revert 了 C1
  WHEN aggregateGenCodeDesc 在 endTime 计算指标
  THEN 这 20 行被 revert 的行对指标贡献为 0
```

### AC-003-5: [Edge] Amend / force-push 使旧 genCodeDesc 孤立

```gherkin
Scenario: [Edge] Amended commit replaces revisionId
  GIVEN commit C1（revisionId "aaa"）已有 genCodeDesc
  AND C1 通过 force-push 被 amend 成 C2（revisionId "bbb"）
  WHEN aggregateGenCodeDesc 查找 amended commit 的 genCodeDesc
  THEN C1 的 genCodeDesc（revisionId "aaa"）变为孤立记录
  AND 只使用 C2 的 genCodeDesc（revisionId "bbb"）
```

### AC-003-6: [Edge] Rebase 以新 revisionId 重放 commit

```gherkin
Scenario: [Edge] Rebased commits need regenerated genCodeDesc
  GIVEN commits C1、C2、C3 被 rebase 到新 base 上
  AND 新 commits C1'、C2'、C3' 有不同 revisionId
  WHEN aggregateGenCodeDesc 查找 C1'、C2'、C3' 的 genCodeDesc
  THEN 它使用匹配新 revisionId 的 genCodeDesc
  AND 旧 C1、C2、C3 revisionId 的 genCodeDesc 被忽略
```

---

## US-004: 行级条件

AS A 代码库维护者,
I WANT 正确处理行级所有权转移和边界情况,
SO THAT 每行 genRatio 准确反映当前作者归因。

### AC-004-1: [Typical] AI 行被人工编辑后转移所有权

```gherkin
Scenario: [Typical] Human edits an AI-generated line
  GIVEN "auth.py" 第 42 行在 commit C1 中由 AI 生成（genRatio 100）
  AND 人类在 [startTime, endTime] 内的 commit C2 中修改了第 42 行
  WHEN aggregateGenCodeDesc 计算指标
  THEN 第 42 行的 genRatio 来自 C2 的 genCodeDesc
  AND C1 的旧 genRatio 100 不再适用
```

### AC-004-2: [Typical] 人工行被 AI 重写后转移所有权

```gherkin
Scenario: [Typical] AI rewrites a human-written line
  GIVEN "utils.py" 第 10 行在 commit C1 中由人类编写（genRatio 0）
  AND AI 在 [startTime, endTime] 内的 commit C2 中重写第 10 行（genRatio 100）
  WHEN aggregateGenCodeDesc 计算指标
  THEN 第 10 行的 genRatio 是 100（来自 C2 的 genCodeDesc）
```

### AC-004-3: [Edge] 仅空白变化可能转移也可能不转移 blame

```gherkin
Scenario: [Edge] Whitespace-only change with default blame settings
  GIVEN "config.py" 第 5 行在 commit C1 中由 AI 生成（genRatio 100）
  AND commit C2 只修改了第 5 行的缩进
  WHEN aggregateGenCodeDesc 计算指标
  THEN 结果取决于 VCS blame 行为（git blame -w 策略）
  AND fork 明确记录自己的空白处理策略
```

### AC-004-4: [Edge] 行尾变化（CRLF↔LF）影响整文件

```gherkin
Scenario: [Edge] CRLF to LF conversion changes all lines
  GIVEN 文件 "data.txt" 有 500 行，genRatio 混合
  AND .gitattributes 变化在 commit C2 中把全部行尾从 CRLF 转为 LF
  WHEN aggregateGenCodeDesc 计算指标
  THEN 全部 500 行都归因到 commit C2
  AND genRatio 来自 C2 的 genCodeDesc
```

### AC-004-5: [Edge] 相同内容删除后再新增获得新归因

```gherkin
Scenario: [Edge] Deleted and re-added identical line has new origin
  GIVEN 文本为 "return 42" 的行在 commit C1 中被删除
  AND 相同文本 "return 42" 在 [startTime, endTime] 内的 commit C2 中被重新加入
  WHEN aggregateGenCodeDesc 计算指标
  THEN 该行归因到 C2（不是原 commit）
  AND genRatio 来自 C2 的 genCodeDesc
```

### AC-004-6: [Edge] 文件内移动的行遵循配置的所有权策略

```gherkin
Scenario: [Edge] Line moved from position 10 to position 50
  GIVEN 行 "x = compute()" 在 commit C1 中位于位置 10
  AND 该行在 [startTime, endTime] 内的 commit C2 中移动到位置 50
  WHEN aggregateGenCodeDesc 按 fork 配置的 move-detection 策略计算指标
  THEN fork 记录该 move 是按保留内容来源处理，还是按 delete+add 处理
  AND 如果按保留内容来源处理，该行保留 blame 报告的 origin，genRatio 来自该来源 revision
  AND 如果按 delete+add 处理，位置 50 的行归因到 C2，genRatio 来自 C2 的 genCodeDesc
  AND 启用 DEBUG 诊断时会记录所选策略
```

---

## US-005: 分支与历史条件

AS A 代码库维护者,
I WANT 指标准确处理长期分支、多次 merge 和时间窗口边界,
SO THAT 多分支工作流不会产生错误结果。

### AC-005-1: [Typical] 时间窗口外的行被排除

```gherkin
Scenario: [Typical] Line committed before startTime is excluded
  GIVEN "main.py" 第 1 行在 T0（startTime 前）提交
  AND 该行在 endTime 仍存活且 genRatio 为 100
  WHEN aggregateGenCodeDesc 为 [startTime, endTime] 计算指标
  THEN 第 1 行不参与指标计算
```

### AC-005-2: [Typical] 窗口内多次 merge 时每行只有一个 origin

```gherkin
Scenario: [Typical] Several branches merged within the window
  GIVEN 分支 A、B、C 都在 [startTime, endTime] 内 merge 到 main
  AND 每个分支贡献了不同的、genRatio 已知的行
  WHEN aggregateGenCodeDesc 计算指标
  THEN 每一行都有且只有一个 blame origin
  AND 没有行在多次 merge 中被重复计数
```

### AC-005-3: [Edge] 深度分叉的长期分支

```gherkin
Scenario: [Edge] Feature branch diverged 6 months from main
  GIVEN 分支 "feature" 从 "main" 分叉已经 6 个月
  AND "feature" 在 [startTime, endTime] 内 merge 到 "main"
  WHEN aggregateGenCodeDesc 计算指标
  THEN blame 会追踪每行在对应分支上的真实 origin commit
  AND 指标只包含 origin commit 位于 [startTime, endTime] 内的行
```

### AC-005-4: [Edge] 浅克隆限制 blame 准确性（AlgA/B）

```gherkin
Scenario: [Edge] Shallow clone causes blame to hit boundary
  GIVEN Git 仓库以 --depth 50 克隆
  AND 某些行的 origin 超过 50 个 commit 边界
  WHEN aggregateGenCodeDesc 使用 AlgA（实时 blame）
  THEN 工具在接受结果前检测 shallow clone 或 blame boundary
  AND 工具要么获取足够历史、要么以清晰错误 abort、要么把结果标记为降级（由 fork 定义策略）
  AND 不会静默地把边界 commit 当作真实行来源
```

### AC-005-5: [Edge] Submodule 有独立 genCodeDesc 链

```gherkin
Scenario: [Edge] Code in a Git submodule
  GIVEN 仓库在路径 "libs/crypto" 包含一个 submodule
  AND 该 submodule 有自己的 repoURL 和 genCodeDesc 链
  WHEN aggregateGenCodeDesc 为父仓库计算指标
  THEN submodule 的行不包含在父仓库指标中
  AND submodule 需要独立运行 aggregateGenCodeDesc
```

---

## US-006: 破坏性与边界条件

AS A 代码库维护者,
I WANT 检测并安全处理丢失、损坏或重复的 genCodeDesc 记录,
SO THAT 指标结果可靠，或明确标记为降级。

### AC-006-1: [Fault] 某个 revision 缺失 genCodeDesc

```gherkin
Scenario: [Fault] One revision's genCodeDesc is missing
  GIVEN commit C5 位于 [startTime, endTime] 内
  AND C5 没有对应 genCodeDesc 记录
  WHEN aggregateGenCodeDesc 处理该窗口
  THEN 对 AlgA/B，缺失记录按配置的 --onMissing 策略处理
  AND 默认 zero 策略将 C5 的行视为 genRatio 0，并输出包含 revisionId C5 的诊断
  AND 缺失记录不会被静默忽略
  AND AlgC 将缺失记录报告为链路中断错误
```

### AC-006-2: [Fault] genCodeDesc 损坏且 revisionId 错误

```gherkin
Scenario: [Fault] genCodeDesc has mismatched REPOSITORY fields
  GIVEN 某个 genCodeDesc 文件声称 revisionId 为 "abc123"
  AND REPOSITORY.repoURL 与目标仓库不匹配
  WHEN aggregateGenCodeDesc 校验该记录
  THEN 该记录因校验错误被拒绝
  AND 错误指出不匹配的字段
```

### AC-006-3: [Misuse] 同一 revision 有重复 genCodeDesc

```gherkin
Scenario: [Misuse] Two genCodeDesc records for the same revisionId
  GIVEN 两个 genCodeDesc 文件都具有 revisionId "abc123"
  AND 它们对同一批行包含不同 genRatio
  WHEN aggregateGenCodeDesc 处理窗口
  THEN 检测到重复记录
  AND 处理要么拒绝二者，要么使用最后写入记录（由 fork 定义策略）
  AND 重复记录作为 warning 记录日志
```

### AC-006-4: [Fault] 时钟偏移导致 AlgC 顺序错误

```gherkin
Scenario: [Fault] Non-monotonic timestamps in AlgC
  GIVEN commit C1 的 revisionTimestamp 是 2026-01-03T00:00:00Z
  AND commit C2 的 revisionTimestamp 是 2026-01-02T00:00:00Z（早于 C1，但实际在 C1 后提交）
  WHEN aggregateGenCodeDesc 使用按 revisionTimestamp 排序的 AlgC
  THEN 累积顺序错误
  AND fork 要么拒绝非单调序列，要么记录不准确性
```

### AC-006-5: [Misuse] genRatio 值超出有效范围

```gherkin
Scenario: [Misuse] genRatio is 150 in a genCodeDesc record
  GIVEN 某个 genCodeDesc DETAIL 条目的 genRatio 是 150
  WHEN aggregateGenCodeDesc 校验该条目
  THEN 该条目被拒绝并报错 "genRatio must be 0-100"
  AND 无效记录中的任何部分数据都不被使用
```

### AC-006-6: [Typical] 强制 CLI 参数名使用 lower camel case

```gherkin
Scenario: [Typical] Mandatory CLI arguments map to runtime and protocol identity
  GIVEN aggregateGenCodeDesc 以 --repoUrl、--repoBranch、--startTime、--endTime 和 --genCodeDescDir 调用
  WHEN 工具解析运行时参数
  THEN --repoUrl 被用作请求的仓库身份，并在聚合输出中回显为 REPOSITORY.repoURL
  AND --repoBranch、--startTime、--endTime 和 --genCodeDescDir 被一致地用于校验、过滤和输入发现
  AND 这五个 lower-camel-case CLI flag 是 BASE 强制参数解析契约
  AND 所有其他 CLI flag 都是可选或按需使用
```

---

## US-007: Git 与 SVN 差异

AS A 代码库维护者,
I WANT 指标能同时正确支持 Git 和 SVN 仓库,
SO THAT 面向 SVN 的 fork 不会被 Git 特定假设破坏。

### AC-007-1: [Typical] Git revision identity 是 SHA hash

```gherkin
Scenario: [Typical] Git revisionId format
  GIVEN 目标仓库使用 Git
  WHEN 创建 genCodeDesc 记录
  THEN revisionId 是 40 字符（SHA-1）或 64 字符（SHA-256）十六进制字符串
  AND aggregator 接受两种格式
```

### AC-007-2: [Typical] SVN revision identity 是顺序整数

```gherkin
Scenario: [Typical] SVN revisionId format
  GIVEN 目标仓库使用 SVN
  WHEN 创建 genCodeDesc 记录
  THEN revisionId 是正整数（例如 "4217"）
  AND aggregator 接受数字 revisionId
```

### AC-007-3: [Edge] SVN merge blame 不精确

```gherkin
Scenario: [Edge] SVN blame returns imprecise results after merge
  GIVEN SVN 仓库有一个通过 svn merge 合并的分支
  AND svn:mergeinfo 记录了该 merge
  WHEN aggregateGenCodeDesc 对 merged lines 使用 svn blame
  THEN fork 记录 SVN blame 可能把行归因到错误 revision
  AND 这作为已知局限写入日志
```

### AC-007-4: [Edge] Rebase/amend 仅 Git 有，SVN 忽略

```gherkin
Scenario: [Edge] SVN fork does not handle rebase
  GIVEN 目标仓库使用 SVN
  WHEN aggregator 检查 rebase 或 amend 条件
  THEN 这些条件被跳过（SVN 历史不可变）
  AND 不抛出错误
```

### AC-007-5: [Edge] SVN branch 是路径而不是 ref

```gherkin
Scenario: [Edge] SVN repoBranch maps to a path
  GIVEN SVN 仓库的分支位于 /branches/feature-x
  WHEN genCodeDesc 记录引用 repoBranch
  THEN repoBranch 包含 SVN 路径（例如 "/branches/feature-x"）
  AND aggregator 正确规范化 SVN branch path
```

---

## US-008: 规模与性能

AS A 代码库维护者,
I WANT `aggregateGenCodeDesc` 能处理参考规模（1K commits × 100 files × 10K lines）,
SO THAT 工具可以用于生产规模仓库。

### AC-008-1: [Performance] AlgA 在可接受时间内完成

```gherkin
Scenario: [Performance] AlgA at reference scale
  GIVEN [startTime, endTime] 内有 1,000 个 commit
  AND 每个 commit 有 100 个文件，每个文件 10,000 行
  AND endTime 约有 10,000 个不同文件
  WHEN aggregateGenCodeDesc 运行 AlgA（实时 blame）
  THEN 工具可以完成（正确性优先于速度，fork 记录实际耗时）
  AND 顺序处理时峰值内存低于 1 GB
```

### AC-008-2: [Performance] AlgA 处理真实 60 天多分支项目

```gherkin
Scenario: [Performance] AlgA on one repo with 5 developers, 20 branches, and 60 days of activity
  GIVEN 一个 Git 仓库有 5 名活跃开发者
  AND 仓库有 20 个活跃分支
  AND 最近 60 天内，仓库所有分支合计每天约有 10 个 commit
  AND aggregateGenCodeDesc 针对 repoBranch "main" 和这 60 天的 [startTime, endTime] 窗口运行 AlgA
  WHEN 工具计算聚合指标
  THEN fromCommit 和 toCommit 只从 repoBranch "main" 可达的 commit 中选择
  AND 到 endTime 仍未 merge 到 "main" 的分支专属 commit 不进入分母
  AND merge、squash merge、cherry-pick、revert 的行遵循已有 AC 的所有权规则
  AND 每条被统计的存活行都按 blame 来源 revision 和来源坐标 join 到 genCodeDescV26.03
  AND 运行结束时记录实际耗时、内存、blame command 数量，以及任何降级结果 warning
```

### AC-008-3: [Performance] AlgC 处理 1 GB genCodeDescV26.04 数据

```gherkin
Scenario: [Performance] AlgC streaming at reference scale
  GIVEN genCodeDescV26.04 文件总计约 1 GB
  AND 这些文件包含足够多的 DETAIL 条目，需要 streaming 而不是一次性全部载入内存
  WHEN aggregateGenCodeDesc 运行 AlgC（内嵌 blame）
  THEN 文件按时间戳顺序流式处理（不一次性全部加载）
  AND 峰值内存由存活行集合限制，fork 记录实际内存用量
```

### AC-008-4: [Edge] 时间窗口内零 commit

```gherkin
Scenario: [Edge] Empty time window
  GIVEN [startTime, endTime] 内包含 0 个 commit
  WHEN aggregateGenCodeDesc 运行
  THEN 所有模式结果都是 0.0%
  AND totalLines 是 0
  AND 工具无错误完成
```

### AC-008-5: [Robust] 工具从流处理中途 I/O 失败恢复

```gherkin
Scenario: [Robust] Disk read fails on one genCodeDesc file
  GIVEN 要处理 1,000 个 genCodeDesc 文件
  AND 第 500 个文件不可读（磁盘错误）
  WHEN aggregateGenCodeDesc 遇到 I/O 错误
  THEN 错误报告包含文件路径和 revisionId
  AND 工具要么以清晰错误 abort，要么以降级结果继续（由 fork 定义策略）
  AND 不写入部分/损坏输出
```

---

## US-009: 算法特定行为

AS A 代码库维护者,
I WANT 每个算法（A、B、C）都正确处理自身特有的行来源发现逻辑,
SO THAT 无论 fork 实现哪个算法，指标结果都准确。

### AlgA —— 实时 Blame

#### AC-009-1: [Typical] Blame 追踪整文件 rename

```gherkin
Scenario: [Typical] AlgA follows renamed file via git blame
  GIVEN 文件 "old_name.py" 在 commit C1 中被重命名为 "new_name.py"
  AND "new_name.py" 第 10 行最初写于 commit C0
  WHEN aggregateGenCodeDesc 对 "new_name.py" 运行 AlgA 的 git blame
  THEN blame 报告第 10 行的 origin 是 commit C0（不是 C1），且 origin file 是 "old_name.py"
  AND genRatio 查找使用 C0 的 genCodeDesc 中的来源坐标
```

#### AC-009-2: [Edge] 通过 -C -C 检测跨文件移动

```gherkin
Scenario: [Edge] AlgA detects code moved from another file
  GIVEN 30 行从 "source.py" 剪切并粘贴到 commit C1 中的 "target.py"
  AND git blame -C -C 已启用
  WHEN aggregateGenCodeDesc 在 "target.py" 上运行 AlgA
  THEN 这 30 行归因到它们在 "source.py" 中被写入的原 commit
  AND fork 记录 -C -C 行为及其性能成本
```

#### AC-009-3: [Fault] AlgA 分析时 VCS 服务器不可达

```gherkin
Scenario: [Fault] Remote VCS is down when AlgA runs blame
  GIVEN 目标仓库托管在远端 Git server
  AND server 在分析时不可达
  WHEN aggregateGenCodeDesc 运行 AlgA 并尝试 git blame
  THEN 工具报告包含 server URL 的清晰连接错误
  AND 不写入部分指标结果
  AND 工具建议重试或使用 AlgC（无需 VCS 访问）
```

#### AC-009-4: [Typical] AlgA 按 blame 来源坐标 join genCodeDesc

```gherkin
Scenario: [Typical] AlgA uses origin coordinates for v26.03 lookup after rename and line-number shift
  GIVEN commit C1 在 [startTime, endTime] 内创建 "src/a.py" 第 2 行，并且 genCodeDescV26.03(C1) 中该行 genRatio 为 100
  AND commit C2 在 endTime 之前把文件改名为 "src/math_utils.py" 并插入 header，使同一条存活文本现在位于当前第 4 行
  WHEN aggregateGenCodeDesc 在 endTime 运行 AlgA
  THEN blame 将当前 "src/math_utils.py" 第 4 行解析到来源 revision C1、来源文件 "src/a.py"、来源第 2 行
  AND genRatio 查找使用 genCodeDescV26.03(C1) DETAIL 中的 "src/a.py" 第 2 行
  AND 实现不会去 C1 的 genCodeDesc 中查 "src/math_utils.py" 第 4 行
  AND 这条活行以 genRatio 100 计入
```

#### AC-009-5: [Typical] AlgA 处理窗口内只有一个 commit

```gherkin
Scenario: [Typical] AlgA computes metrics when exactly one commit is inside the time window
  GIVEN [startTime, endTime] 内唯一的 commit 是 C1
  AND C1 新增 3 行、修改 2 行已有代码、删除 1 行，并让 10 行窗口前代码保持不变
  AND 在 endTime，3 行新增代码和 2 行修改后的当前版本仍然存活
  WHEN aggregateGenCodeDesc 运行 AlgA
  THEN fromCommit 是 C1，toCommit 也是 C1
  AND blame 只把来源 revision 为 C1 的 5 条存活行计入指标分母
  AND 被删除的行和 10 行未触碰的窗口前代码不进入分母
  AND 这 5 条来源为 C1 的存活行的 genRatio 从 genCodeDescV26.03(C1) 查找
```

#### AC-009-6: [Typical] AlgA 在隔离的 endTime 快照上运行 blame

```gherkin
Scenario: [Typical] AlgA resolves endTime snapshot before running blame
  GIVEN repoBranch 上 commit C1 是 timestamp <= endTime 的最后一个 revision
  AND commit C2 位于 endTime 之后
  AND 本地 repoPath 当前 checkout 在 C2，且存在未提交修改
  WHEN aggregateGenCodeDesc 针对 [startTime, endTime] 运行 AlgA
  THEN 工具将 toCommit 解析为 C1
  AND blame 在 C1 的干净隔离 checkout 或 worktree 上运行
  AND blame 不在当前 HEAD C2 上运行
  AND 用户 working tree 和未提交文件不会被修改
```

### AlgB —— Diff 重放

#### AC-009-7: [Typical] 按拓扑顺序重放多文件 diff

```gherkin
Scenario: [Typical] AlgB replays multi-file, multi-hunk diffs in correct commit order
  GIVEN [startTime, endTime] 内有 5 个 commit（C1→C2→C3→C4→C5）
  AND 至少一个 commit patch 同时包含 "main.py" 和 "helper.py" 的 diff section
  AND 至少一个文件 diff section 包含多个 hunk（`@@ ... @@`）
  WHEN aggregateGenCodeDesc 运行 AlgB
  THEN diff 按拓扑顺序重放（C1、C2、C3、C4、C5）
  AND 每个 commit patch 中的每个文件 diff section 和每个 hunk 都被重放
  AND 目录顺序、文件名排序、patch 文件修改时间不控制重放顺序
  AND 最终 line-to-origin 映射匹配 endTime 的活文件状态
```

#### AC-009-8: [Edge] 跨重命名链追踪行位置

```gherkin
Scenario: [Edge] AlgB tracks lines across rename chain
  GIVEN 文件 "v1.py" 在 commit C2 中被重命名为 "v2.py"
  AND "v2.py" 在 commit C4 中被重命名为 "v3.py"
  AND 第 15 行在两次 rename 中都未被修改
  WHEN aggregateGenCodeDesc 运行 AlgB 重放 diff C1→C2→C3→C4→C5
  THEN endTime 时 "v3.py" 的第 15 行归因到最初写入它的 commit
  AND rename graph 正确映射 v1.py → v2.py → v3.py
```

#### AC-009-9: [Fault] 链中缺失一个 diff

```gherkin
Scenario: [Fault] AlgB cannot retrieve diff for commit C3
  GIVEN [startTime, endTime] 内有 5 个 commit（C1→C2→C3→C4→C5）
  AND commit C3 的 diff 不可用（网络错误或 ref 被删除）
  WHEN aggregateGenCodeDesc 运行 AlgB
  THEN 工具报告缺失 diff 及 commit C3 的 revisionId
  AND 工具要么以 chain-break 错误 abort，要么跳过 C3 并标记精度降级（由 fork 定义策略）
  AND 不产生静默错误结果
```

### AlgC —— 内嵌 Blame（仅 v26.04）

#### AC-009-10: [Typical] Add/delete 操作构建正确存活集合

```gherkin
Scenario: [Typical] AlgC accumulates surviving lines from add/delete entries
  GIVEN 3 个 genCodeDesc 文件（对应 C1、C2、C3）按时间戳顺序处理
  AND C1 向 "app.py" 新增 100 行
  AND C2 从 "app.py" 删除 20 行并新增 30 行
  AND C3 从 "app.py" 删除 10 行
  WHEN aggregateGenCodeDesc 运行 AlgC
  THEN "app.py" 的存活集合正好有 100 行（100 - 20 + 30 - 10）
  AND 每个存活行的 genRatio 匹配其 add entry 的 genCodeDesc
```

#### AC-009-11: [Edge] 同一 file+line 有重复 add entry

```gherkin
Scenario: [Edge] AlgC encounters duplicate add for the same line position
  GIVEN C1 的 genCodeDesc 对 "app.py" 第 42 行有一个 add entry
  AND C2 的 genCodeDesc 对 "app.py" 第 42 行也有 add entry，且之前没有 delete
  WHEN aggregateGenCodeDesc 运行 AlgC
  THEN 检测到重复
  AND 工具要么以 warning 拒绝第二个 entry，要么用后一个覆盖（由 fork 定义策略）
  AND 不一致情况被记录日志
```

#### AC-009-12: [Fault] SUMMARY lineCount 与实际 DETAIL entries 不一致

```gherkin
Scenario: [Fault] AlgC detects mismatch between SUMMARY and DETAIL
  GIVEN 某个 genCodeDesc 文件的 SUMMARY 声称 lineCount 是 500
  AND 实际 DETAIL section 只包含 487 个 entry
  WHEN aggregateGenCodeDesc 在 AlgC 处理期间校验该文件
  THEN 检测到不一致（expected 500, found 487）
  AND 工具报告该文件 revisionId 的差异
  AND 处理要么 abort，要么带 warning 继续（由 fork 定义策略）
```

---

## US-010: 诊断与日志

AS A 工具开发者,
I WANT `aggregateGenCodeDesc` 支持 `--logLevel` 和 `--timing`,
SO THAT 我可以检查运行时行为、诊断问题，并理解各阶段运行耗时。

### AC-010-1: [Typical] 默认日志级别是 INFO，包含加载/处理/汇总阶段

```gherkin
Scenario: [Typical] Running without --logLevel shows INFO progress
  GIVEN aggregateGenCodeDesc 调用时没有传 --logLevel
  AND 工具处理覆盖 5 个文件的 10 个 genCodeDesc 文件
  WHEN 工具运行
  THEN 日志输出显示三个阶段及进度：
    - LOAD phase：每个 genCodeDesc 文件被加载（"LOAD [3/10] revisionId=abc123 entries=1500"）
    - PROCESS phase：每个文件的逐行来源解析和逐行状态
      （"PROCESS file=auth.py line=42 state=ADDED origin=abc123 genRatio=100"）
      （"PROCESS file=auth.py line=43 state=DELETED origin=def456"）
    - SUMMARY phase：每个文件和整体的最终指标
      （"SUMMARY file=auth.py totalLines=100 weighted=58.0% fullyAI=50.0% mostlyAI=80.0%"）
      （"SUMMARY aggregate totalLines=500 weighted=65.0% fullyAI=42.0% mostlyAI=71.0%"）
  AND 日志输出不包含 DEBUG 级消息
```

### AC-010-2: [Typical] DEBUG 级别显示逐文件和逐行细节

```gherkin
Scenario: [Typical] --logLevel DEBUG reveals detailed processing steps
  GIVEN aggregateGenCodeDesc 以 --logLevel DEBUG 调用
  WHEN 工具处理 commit C1 的 genCodeDesc 文件
  THEN 日志输出包含：
    - 正在运行哪个算法（AlgA/B/C）
    - 每个正在处理的文件及行数
    - 每个已加载 genCodeDesc 文件的 revisionId 和 entry count
    - 行来源解析决策（blame result / diff replay step / add-delete operation）
  AND 所有 DEBUG 消息都包含 timestamp
```

### AC-010-3: [Typical] WARN 级别记录非致命异常

```gherkin
Scenario: [Typical] Warnings logged for degraded but non-fatal conditions
  GIVEN aggregateGenCodeDesc 遇到 SUMMARY/DETAIL 不一致的 genCodeDesc
  AND fork 策略是继续处理并发出 warning
  WHEN 工具处理该文件
  THEN 发出 WARN 级消息，包含：
    - 问题文件的 revisionId
    - 异常性质（例如 "expected 500 entries, found 487"）
  AND 处理继续
```

### AC-010-4: [Typical] ERROR 级别记录致命失败

```gherkin
Scenario: [Typical] Errors logged when processing must abort
  GIVEN aggregateGenCodeDesc 遇到不可读的 genCodeDesc 文件
  AND fork 策略是 I/O 错误时 abort
  WHEN 工具处理该文件
  THEN 发出 ERROR 级消息，包含：
    - 文件路径和 revisionId
    - OS 级错误描述
  AND 工具以非零 exit code 退出
  AND 不写入部分输出
```

### AC-010-5: [Edge] --logLevel ERROR 抑制 INFO 和 WARN

```gherkin
Scenario: [Edge] Minimal output with --logLevel ERROR
  GIVEN aggregateGenCodeDesc 以 --logLevel ERROR 调用
  AND 处理过程中遇到 WARN 级异常但没有 error
  WHEN 工具成功完成
  THEN 日志输出不包含任何消息（无 INFO、无 WARN）
  AND 只有最终指标结果写到 stdout
```

### AC-010-6: [Observability] 结构化日志格式便于机器解析

```gherkin
Scenario: [Observability] Log output is structured and parseable
  GIVEN aggregateGenCodeDesc 以 --logLevel DEBUG 调用
  WHEN 产生日志消息
  THEN 每行日志包含：timestamp、level、component、message
  AND 格式一致（例如 "2026-04-14T10:30:00Z [DEBUG] [AlgC] Loading genCodeDesc revisionId=abc123 entries=1000"）
  AND 日志输出到 stderr（不与 stdout 上的指标结果混合）
```

### AC-010-7: [Testability] 测试中可不经 CLI 配置日志级别

```gherkin
Scenario: [Testability] Unit tests can set log level programmatically
  GIVEN 单元测试导入 aggregateGenCodeDesc library
  WHEN 测试通过 API 设置日志级别为 DEBUG（不是 CLI flag）
  THEN DEBUG 级消息被测试输出捕获
  AND 测试用例之间没有全局状态泄漏
```

### AC-010-8: [Observability] TIMING 以秒报告各阶段耗时

```gherkin
Scenario: [Observability] Timing output shows clone, blame, and aggregate cost
  GIVEN aggregateGenCodeDesc 以 --timing summary 调用
  AND AlgA 在分析前自动 clone 一个远端 Git 仓库
  WHEN 工具成功完成
  THEN 聚合 JSON 包含顶层 TIMING object
  AND TIMING 包含 totalSeconds、cloneRepoSeconds、checkoutSeconds、loadGenCodeDescSeconds、blameSeconds、aggregateSeconds 和 writeOutputSeconds 的非负秒数
  AND totalSeconds 大于或等于每个单独测量阶段
  AND 所选算法或访问模式未使用的阶段要么为 0，要么列在 TIMING.notRun 中
  AND timing 输出不会改变 SUMMARY 计数、DETAIL 条目、AGGREGATE 指标或 commitStart2EndTime.patch 内容
```

| US | 标题 | AC 数量 | 覆盖分类 |
| --- | --- | --- | --- |
| US-001 | 核心度量计算 | 9 | Typical, Edge |
| US-002 | 文件级条件 | 4 | Typical, Edge |
| US-003 | 提交级条件 | 6 | Typical, Edge |
| US-004 | 行级条件 | 6 | Typical, Edge |
| US-005 | 分支与历史条件 | 5 | Typical, Edge |
| US-006 | 破坏性与边界条件 | 6 | Fault, Misuse, Typical |
| US-007 | Git 与 SVN 差异 | 5 | Typical, Edge |
| US-008 | 规模与性能 | 5 | Performance, Edge, Robust |
| US-009 | 算法特定行为 | 12 | Typical, Edge, Fault |
| US-010 | 诊断与日志 | 8 | Typical, Edge, Observability, Testability |
| **总计** | | **66 AC** | |

---

## Fork 如何使用本文档

1. 针对特定 CodeAgent & LLM 组合 fork 本仓库。
2. 上面的每个 AC 都通过 `test-driven-development` skill 转成一个测试用例。
3. **RED** — 根据 GIVEN/WHEN/THEN 场景写一个失败测试。
4. **GREEN** — 实现最小代码让测试通过。
5. **REFACTOR** — 清理实现。
6. 当全部 66 个 AC 都通过时，说明你的实现符合 BASE 规范。

> **不是每个 AC 都适用于每个 fork。** Git-only 条件（rebase、amend、shallow clone）
> 可以被 SVN fork 跳过。AlgC-specific AC 可以被只实现 AlgA 的 fork 跳过。
> 请记录哪些 AC 被跳过，以及为什么。

---

## 附录：VCS 与算法覆盖矩阵

### 覆盖范围

这些 AC 是 **Git-first** 的。Git 是现代标准，也是主要目标。
SVN 是 legacy；在协议能力允许范围内支持，但存在已知局限。

### 每个 AC 的 VCS 覆盖

| AC | Git | SVN | 说明 |
| --- | --- | --- | --- |
| **US-001（核心度量）** | | | |
| AC-001-1 ~ AC-001-8 | ✅ | ✅ | VCS 无关：指标数学、稀疏 DETAIL 语义，以及 `(window diff) ∩ (alive at endTime)` 聚合集 |
| **US-002（文件级）** | | | |
| AC-002-1 ~ AC-002-2（rename） | ✅ | ✅ | Git：启发式 `-M`。SVN：显式 `svn move`，更可靠 |
| AC-002-3（delete） | ✅ | ✅ | 行为相同 |
| AC-002-4（copy） | ✅ | ⚠️ | SVN `svn copy` 语义不同于 Git，fork 必须适配 |
| **US-003（提交级）** | | | |
| AC-003-1（merge） | ✅ | ⚠️ | SVN 没有真正的 merge commit，基于 `svn:mergeinfo`，blame 可能不精确 |
| AC-003-2（squash） | ✅ | ✅ | SVN squash 是手工操作，但概念相同 |
| AC-003-3（cherry-pick） | ✅ | ⚠️ | SVN `svn merge -c`，mergeinfo 可能干扰 blame |
| AC-003-4（revert） | ✅ | ✅ | 行为相同 |
| AC-003-5（amend/force-push） | ✅ | ❌ N/A | SVN 历史不可变，跳过 |
| AC-003-6（rebase） | ✅ | ❌ N/A | SVN 不存在，跳过 |
| **US-004（行级）** | | | |
| AC-004-1 ~ AC-004-6 | ✅ | ✅ | VCS 无关：逐行所有权逻辑 |
| AC-004-3（whitespace） | ✅ | ⚠️ | `svn blame` 没有 `-w` 等价项，空白总是显著 |
| **US-005（分支/历史）** | | | |
| AC-005-1 ~ AC-005-2 | ✅ | ✅ | 标准 blame 行为 |
| AC-005-3（long-lived branch） | ✅ | ⚠️ | SVN 对长期分叉分支的 blame 可能受 mergeinfo 影响而不精确 |
| AC-005-4（shallow clone） | ✅ | ❌ N/A | SVN 没有 shallow clone，总是通过 server 获取完整历史 |
| AC-005-5（submodule） | ✅ | ⚠️ | SVN 使用 `svn:externals`，语义不同，路径解析更复杂 |
| **US-006（破坏/边界）** | | | |
| AC-006-1 ~ AC-006-3 | ✅ | ✅ | VCS 无关：genCodeDesc 校验 |
| AC-006-4（clock skew） | ✅ | ❌ N/A | SVN timestamp 由 server 分配且单调，无 skew 风险 |
| AC-006-5（invalid genRatio） | ✅ | ✅ | VCS 无关：输入校验 |
| AC-006-6（CLI arguments） | ✅ | ✅ | VCS 无关：CLI 解析 |
| **US-007（Git vs SVN）** | | | |
| AC-007-1 ~ AC-007-5 | ✅ | ✅ | 专门记录差异 |
| **US-008（规模/性能）** | | | |
| AC-008-1 ~ AC-008-5 | ✅ | ✅ | 同一规模模型适用于二者 |

**图例：** ✅ = 完全适用，⚠️ = 适用但有已知局限，❌ N/A = 不适用（跳过）

### 每个 AC 的算法覆盖

| AC | AlgA（实时 blame） | AlgB（diff 重放） | AlgC（内嵌 blame） |
| --- | --- | --- | --- |
| **US-001 ~ US-004** | ✅ | ✅ | ✅ |
| AC-005-4（shallow clone） | ✅ detect/fetch/abort/degrade | ✅ diffs unavailable beyond depth | ❌ N/A（自给自足） |
| AC-006-1（missing genCodeDesc） | ✅ --onMissing policy | ✅ --onMissing policy | ⚠️ chain break |
| AC-006-4（clock skew） | ❌ N/A（order-independent） | ❌ N/A（topological order） | ✅ sorts by timestamp |
| AC-008-1（AlgA perf） | ✅ | ❌ N/A | ❌ N/A |
| AC-008-2（realistic AlgA workload） | ✅ | ❌ N/A | ❌ N/A |
| AC-008-3（AlgC perf） | ❌ N/A | ❌ N/A | ✅ |
| **AC-009-1（rename blame）** | ✅ | ❌ N/A | ❌ N/A |
| **AC-009-2（cross-file -C -C）** | ✅ | ❌ N/A | ❌ N/A |
| **AC-009-3（VCS unreachable）** | ✅ | ❌ N/A | ❌ N/A |
| **AC-009-4（origin-coordinate lookup）** | ✅ | ❌ N/A | ❌ N/A |
| **AC-009-5（single in-window commit）** | ✅ | ❌ N/A | ❌ N/A |
| **AC-009-6（isolated endTime snapshot）** | ✅ | ❌ N/A | ❌ N/A |
| **AC-009-7（topological multi-file replay）** | ❌ N/A | ✅ | ❌ N/A |
| **AC-009-8（chained renames）** | ❌ N/A | ✅ | ❌ N/A |
| **AC-009-9（missing diff）** | ❌ N/A | ✅ | ❌ N/A |
| **AC-009-10（surviving set）** | ❌ N/A | ❌ N/A | ✅ |
| **AC-009-11（duplicate add）** | ❌ N/A | ❌ N/A | ✅ |
| **AC-009-12（SUMMARY mismatch）** | ❌ N/A | ❌ N/A | ✅ |

> **对于 SVN forks：** 跳过所有 ❌ N/A 行。把 ⚠️ 行作为已知局限记录到你的 fork README。
>
> **对于单算法 forks：** 只实现所选算法对应的 AC。
> 只实现 AlgA 的 fork 可以跳过 AlgC-specific AC（AC-008-3、AC-006-4）。
