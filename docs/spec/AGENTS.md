# 技术规格（Spec）

定义目标具体什么样，精确的技术规格。如模块划分、接口与数据设计、字节编码、字段宽度、实现的细则等。完成的是 **What exactly must the system do**。

该部分由 AI 根据 Conception 和 Decision 的内容分析创建出来，注意在简洁和足够的细节之间保持平衡。衡量标准是：Plan 可以遵循这些细粒度规格编写实现计划而不需要自由发挥。

文档内容大致遵循如下结构：

- 来源追溯
- 概述
- 规格正文
- 边界与限制
- 待决问题
- 对 Plan 的约束

如果您在思考和编写规格的过程中，发现上级构想/决策有问题，或实现路径并不是最佳，请向作者反馈，提出您的质疑和建议。必要的返工是合理的，这比僵化地按原始构想/决策变通（绕弯）地完成任务更有意义。


## 文件清单

|    编号    |    文件    |  覆盖主题  |
|------------|------------|------------|
| SPEC-0001 | `SPEC-0001-overview.md` | 总览、约定、外部标准引用、全局待决集（P-NN）与评审者补充细则（R-NN） |
| SPEC-0002 | `SPEC-0002-bytecode-format.md` | 字节码格式：附参类型、变长整数、关联数据、逐条指令格式表、由版本取值表给出的附参 |
| SPEC-0003 | `SPEC-0003-load-validation.md` | 加载期校验：段与加载时点、加载步骤、规范解码、结构规则、表达式文法、模式体、段级规则、加载期成本 |
| SPEC-0004 | `SPEC-0004-value-model.md` | 值模型：类型、Float 规则、数值提升、比较、值语义、`size` |
| SPEC-0005 | `SPEC-0005-execution-model.md` | 执行模型：状态、实参与去向、块与局部域、通关状态机、结果、公共 / 私有执行、GOTO / EMBED / EVAL / MAP / FILTER、循环与 SWITCH、校验域 |
| SPEC-0006 | `SPEC-0006-limits-and-cost.md` | 硬限额与检查时点、成本框架与计费时点、成本记法（`K`、`B(x)`）、初始成本表、三层预算 |
| SPEC-0007 | `SPEC-0007-versioning.md` | 版本号、每段的版本、版本冻结的表、未分配与未知版本即失败、跨版本规则、升级规则（预发布） |
| SPEC-0008 | `SPEC-0008-host-interface.md` | 宿主接口：宿主规格的必备内容、`ENV` 索引划分与核心字段、环境、接口契约、创建时的检查、执行结果与标记、私有执行 |
| SPEC-0101 | `SPEC-0101-core-instructions.md` | 指令语义的通用约定（0101–0106 共用）；值、截取、栈操作、集合、交互、结果、流程控制指令 |
| SPEC-0102 | `SPEC-0102-conversion-arithmetic.md` | 转换（含 `INT` / `TIME` 的文本文法、UTF-8 解码）、运算、比较、逻辑指令 |
| SPEC-0103 | `SPEC-0103-model-matching.md` | MODEL：目标解码、模式单元、深匹配、`^?` 的部分编码、类型标识表、范围与正则、求解与回溯、取值、成本 |
| SPEC-0104 | `SPEC-0104-environment-tools-functions.md` | 正则约定与全部匹配算法；环境（含 `SOURCE`）、工具（`MATCH`、`REPLACE`、`RANDOM`、`CALL` 等）、系统、函数（编码与哈希）指令 |
| SPEC-0105 | `SPEC-0105-module-methods.md` | 模块与扩展：调用方式、v1 方法表（`MO_TIME`、`MO_RE`、`MO_BINT`、`MO_MATH`、`MO_FMT`、Dict 的 `CALL` 方法） |
| SPEC-0106 | `SPEC-0106-private-execution.md` | 私有执行：Float、私有通道、`SYS_TIME`、`SHELL`、`FN_PRIV` / `EXT_PRIV`、值的默认文本形式 |
| SPEC-0201 | `SPEC-0201-text-syntax.md` | 文本语法：注释、词法、字面量、附参与别名、块与表达式、模式文本、规范反编译 |
| SPEC-0202 | `SPEC-0202-test-vectors.md` | 测试向量的 JSON 格式、环境夹具、成本断言 |
| SPEC-0901 | `SPEC-0901-evidcoin-host.md` | Evidcoin 宿主：预发布、字段与见证、签名消息、地址、`SYS_CHKPASS`、创建时检查 |


**维护规则：**

- Spec 服从于 Conception 与 Decision。
- 每篇「来源追溯」必须可回溯到具体 Conception 章节与 `DEC-<NNNN>`。
- 待决项严格限于全局待决集，相关规格须显式标注，不得默认选值固化。
