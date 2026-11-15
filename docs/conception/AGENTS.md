# 设计构想（Conception）

主要由用户完成（可能与 AI 头脑风暴），主要是探索思考、分析问题，澄清目标，以及为什么。主要是 **What** 和 **Why** 的逻辑。

不过如果必要，也可以包含具体的想法和细节，或者某些技术选择。

## 文件清单

|    功能    |    文件    |  简要说明  |
|------------|------------|------------|
| 栈脚本设计 | [Design.md](Design.md) | 数据栈、实参区、附参与执行模型；指令码分段、命名与特性指令；解析规则、未知码位与版本升级；限额、成本、锁定/解锁；源码格式与 `SYS_CHKPASS` 超级赎回 |
| 基本约束 | [Instruction/0.基本约束.md](Instruction/0.基本约束.md) | 脚本只读；栈、实参区、局部域等概念；指令结构与说明格式；符号/命名指令与源码注释 |
| 值指令 | [Instruction/1.值指令.md](Instruction/1.值指令.md) | 码位 `[0-17]` 的字面量自动入栈（`NIL`、数值、字符串、`DATA`、`DICT`、`CODE`、`SCRIPT` 等），以及解锁区取值 `GVAL`（码位 18） |
| 截取指令 | [Instruction/2.截取指令.md](Instruction/2.截取指令.md) | 码位 `[19-24]`：`@`、`&`、`~`、`_`，拦截返回值到实参区或局部域，或取值、丢弃 |
| 栈操作指令 | [Instruction/3.栈操作指令.md](Instruction/3.栈操作指令.md) | 码位 `[25-35]`：入栈、出栈、引用与转移（`PUSH`、`POP`/`POPS`、`TOP`、`PEEK`、`SHIFT`、`CLONE`、`VIEW`） |
| 集合指令 | [Instruction/4.集合指令.md](Instruction/4.集合指令.md) | 码位 `[36-45]`：切片、字典与 `Bytes` 的截取、合并、展开、索引与设值；解锁脚本可用的末段 |
| 交互指令 | [Instruction/5.交互指令.md](Instruction/5.交互指令.md) | 码位 `[46-50]`：经独立缓存区导入导出（`INPUT`、`OUTPUT`、`BUFDUMP`）与控制台打印 |
| 结果指令 | [Instruction/6.结果指令.md](Instruction/6.结果指令.md) | 码位 `[51-57]`：通关、跳转与结束（`PASS`、`CHECK`、`GOTO`、`EMBED`、`EXIT`、`RETURN`、`END`） |
| 流程控制 | [Instruction/7.流程控制.md](Instruction/7.流程控制.md) | 码位 `[58-66]`：`IF`/`ELSE`、`SWITCH`/`CASE`、`EACH` 有限循环、`BREAK`/`CONTINUE`、`BLOCK` |
| 转换指令 | [Instruction/8.转换指令.md](Instruction/8.转换指令.md) | 码位 `[67-79]`：类型与值转换（`BOOL`、`INT`、`FLOAT`、`STRING`、`BYTES`、`TIME`、`ANYS` 等），不提供转到 `Script` |
| 运算指令 | [Instruction/9.运算指令.md](Instruction/9.运算指令.md) | 码位 `[80-103]`：表达式分组、符号与命名四则、位运算，以及 `DIVMOD`、`REP`、`DEL`、`CLEAR` |
| 比较指令 | [Instruction/10.比较指令.md](Instruction/10.比较指令.md) | 码位 `[104-111]`：相等与大小比较、异常浮点、`WITHIN` 范围判断 |
| 逻辑指令 | [Instruction/11.逻辑指令.md](Instruction/11.逻辑指令.md) | 码位 `[112-115]`：`BOTH`、`EITHER`、`EVERY`、`SOME` |
| 模式指令 | [Instruction/12.模式指令.md](Instruction/12.模式指令.md) | 码位 `[116-127]`：`MODEL` 形式化匹配与取值（通配、可选、类型、范围、正则） |
| 环境指令 | [Instruction/13.环境指令.md](Instruction/13.环境指令.md) | 码位 `[128-137]`：系统、交易、校验与脚本域取值（`ENV`、`IN`、`OUT`、`VAR`、`SOURCE` 等） |
| 工具指令 | [Instruction/14.工具指令.md](Instruction/14.工具指令.md) | 码位 `[138-163]`：`EVAL`、`COPY`、`MATCH`、`CALL`、`RANGE`、`MAP`/`FILTER`、`SHELL`；`156-163` 为量子安全保留 |
| 系统指令 | [Instruction/15.系统指令.md](Instruction/15.系统指令.md) | 码位 `[164-169]`：`SYS_TIME`、`SYS_AWARD`、`SYS_CHKPASS`、`SYS_NULL` |
| 函数指令 | [Instruction/16.函数指令.md](Instruction/16.函数指令.md) | 码位 `[170-220]`：编码、地址、签名与哈希等基础函数，以及 `FN_X` / `FN_PRIV` 扩展引用 |
| 模块指令 | [Instruction/17.模块指令.md](Instruction/17.模块指令.md) | 码位 `[221-250]`：`MO_TIME`、`MO_RE`、`MO_BINT`、`MO_MATH`、`MO_FMT` |
| 扩展指令 | [Instruction/18.扩展指令.md](Instruction/18.扩展指令.md) | 码位 `[251-253]`：`EXT_MO` 扩展模块与 `EXT_PRIV` 私有扩展 |


**维护规则：**

- 当用户或用户委托 Agent 完成一份构想文档时，更新上面的清单。
