# 测试指南 —— `aggregateGenCodeDesc`

本指南定义 MyGenCodeDescBase 的每个 fork 都应遵循的测试模型。
每个 fork 可以选择 Python、C++ 或 Rust，但测试意图保持一致：

> UserStory + UserGuide -> AcceptanceCriteria -> TestCase -> Verified implementation。

这是本仓库的 CaTDD（Comment-alive Test-Driven Development，注释驱动的测试先行开发）流程。在本指南中，UnitTesting、SysTesting 和 UserTesting 是同一条 CaTDD 链路上的三个验证层，而不是三种互不相关的测试风格。

测试套件分为三层：

| 层级 | 目的 | 主要受众 | 典型位置 |
| --- | --- | --- | --- |
| UnitTesting | 隔离验证小而确定的逻辑。 | 工具开发者 | `tests/unit/` |
| SysTesting | 使用真实仓库和协议文件验证完整 CLI/工具行为。 | 工具开发者 / 维护者 | `tests/system/` |
| UserTesting | 验证用户指南和用户故事中的面向用户工作流。 | 代码库维护者 / 审阅者 | `tests/user/` 或 `tests/demo/` |

所有测试都应在 [README_ForkWorkflow_ZH.md](README_ForkWorkflow_ZH.md) 描述的 Dev Container 内运行，这样代码代理可以在安全、可复现的环境中执行测试。

## 0. CaTDD 参考

CaTDD 把意图、验证设计和可执行检查连接起来：

| CaTDD 产物 | 含义 | 主要测试层 |
| --- | --- | --- |
| UserStory | 定义用户价值和必需行为。 | UserTesting、SysTesting |
| UserGuide | 定义文档化的命令流程和面向用户的契约。 | UserTesting、SysTesting |
| AcceptanceCriteria | 把故事转成可验证的 Given/When/Then 场景。 | UnitTesting、SysTesting、UserTesting |
| TestCase | 把每个 AC 转成一个或多个可执行或手动检查。 | UnitTesting、SysTesting、UserTesting |
| Verified implementation | 确认实现满足 BASE 契约。 | 所有层 |

每个测试都应保留可见的追踪链路：

```text
US -> AC -> TC -> implementation/test assertion -> result
```

三层测试的使用方式如下：

| 层级 | CaTDD 角色 |
| --- | --- |
| UnitTesting | 证明某个 AC 背后的单条规则、解析器行为、公式或策略分支。 |
| SysTesting | 通过真实 CLI 和真实产物证明一个完整 AC 或用户故事路径。 |
| UserTesting | 证明维护者可以按照 UserGuide 操作并识别预期结果。 |

编写或审阅测试时，应能清楚看出它保护的是哪个 US 和 AC。缺少这条链接时，该测试还没有对齐 CaTDD。

### 0.1 CaTDD 优先级分类

CaTDD 还会为每个测试用例标注优先级和类别。测试层说明测试在哪里运行；CaTDD 类别说明测试保护的是哪一种风险或行为。

| 优先级 | 类别 | 保护内容 | 常见层级 |
| --- | --- | --- | --- |
| P1 Functional | Typical | 用户期望正常工作的主路径行为。 | UnitTesting、SysTesting、UserTesting |
| P1 Functional | Boundary | 边界值、限制、空输入、阈值和零分母。 | UnitTesting、SysTesting |
| P1 Functional | Misuse | 无效输入、不支持的参数组合和用户误用。 | UnitTesting、SysTesting、UserTesting |
| P1 Functional | Fault | 损坏文件、缺失版本、重复记录和外部命令失败。 | UnitTesting、SysTesting |
| P2 Design | State | 有状态转换，例如 add/delete 回放、rename 跟踪和存活行状态。 | UnitTesting、SysTesting |
| P2 Design | Capability | 可选模式和受支持的功能组合，例如 Algorithm A/B/C 与 Scope A/B/C/D。 | SysTesting |
| P2 Design | Concurrency | 并行 blame、分片回放、缓存行为和竞争敏感处理。 | UnitTesting、SysTesting |
| P3 Quality | Performance | 大窗口、大文件、流式处理和有界内存处理。 | SysTesting |
| P3 Quality | Robust | 恢复行为、降级模式、确定性诊断和可重复性。 | SysTesting、UserTesting |
| P3 Quality | Compatibility | Git vs SVN、通过 Dev Container 屏蔽主机 OS 差异，以及协议版本兼容性。 | SysTesting、UserTesting |
| P3 Quality | Configuration | CLI 选项和策略设置，例如 `onMissing`、`onDuplicate`、`onClockSkew`。 | UnitTesting、SysTesting |
| P4 Addons | Demo/Example | 文档示例、维护者演示和引导式验收 walkthrough。 | UserTesting |

