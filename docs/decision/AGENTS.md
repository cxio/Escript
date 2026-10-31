# 架构决策（Decision）

定义目标的解决方向、方式，构想没有明确的技术选型、实现路径、必要的关键细节等。完成的是 **Which** 和 **Why this**。由 AI 提出，人类参与互动讨论，最终由人类拍板确定。

不涉及详细的技术规格——评判标准是：如果没有这些决策，技术规格（Spec）将无法精确细化。但显而易见的常规细节处理，则无需进入决策层。

文档内容基本上遵循如下结构：

- 背景 （Context）
- 决策 （Decision）
- 理由 （Rationale）
- 影响 （Consequences）
- 构想层依据 （Conception References）
- 开放问题 （Open Questions）

> **注：**
> 某些时候可能需要明确*不做什么*，以旁证决策的合理性。

## 文件清单

|    编号    |    文件    |  覆盖主题  |
|------------|------------|------------|
| DEC-0001 | [DEC-0001-spec-authority.md](DEC-0001-spec-authority.md) | 规格权威与外部算法：标准引用、Go 行为的固定方式、随机数、测试向量 |
| DEC-0002 | [DEC-0002-core-vm-and-host.md](DEC-0002-core-vm-and-host.md) | 核心 VM 与宿主接口：职责划分、宿主接口、数据可用性、Evidcoin 宿主 |
| DEC-0003 | [DEC-0003-bytecode-and-load-validation.md](DEC-0003-bytecode-and-load-validation.md) | 字节码解码与加载期校验：规范解码、结构规则、解锁/锁定边界、系统占位符 |
| DEC-0004 | [DEC-0004-versioning-and-upgrade.md](DEC-0004-versioning-and-upgrade.md) | 脚本版本与升级：版本号、冻结内容、未分配码位的宽松通过、软分叉 |
| DEC-0005 | [DEC-0005-values-types-errors.md](DEC-0005-values-types-errors.md) | 数值、类型与异常：浮点限制、类型提升、溢出、比较、转换、错误分类 |
| DEC-0006 | [DEC-0006-execution-semantics.md](DEC-0006-execution-semantics.md) | 执行语义：通关状态机、执行上下文、块与局部域、表达式求值、公私模式 |
| DEC-0007 | [DEC-0007-limits-and-cost.md](DEC-0007-limits-and-cost.md) | 资源限额与成本模型：硬限额、成本函数、初始成本表、预算 |


**维护规则：**

- 新增 Decision 前必须先检查 Conception 部分是否已经明确该规则。
- 若 Conception 已明确，直接引用而不是新增 Decision。
- 若后续 Conception 修订吸收了某个 Decision，应删除该 Decision 或标注已吸收。
- Decision 文件命名为 `DEC-<NNNN>-<short-description>.md`。其中 `<NNNN>` 为序号，如 `0201`（有类别） 或 `0001`（无类别）。
