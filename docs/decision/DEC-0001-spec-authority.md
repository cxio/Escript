# DEC-0001 规格权威与外部算法

- 状态：Accepted
- 日期：2026-10-28
- 覆盖评审条目：P0-5（Q10、Q11）


## 背景

构想层有多处语义实际由 Go 标准库的行为决定：正则（`regexp` 与 Unicode 表）、格式化（`fmt`）、时间（`time.Time` 的格式化与时区）、随机数（原为 `math/rand/v2`）、变长整数（`binary.Varint`）、数值转换（`strconv`、`math/big`）。这些行为会随 Go 版本变化，其它语言也难以复现。

脚本结果属于共识。同一段脚本在两个节点上结果不同，就会分叉。因此需要先定下：脚本的语义以什么为准。


## 决策

### 1. 权威来源

**规格文档加测试向量是权威，Go 实现是参考实现。**

参考实现与规格不一致时，视为实现缺陷，修正实现。确实需要改变语义时，先修订 Decision 或 Spec，再改实现。

### 2. 行为按三类处理

| 类别 | 范围 | 规格的写法 |
| --- | --- | --- |
| 一、本设计自定义的语义 | 字节码编码、值类型、栈与实参区、控制流、通关状态、运算与比较、MODEL 匹配、限额与成本 | Spec 逐条定义，不引用任何库的行为 |
| 二、有公开标准的算法 | 哈希、Base32 / Base64、RFC 3339、UTF-8、签名算法 | Spec 写明标准名称和版本，见下表 |
| 三、由 Go 实现决定的行为 | 正则的 Unicode 类与大小写折叠、`fmt` 输出、`time` 的格式化与时区、随机数派生、`binary.Varint` 的宽松解码 | 移出公共域，或由 Spec 自行定义，见第 3 条 |

第二类的标准引用：

| 用途 | 标准 |
| --- | --- |
| `FN_HASH*` 算法 0 | BLAKE3（官方规范与测试向量） |
| `FN_HASH*` 算法 1 | BLAKE2b（RFC 7693） |
| `FN_HASH*` 算法 2 | SHA-2（FIPS 180-4） |
| `FN_HASH*` 算法 3 | SHA-3（FIPS 202） |
| `FN_BASE32` / `FN_BASE64` | RFC 4648，字母表与填充规则按构想层（Base32 大写无填充；Base64 URL 字母表无填充） |
| `DATE{}` 文本形式 | RFC 3339 |
| 字符串编码 | UTF-8（RFC 3629） |
| 签名验证 | 由宿主规格指定（见 DEC-0002），如 ML-DSA（FIPS 204）、SLH-DSA（FIPS 205） |

`FN_BASE58` / `FN_ADDRESS` 没有正式标准，归入第一类：Spec 自行写明字母表（Bitcoin 字母表）、前导零的编码规则和校验码算法。

### 3. 第三类行为的处理

| 行为 | 处理 |
| --- | --- |
| 正则 | 保留在公共域。规格写明「RE2 语法 + Unicode 版本 X」，X 取参考实现所用 Go 工具链的 `unicode.Version`。CI 断言该值。将来升级 Go 而 Unicode 版本变化时，把 `regexp` 及其 Unicode 表固化一份到项目内，不跟随升级 |
| `MO_FMT` | 私有域专属。公共节点上的 `PRINT`、`FN_PRINTF` 本来就是空操作，不受影响 |
| 时间 | 公共域只开放数值读取：年、月、日、时、分、秒、毫秒、Unix 时间戳（毫秒），时区只认 UTC 和固定偏移。`Format`、`Local`、`In` 等依赖格式化或时区数据库的方法为私有域专属。`MO_TIME` 与 Time 对象的方法表按此标注公/私（见 DEC-0004） |
| 随机数 | 采用构想层 `RANDOM` / `SLRAND` 已写明的 BLAKE3 keyed + XOF 算法，见第 4 条 |
| 变长整数 | 规范解码规则见 DEC-0003 |
| `strconv` / `math/big` | 由 Spec 定义：`INT(String)` 接受的文本文法（按构想层：可选符号、十进制或 `0b` / `0o` / `0x` 前缀、数字间单下划线分隔）；`STRING(Int)` / `STRING(BigInt)` 在 2–36 进制下的输出格式（小写字母、无前导零）；`STRING(Byte)` / `STRING(Rune)` 的「合法字符」按合法 Unicode 标量值判定，不使用随 Unicode 版本变化的「可打印」判断；BigInt 的位运算语义（二进制补码，见 DEC-0005） |
| 数据栈 `[]any` | 属于实现细节。Spec 按 DEC-0005 的值模型定义数据栈，不涉及 Go 类型 |