每个 CaTDD 测试用例都应声明两个字段：

```text
@[Priority]: P1 Functional
@[Category]: Typical
```

这个类别独立于 UnitTesting/SysTesting/UserTesting。例如，一个 `Boundary` 用例既可以是度量公式的 UnitTesting，也可以是 CLI 输出的 SysTesting。

---

## 1. UnitTesting

UnitTesting 验证最小的有意义实现片段，不依赖活的 Git/SVN 仓库、网络访问、外部服务或大型 fixture 目录。

### 1.1 CaTDD 角色

在 CaTDD 中，UnitTesting 是证明某条 AC 规则正确工作的最低成本方式。一个单元测试通常映射到一个公式、解析规则、校验规则、策略分支或辅助函数行为。

好的 UnitTesting 应回答：

- 这个小行为支持哪个 US 和 AC？
- 被验证的精确规则是什么？
- 哪些输入、行为和预期输出能让这条规则可观察？
- 这个测试是否足够独立，能在接近根因的地方失败？

推荐的 CaTDD 测试用例形状：

```text
[@AC-001-1,US-001]
TC-Unit-001:
  @[Name]: verifyWeightedRatio_byMixedGenRatio_expectSeventySevenPercent
  @[Priority]: P1 Functional
  @[Category]: Typical
  @[Purpose]: Protect the weighted metric formula.
  @[Brief]: Given 10 live lines with mixed genRatio values, compute weighted ratio.
  @[Expect]: Result is 77.0%.
```

### 1.2 UnitTesting 必须覆盖什么

每个 fork 至少应做以下单元测试：

| 领域 | 必需覆盖 |
| --- | --- |
| 度量公式 | Weighted、Fully AI、Mostly AI 计算，包括零分母。 |
| 阈值处理 | 边界值 `0`、`1`、`60`、`99`、`100`，以及 `0..100` 外的非法值。 |
| 协议解析 | v26.03 和 v26.04 的合法记录、缺失字段、错误类型、畸形 JSON。 |
| 协议校验 | 混合协议版本、重复 `revisionId`、不匹配的 `repoURL`、`repoBranch` 或 `revisionId`。 |
| 稀疏 DETAIL 处理 | v26.03 DETAIL 省略的行按有效 `genRatio=0` 处理。 |
| 行分类 | 根据所选 scope 路由 code 行和 doc 行。 |
| 聚合辅助逻辑 | full、partial、manual/unattributed、mostly-AI 行数。 |
| 策略选项 | `onMissing`、`onDuplicate`、`onClockSkew` 行为。 |

### 1.3 UnitTesting 规则

- 测试必须确定且快速。
- 除非被测单元明确是 VCS 命令适配器，否则测试不应要求真实 Git/SVN 仓库。
- 公式和校验逻辑优先使用很小的内联 fixture。
- 只有当 JSON 形状太大、内联后不易阅读时，才使用 golden JSON fixture。
- 测试失败信息应指出它保护的用户故事或验收标准。

### 1.4 推荐的单元测试命名

使用能保留追踪关系的名称：

```text
test_US001_AC001_1_weighted_mixed_authorship
test_US001_AC001_6_zero_denominator
test_US006_duplicate_revision_rejected
test_protocol_v2604_rejects_missing_embedded_blame
```

具体命名风格可以遵循 fork 所选语言和测试框架，但每个测试都应让所覆盖行为一眼可见。

---

## 2. SysTesting

SysTesting 把工具作为完整系统进行验证。这类测试执行真实的 `aggregateGenCodeDesc` 入口，并使用真实输入校验生成的输出产物。

### 2.1 CaTDD 角色

在 CaTDD 中，SysTesting 证明一个或多个 AC 经得起真实工具边界的检验：CLI 参数、仓库 fixture、genCodeDesc 文件、生成的 JSON、生成的 patch、日志和退出码。

好的 SysTesting 应回答：

- 哪个 AC 通过真实可执行路径得到验证？
- 涉及哪个算法、协议版本、VCS 模式和 scope？
- 产生并比对了哪些产物？
- 失败点是否能告诉开发者问题在解析、算法行为、输出成形还是诊断？

推荐的 CaTDD 测试用例形状：

```text
[@AC-001-1,US-001]
TC-Sys-001:
  @[Name]: verifyAggregateCli_byAlgAWithV2603Fixture_expectWeightedSeventySevenPercent
  @[Priority]: P1 Functional
  @[Category]: Typical
  @[Purpose]: Prove the documented metric is produced by the real CLI.
  @[Brief]: Run aggregateGenCodeDesc on a tiny Git fixture with v26.03 metadata.
  @[Expect]: JSON contains weighted=0.77 and patch artifact is created.
```

### 2.2 SysTesting 必须覆盖什么

每个 fork 至少应做以下系统测试：

| 领域 | 必需覆盖 |
| --- | --- |
| CLI 参数契约 | 必需参数、可选参数、非法参数组合、退出码。 |
| Algorithm A | 使用 v26.03 输入，对本地 Git 仓库执行实时 blame。 |
| Algorithm B | 使用 `commitPatchDir` 和 v26.03 输入进行离线 diff 回放。 |
| Algorithm C | 使用 v26.04 输入进行内嵌 blame 回放，且不访问活 VCS。 |
| 输出产物 | 同时生成 `genCodeDescV26.03.json` 和 `commitStart2EndTime.patch`。 |
| 输出 schema | 聚合 JSON 遵循 v26.03 外壳并包含 `AGGREGATE` 扩展。 |
| 诊断 | 正确输出缺失版本、重复版本、混合版本、时钟偏移和 warning。 |
| 仓库行为 | 根据适用情况覆盖 rename、delete、copy、merge、squash、cherry-pick、revert、rebase/force-push。 |
| Scope 行为 | Scope A/B/C/D 过滤器会一致地改变分母和输出。 |

### 2.3 时间窗口 Diff 与存活代码聚合契约

系统测试必须验证：`aggregateGenCodeDesc` 聚合的是时间窗口 diff 里的存活子集：

```text
aggregate set = (startTime..endTime diff 新增/修改过的行) ∩ (endTime 仍存活的行)
```

patch 产物和 JSON 指标使用同一个时间窗口和 scope，但回答不同的审计问题：

| 概念 | 含义 | 必需验证 |
| --- | --- | --- |
| `commitStart2EndTime.patch` | 如 [README_UserGuide_ZH.md](README_UserGuide_ZH.md) 所定义，从 `startTime` 之前的版本到 `endTime` 的累计 diff。 | 展示窗口内哪些行发生过变化，并作为 patch 产物保持可审计。 |
| diff 的存活子集 | 这个 diff 中当前版本在 `endTime` 仍然存活的行。 | 定义 JSON 指标分母，并排除已删除、已 revert、以及来源在窗口外的行。 |

必需的 CaTDD 对齐用例：