### 4. 随机数算法的补充约定

构想层 `14.工具指令.md` 已写明算法，这里只补三条 Spec 需要的约定：

- 种子必须恰好 32 字节，否则失败。
- 每次 `RANDOM` / `SLRAND` 调用都从 XOF 输出流的起点开始读取。同一种子、同一实参，结果相同。
- tag 是 ASCII 字节串，按构想层写法为 `Escript/RANDOM/v1` 与 `Escript/SLRAND/v1`（大小写敏感）。

`RANDOM` 的 Float 分支在公共域会产生 Float，按 DEC-0005 即失败，只在私有域有效。

### 5. 测试向量

- 每条指令都要有正常、边界、失败三类用例。格式为「字节码 + 环境 → 结果 + 成本」，导出为 JSON。
- 向量集随仓库发布，供其它语言的实现对照。这与 Bitcoin 的 `script_tests.json`、Ethereum 的 `ethereum/tests` 做法相同。
- TDD 本来就要写这些用例，导出是附带的工作。

### 6. 工具链

`go.mod` 固定 toolchain 版本。升级 Go 必须全量通过向量集，并通过 Unicode 版本断言。


## 理由

- 规格自足是兼容实现的前提，也是「通用化使用」的前提。但初期没有必要从零重写正则引擎这类成熟组件，只需要把会随版本漂移的部分圈出来。
- 第一类本来就是 Spec 要写的内容；第二类的标准自带测试向量；真正有成本的只有第三类，而其中多数（`fmt`、时间格式化）对共识没有价值，移到私有域是最便宜的做法。
- 随机数改用 BLAKE3：它已经是系统的默认哈希，不增加依赖；密钥恰好 32 字节，与种子长度一致；各语言都有实现和官方测试向量。ChaCha8 在 Go 之外的实现很少，`IntN`、`Perm` 等派生方法也不在其规范内。


## 影响

- Spec 需要一张「外部标准引用表」（即第 2 条的表），以及正则的 Unicode 版本号。
- `MO_FMT`、Time 的格式化与时区方法在方法表中标为私有域专属；公共验证调用即失败。
- Plan 需要包含：测试向量的导出格式、CI 中的 Unicode 版本断言、toolchain 固定。
- 其它语言实现只需对照 Spec 与向量集，不必阅读 Go 源码。


## 构想层依据

- `conception/Design.md`：数据栈（`[]any`）
- `conception/Instruction/1.值指令.md`：正则表达式（RE2）、`DATE{}`（RFC 3339）
- `conception/Instruction/14.工具指令.md`：`RANDOM`、`SLRAND` 的算法；`CALL`
- `conception/Instruction/16.函数指令.md`：`FN_HASH*` 算法标识、`FN_BASE32` / `FN_BASE64` 规范
- `conception/Instruction/17.模块指令.md`：`MO_FMT`、`MO_TIME`
- 评审问答：P0-5，Q10、Q11


## 开放问题

1. 正则 Unicode 版本的初值，在参考实现选定 Go 工具链时记录。
2. `FN_BASE58` / `FN_ADDRESS` 的输入长度上限与成本，见 DEC-0007。