| TC | 来源追踪 | 预期区分 |
| --- | --- | --- |
| `TC-Sys-WindowDiffDeletedFile` | `[@AC-002-3,US-002]` | 已删除行可以作为删除内容出现在窗口 diff 中，但对存活代码度量贡献为零。 |
| `TC-Sys-WindowDiffRevertedLines` | `[@AC-003-4,US-003]` | 被 revert 的行在窗口历史中可见，但不在 `endTime` 的存活快照中。 |
| `TC-Sys-PreWindowAliveLineExcluded` | `[@AC-005-1,US-005]` | `startTime` 之前提交的行即使在 `endTime` 仍存活，也不进入窗口内分母。 |
| `TC-Sys-DiffPatchAndJsonAgreeOnScope` | `[@AC-001-8,US-001]` | `commitStart2EndTime.patch` 和 `genCodeDescV26.03.json` 使用同一个时间窗口和 scope 过滤器，但 JSON 分母只聚合 diff 的存活子集。 |

如果测试只检查 patch 文件存在，这是不够的。它还必须证明 JSON 指标来自 `(startTime..endTime diff) ∩ (endTime 仍存活)`，而不是直接来自原始 added/deleted diff 行，也不是来自仓库里所有存活行。

### 2.4 系统 fixture 策略

每个 fork 应保持 fixture 小，但行为覆盖丰富：

| Fixture 类型 | 推荐内容 |
| --- | --- |
| Tiny Git repos | 测试 setup 阶段用脚本创建，包含确定的提交和时间戳。 |
| genCodeDesc fixtures | 每个场景一个目录，每个 revision 一个 JSON 文件。 |
| Patch fixtures | Algorithm B 每个 revision 一个 patch 文件。 |
| Expected outputs | 对稳定场景提供 golden aggregate JSON 和 patch 文件。 |

系统测试可以在运行时生成临时仓库。如果把 fixture 仓库提交进仓库，应保持最小化，并记录它们的生成方式。

### 2.5 系统测试退出标准

在系统测试证明以下内容之前，fork 不应宣称实现完成：

1. 同一个用户故事在所有兼容算法下产生匹配的指标。
2. 不支持的协议/算法组合按文档退出码失败。
3. `stdout` 和输出文件是机器可读的，运行时日志进入 `stderr`。
4. 在期望确定性输出的场景中，重复运行同一测试产生字节稳定输出。

---

## 3. UserTesting

UserTesting 验证代码库维护者能否按照文档化流程操作并得到预期结果。这类测试面向验收，可以是手动的、脚本化的，也可以是 demo 风格。

### 3.1 CaTDD 角色

在 CaTDD 中，UserTesting 从维护者视角证明 UserStory 和 UserGuide 为真。它可以比 UnitTesting 粗粒度，也可以比 SysTesting 更少关注实现细节，但必须端到端展示用户价值。

好的 UserTesting 应回答：

- 维护者是否能在不了解实现内部的情况下跟随文档流程？
- 展示了哪个 US 和 AC？
- 应运行什么命令或手动步骤？
- 维护者应该看到什么结果，又应如何解释它？

推荐的 CaTDD 测试用例形状：

```text
[@AC-001-1,US-001]
TC-User-001:
  @[Name]: verifyGuideFlow_byDemoRepository_expectReadableAggregateMetrics
  @[Priority]: P4 Addons
  @[Category]: Demo/Example
  @[Purpose]: Demonstrate the guide workflow for a maintainer.
  @[Brief]: Follow README_UserGuide against a prepared demo repository.
  @[Expect]: User can identify Weighted, Fully AI, and Mostly AI results.
```

### 3.2 UserTesting 必须覆盖什么

每个 fork 至少应做以下用户测试：

| 领域 | 必需覆盖 |
| --- | --- |
| README_UserGuide 流程 | 维护者可以运行指南中的文档化命令。 |
| README_UserStories 场景 | 关键验收标准得到可展示的满足。 |
| 安全 agent 工作流 | 测试可以在 Dev Container 内运行，无需主机特定设置。 |
| 结果解释 | 用户可以在输出中识别 Weighted、Fully AI、Mostly AI 的值。 |
| 错误解释 | 用户能从 stderr/log 输出理解常见校验失败。 |
| 跨平台主机设置 | macOS、Linux、Windows 主机使用相同的容器化测试流程。 |

### 3.3 UserTesting 形式

一个 fork 可以使用以下一种或多种形式：

| 形式 | 适用时机 |
| --- | --- |
| 手动测试脚本 | 适合早期 fork，实现仍在变化时。 |
| Demo 测试用例 | 适合展示从 setup 到 expected output 的完整用户故事。 |
| 验收测试 | 适合 CLI 和输出契约稳定之后。 |
| 发布检查清单 | 适合打 tag 或比较多个 agent/LLM fork 之前。 |

### 3.4 必需的用户测试 README

每个面向用户的 demo 或手动测试都应包含一个短 README：

| Section | Content |
| --- | --- |
| Purpose | 这个测试展示哪个用户故事或指南工作流。 |
| Status | Draft、implemented、passing、failing 或 blocked。 |
| Covered | 覆盖的具体 AC ID、CaTDD 优先级/类别、算法模式、协议和 scope。 |
| Manual | 要运行的精确命令和预期输出摘要。 |

这能让用户测试对维护者保持可读，也让代码代理的工作更容易审阅。

---

## 4. 测试追踪矩阵

每个 fork 都应维护从需求到测试的追踪关系。

| 来源 | 测试层 |
| --- | --- |
| `README_UserStories.md` | CaTDD 的 US/AC 来源；UnitTesting 覆盖原子规则，SysTesting/UserTesting 覆盖完整场景。 |
| `README_UserGuide.md` | CaTDD 的用户工作流来源；SysTesting 覆盖 CLI 契约，UserTesting 覆盖文档化工作流。 |
| `README_Protocol.md` | UnitTesting 覆盖 schema/校验，SysTesting 覆盖真实协议文件。 |
| `README_AlgABC.md` | UnitTesting 覆盖算法辅助逻辑，SysTesting 覆盖完整算法行为。 |
| `Protocols/*.json` | UnitTesting 覆盖 schema 兼容性和 fixture 校验。 |

一个好的测试用例应能回答：它保护的是哪个用户故事、哪个验收标准、哪个 CaTDD 优先级/类别、哪个协议版本、哪个算法、哪个 scope？

---

## 5. Fork 的最小测试集合

在扩展更广覆盖之前，fork 应先从这个最小集合开始：

1. 三种度量公式的 UnitTesting，并链接到 US/AC ID。
2. 协议解析和校验错误的 UnitTesting，并链接到 US/AC ID。
3. 每个受支持算法一个 happy path 的 SysTesting，并链接到 US/AC ID。
4. 不支持的协议/算法组合的 SysTesting，并链接到 US/AC ID。
5. 一个文档化端到端指南流程的 UserTesting，并链接到指南和 US/AC ID。
6. 在 Dev Container 内运行并证明安全 agent 工作流的 UserTesting。

这个最小集合能在开发期间提供快速反馈，并足以建立 fork 满足 BASE 契约的信心。

---

## 6. 运行测试

由于每个 fork 会选择自己的实现语言，具体命令由 fork 决定。推荐约定如下：

| 语言 | UnitTesting | SysTesting/UserTesting |
| --- | --- | --- |
| Python | `pytest tests/unit` | `pytest tests/system tests/user` |
| C++ | `ctest -R unit` | `ctest -R "system\|user"` |
| Rust | `cargo test unit` | `cargo test system user` |

除非某个测试明确说明为什么必须在别处运行，否则所有命令都应在 Dev Container 内运行。

---

## 7. 完成定义

一个 fork 的测试可视为完成，当且仅当：

- UnitTesting 覆盖确定性逻辑和校验行为。
- SysTesting 覆盖真实 CLI、输出产物和算法/协议兼容性。
- UserTesting 证明维护者可以跟随指南并解释结果。
- UnitTesting、SysTesting 和 UserTesting 都保留从 US -> AC -> TC -> result 的 CaTDD 追踪关系。
- 每个 TC 都声明 CaTDD 优先级和类别。
- 每个失败或跳过的测试都有明确原因。
- 测试套件可以在安全容器工作流中运行，不依赖隐藏的主机状态。
