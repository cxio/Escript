# SPEC-0202 测试向量

- 状态：草案，待作者审定
- 日期：2026-11-13
- 修订：2026-11-16（`SHELL` 取消私有专属：夹具改为节点配置 `shell`，期望改为 `expect.shell`；增加公共执行中开启本地输出的夹具 `local`：第 3.2、4、4.3、6.7 条）


## 来源追溯

- DEC-0001 第 1、5 条（规格文档加测试向量是权威；「字节码 + 环境 → 结果 + 成本」，导出为 JSON，随仓库发布）
- DEC-0002 影响（核心可以单独生成测试向量，宿主的变化不影响核心的向量集）
- DEC-0004 影响（版本相关的必测情形）
- SPEC-0005 第 9 条（验证结果）、SPEC-0006 第 3、4 条（成本与预算）、SPEC-0001 第 6.2 条 R-18（只有通过时成本属于共识）
- SPEC-0008（宿主接口；第 6 条执行结果与标记；「对 Plan 的约束」中的测试宿主）
- SPEC-0106 第 3、4、6 条（私有通道与私有专属指令）、SPEC-0104 第 3.13 条（`SHELL`）
- SPEC-0201（文本，用于文本向量）、SPEC-0901（Evidcoin 宿主向量）


## 概述

测试向量与规格文档一起构成权威（DEC-0001）。其它语言的实现只对照规格与向量集，不需要阅读 Go 源码。本篇定义以下内容：

- 向量文件的 JSON 结构；
- 值的 JSON 编码；
- 四种向量：加载、执行、文本、宿主；
- 环境夹具与判定规则；
- 核心测试宿主：不依赖任何链的宿主配置，用于生成核心向量；
- Evidcoin 宿主向量的补充约定。

本篇的格式与测试宿主都是评审者补充的细则〔R-50、R-51〕。


## 规格正文

### 1. 文件〔R-50〕

一个向量文件是一个 UTF-8 编码的 JSON 对象（RFC 8259）：

```json
{
  "format": "escript-vectors/1",
  "host": "test",
  "version": 1,
  "costs": "spec-0006-draft",
  "vectors": [ ]
}
```

| 字段 | 含义 |
| --- | --- |
| `format` | 固定为 `escript-vectors/1`。格式发生不兼容的改变时，改为 `/2` |
| `host` | 宿主配置：`test` 为核心测试宿主（第 6 条），`evidcoin` 为 SPEC-0901 |
| `version` | 默认的核心版本号，单个向量可以覆盖 |
| `costs` | 成本断言所依据的成本参数集的标识。成本系数在 P-02 决定之前是校准起点（SPEC-0006），标识为 `spec-0006-draft`；参数集改变时，标识随之改变，向量重新生成 |
| `vectors` | 向量的数组 |

每个向量是一个 JSON 对象，共同字段如下：

| 字段 | 含义 |
| --- | --- |
| `id` | 字符串，在文件内唯一 |
| `kind` | `load`、`exec`、`text`、`host` 之一（第 3 条） |
| `desc` | 可选，说明 |
| `refs` | 可选，相关的规格条款，字符串数组，如 `["SPEC-0101 §3"]` |
| `version` | 可选，覆盖文件的默认版本 |
| `pending` | 可选，期望值依赖的待决项，如 `["P-08"]`。这些向量在待决项决定后重新生成 |

书写约定：

- 字节写作小写十六进制字符串，不带 `0x`。
- 值的载荷中的整数写作十进制字符串（第 2 条），避免超出 2⁵³ 的精度问题。成本、预算、版本号、序位写作 JSON 数。
- 以 `x-` 开头的字段供工具使用，实现必须忽略。其它未知字段使该向量无效，以免拼错的字段名让断言悄悄失效。

### 2. 值的 JSON 编码〔R-50〕

值编码为一个带 `t`（类型）字段的对象：

| 类型 | 编码 |
| --- | --- |
| Nil | `{"t":"Nil"}` |
| Bool | `{"t":"Bool","v":true}` |
| Byte | `{"t":"Byte","v":65}` |
| Rune | `{"t":"Rune","v":19990}` |
| Int | `{"t":"Int","v":"-5"}`，十进制字符串 |
| BigInt | `{"t":"BigInt","v":"-170141183460469231731687303715884105728"}`，十进制字符串 |
| Float | `{"t":"Float","v":"3ff0000000000000"}`，binary64 的 16 位小写十六进制位模式（可以表示 NaN、±Inf 与 −0） |
| String | 内容是合法 UTF-8 时写作 `{"t":"String","v":"Hello"}`，否则写作 `{"t":"String","hex":"ff00"}` |
| Bytes | `{"t":"Bytes","v":"46af"}` |
| RegExp | `{"t":"RegExp","v":"(?i)abc"}`，源文本 |
| Time | `{"t":"Time","v":"1608883593536","offset":28800}`：毫秒时刻（十进制字符串）与偏移（秒）。偏移为 0 时可以省略 |
| Script | `{"t":"Script","v":"<hex>","version":1,"origin":"CODE"}`：字节码、版本，以及由 `CODE` 还是 `SCRIPT` 创建（`ENV` 的 `InScript`、`InCode` 依赖它）。`SCRIPT` 取得未知版本时（SPEC-0007 第 2 条），`version` 为 `null` |
| 切片 | `{"t":"Slice","elem":"Int","v":[ … ]}`：`elem` 为 `any` 或成员类型名；`v` 中每个成员都按本表编码，`elem` 不是 `any` 时成员的 `t` 必须等于 `elem` |
| Dict | `{"t":"Dict","v":[[K, V], …]}`：按插入顺序列出键值对，K 按 String 编码 |
| 模块对象 | `{"t":"Module","v":"MO_MATH"}`；扩展模块写作 `"EXT_MO:<模块索引>"` |

判定向量时，两个值相等，当且仅当 `t` 相同且载荷相同。这比语言的相等比较（SPEC-0004 第 5 条）更严格。具体地：String 比较字节；Float 比较位模式；Time 比较时刻与偏移；Script 比较字节码、版本与来源；切片比较 `elem` 与逐个成员；Dict 比较有序的键值对。

### 3. 向量的种类

#### 3.1 加载向量 `load`

```json
{ "id": "load-0001", "kind": "load", "segment": "lock",
  "bytes": "06084000000000000000",
  "expect": { "result": "fail" } }
```

| 字段 | 含义 |
| --- | --- |
| `segment` | 段的种类：`unlock`、`lock`、`goto`、`embed`、`eval` |
| `bytes` | 段的字节码 |
| `text` | 可选，对照用的文本（SPEC-0201），不参与判定 |
| `budget` | 可选，加载时可用的成本预算，默认为宿主的 `MaxInputScriptCost` |
| `expect.result` | `ok` 或 `fail` |
| `expect.cost` | `ok` 时必须给出，为加载期成本（SPEC-0003 第 8 条）；`fail` 时不得给出 |

判定：按 SPEC-0003 在给定版本、段的种类与宿主配置下加载，结论与 `expect.result` 相同；成功时成本也相同。

#### 3.2 执行向量 `exec`

```json
{ "id": "exec-0001", "kind": "exec",
  "unlock": "…", "lock": "…",
  "mode": "public",
  "env": { },
  "expect": { "result": "pass", "cost": 123, "height": false } }
```

| 字段 | 含义 |
| --- | --- |
| `unlock`、`lock` | 解锁段与锁定段的字节码。`version` 是被花费输出的版本，两段都按它加载（SPEC-0007 第 2 条） |
| `unlockText`、`lockText` | 可选，对照用的文本，不参与判定 |
| `mode` | `public`（默认）或 `private` |
| `budget` | 可选，`MaxInputScriptCost`，默认按宿主 |
| `env` | 环境夹具（第 4 条） |
| `expect.result` | `pass`、`fail`、`unavailable` |
| `expect.cost` | 结果为 `pass` 时必须给出；其它结果不得给出〔R-18〕 |
| `expect.height` | 可选，「高度相关」执行标记（SPEC-0008 第 6 条） |
| `expect.exit` | 可选，仅用于 `pass`：顶层 `EXIT` 的实参（第 2 条编码），没有实参时为 `null` |
| `expect.stack` | 可选，仅用于 `pass`：结束时数据栈的内容，自底向上 |
| `expect.calls` | 可选：按顺序发起的宿主接口调用的种类，`load`、`sig`、`func`（SPEC-0008 第 5.1–5.3 条）。用于断言「预算不足时没有发起调用」 |
| `expect.private` | 可选，用于 `mode` 为 `private`，或 `mode` 为 `public` 且夹具 `local` 为 true：私有通道与本地输出的输出（第 4.3 条） |
| `expect.shell` | 可选，仅用于夹具 `shell` 为 `exec`：按顺序交给执行器的内容（第 4.3 条） |

判定：结果与 `expect.result` 相同，并且给出的其它断言都成立。

#### 3.3 文本向量 `text`

```json
{ "id": "text-0001", "kind": "text", "segment": "lock",
  "text": "TRUE  PASS", "bytes": "0133", "canonical": "TRUE PASS\n" }
{ "id": "text-0002", "kind": "text", "text": "{007}", "error": true }
```

| 字段 | 含义 |
| --- | --- |
| `text` | 源文本 |
| `bytes` | 编译结果 |
| `error` | 为 `true` 时，期望编译错误；此时不得给出 `bytes` 与 `canonical` |
| `segment` | 可选，段的种类。给出时，`bytes` 必须能按该种类加载，并且检查反编译 |
| `canonical` | 可选，规范文本。给出 `segment` 而省略 `canonical` 时，规范文本就是 `text` |

判定：

1. 编译 `text` 得到 `bytes`（或者在 `error` 时编译失败）。
2. 给出 `segment` 时，反编译 `bytes` 得到的文本与规范文本逐字符相同，并且编译规范文本仍得到 `bytes`。

文本向量检查编译器与反编译器，不检查语义；语义以加载与执行向量的字节码为准。

#### 3.4 宿主向量 `host`

```json
{ "id": "evd-pre-0001", "kind": "host", "case": "prerelease",
  "input": { }, "expect": { } }
```

宿主专属的检查，例如 Evidcoin 的预发布统计与创建时检查，不经过脚本执行。`case` 的取值以及 `input`、`expect` 的结构由宿主定义，Evidcoin 的定义见第 7 条。核心测试宿主没有宿主向量。

### 4. 环境夹具 `env`

```json
"env": {
  "fields":  { "GVAL": { "Height": { "t": "Int", "v": "100" } }, "IN": { } },
  "outputs": [ { "Amount": { "t": "Int", "v": "5" } } ],
  "inouts":  [ ],
  "unavailable": [ "IN.Memo", "INOUT.0.Locked" ],
  "witness": { "Sig": { "t": "Bytes", "v": "…" } },
  "chkpass": { "type": 1, "flag": 129 },
  "sigmsg":  { "81": "…" },
  "scripts": [ { "ref": [2026, "0a0b0c0d", 0], "code": "…", "version": 1 },
               { "ref": [2026, "0a0b0c0d", 1], "result": "missing" } ],
  "local":   false,
  "shell":   "off",
  "private": { }
}
```

全部字段都可以省略。

#### 4.1 字段与见证

| 字段 | 含义 |
| --- | --- |
| `fields` | 按指令（`GVAL`、`ENV`、`IN`、`XFROM`）分组，以字段名为键给出字段的值。`ENV` 中只能给宿主字段，核心字段由核心计算 |
| `outputs` | `OUT` 的输出集，数组下标就是输出序位，每个成员以字段名为键 |
| `inouts` | `INOUT` 的输出集，结构同上 |
| `unavailable` | 数据不可用的字段，写作 `指令.字段` 或 `指令.序位.字段` |
| `witness` | 见证条目，以条目名为键（不带尖括号） |
| `chkpass` | `SYS_CHKPASS` 从见证信息取得的其余项：签名类别 `type` 与授权标识 `flag`（Byte 的值） |
| `sigmsg` | 签名消息：以授权标识的两位小写十六进制为键，值为消息的字节（十六进制）。签名消息的构造属于宿主（SPEC-0008 第 5.2 条），向量直接给出构造的结果 |

- 字段的值必须符合宿主字段表声明的类型，或者为 Nil，否则该向量无效。
- 夹具中没有给出的字段，按宿主字段表声明的「取不到」处理（SPEC-0008 第 4 条第 4 项）。

#### 4.2 外部脚本

`scripts` 是外部脚本库。每个条目的 `ref` 是按宿主的引用字段序列给出的数组：整数字段写作 JSON 数（超出 2⁵³ 时写作十进制字符串），`bytes(N)` 字段写作十六进制字符串。

| 条目 | 加载结果（SPEC-0008 第 5.1 条） |
| --- | --- |
| 有 `code` 与 `version` | 字节码与版本。`version` 不是已发布的核心版本号时，宿主返回「未知版本」 |
| `"result": "missing"` | 不存在 |
| `"result": "unavailable"` | 数据不可用 |
| `scripts` 中没有与引用相同的条目 | 不存在 |

#### 4.3 私有执行与本地输出

```json
"private": {
  "now":   { "t": "Time", "v": "1767225600000", "offset": 28800 },
  "input": [ { "t": "Int", "v": "1" } ]
}
```

| 字段 | 含义 |
| --- | --- |
| `now` | `SYS_TIME` 读取的当前时间（时刻与本地偏移），整个执行中固定（SPEC-0106 第 4 条允许宿主固定时间）。没有给出时，`SYS_TIME` 失败 |
| `input` | 执行前注入 INPUT 缓存区的数据 |

`env` 中另有两项节点配置：

| 字段 | 含义 |
| --- | --- |
| `local` | 仅用于 `mode` 为 `public`：为 true 时，节点开启本地输出 `PRINT`、`FN_PRINTF`、`OUTPUT`、`BUFDUMP`（SPEC-0106 第 3 条），其输出按下表写在 `expect.private` 中。默认 false |
| `shell` | 两种模式都适用。`SHELL` 的节点配置（SPEC-0104 第 3.13 条）：`off` 开关关闭（默认）；`on` 开关开启、没有注册执行器；`exec` 开关开启，并注册一个只做记录的执行器。测试中不启动任何进程 |

`shell` 为 `exec` 时，`expect.shell` 按顺序列出每次交给执行器的内容：`{"cmd": "命令文本", "source": [值…]}`。`cmd` 为 JSON 字符串（命令文本总是合法的 UTF-8），`source` 为来源输出数据，按宿主的字段顺序以第 2 条编码（测试宿主见第 6.7 条）。

期望的输出写在 `expect.private` 中：

| 字段 | 含义 |
| --- | --- |
| `dumps` | 按顺序列出每次 `BUFDUMP` 交给处理器的内容：`{"id": n, "items": [值…]}` |
| `buffer` | 结束时 OUTPUT 缓存区中剩余的值 |
| `console` | `PRINT` 与 `FN_PRINTF` 写到控制台的全部文本，按顺序连接：合法 UTF-8 时写作 JSON 字符串，否则写作 `{"hex": "…"}` |

### 5. 判定〔R-50〕

- 一个向量通过，当且仅当它给出的全部断言都成立。没有给出的项不判定。
- 诊断信息（失败的位置、原因）不判定（SPEC-0003、SPEC-0008 第 6 条）。
- 成本断言只在加载成功与执行通过时出现〔R-18〕。它的数值依赖文件的 `costs` 标识：实现使用的成本参数集与标识不符时，跳过成本断言并报告，不判为失败。
- 依赖待决项（`pending`）的向量照常判定。待决项决定后，这些向量随之重新生成。

### 6. 核心测试宿主 `test`〔R-51〕

SPEC-0008 要求核心提供一个不依赖任何链的测试宿主。本条给出它的完整配置。它只用于测试向量，不用于任何链；它的取值不决定任何 P 项，Evidcoin 的取值见 SPEC-0901。

#### 6.1 版本与外部引用

- 版本：夹具中的版本号就是核心版本号，没有映射。新版本的接受不适用：本实现已实现的核心版本都可以直接使用。
- 外部脚本引用的字段序列：`uvarint, bytes(4), uvarint`。字段没有额外的约束。取数按第 4.2 条。

#### 6.2 环境字段表

所有字段在夹具中没有给出时，取值为 Nil。例外：`IN` 的 `Locked`、`Unlock` 没有给出时，取向量的 `lock`、`unlock` 字节。「源码类」字段的大小上限为 16KB（SPEC-0006 第 1 条）。

| 指令 | 索引 | 名称 | 类型 | 标注 |
| --- | --- | --- | --- | --- |
| `GVAL` | 0 | `Height` | Int | 高度相关 |
| `GVAL` | 1 | `TxID` | Bytes | |
| `ENV` | 32 | `Height` | Int | 高度相关 |
| `ENV` | 33 | `Time` | Int | 高度相关 |
| `ENV` | 34 | `TxID` | Bytes | |
| `IN` | 0 | `Amount` | Int | |
| `IN` | 1 | `Receiver` | Bytes | |
| `IN` | 2 | `Memo` | String | |
| `IN` | 3 | `Stamp` | Time | |
| `IN` | 4 | `Locked` | Bytes | 源码类 |
| `IN` | 5 | `Unlock` | Bytes | 源码类 |
| `OUT`、`INOUT` | 0 | `Amount` | Int | |
| `OUT`、`INOUT` | 1 | `Receiver` | Bytes | |
| `OUT`、`INOUT` | 2 | `Memo` | String | |
| `OUT`、`INOUT` | 3 | `Locked` | Bytes | 源码类 |
| `XFROM` | 0 | `Amount` | Int | |
| `XFROM` | 1 | `Locked` | Bytes | 源码类 |
| `XFROM` | 2 | `Source` | Bytes | 源码类 |

表中没有的索引未分配。

#### 6.3 见证条目表

| 索引 | 名称 | 类型 |
| --- | --- | --- |
| 0 | `<Sig>` | Bytes |
| 1 | `<PubKey>` | Bytes |
| 2 | `<Sigs>` | `[]Bytes` |
| 3 | `<PubKeys>` | `[]Bytes` |
| 4 | `<Rests>` | `[]Bytes` |

夹具中没有给出的条目，执行到对应的占位符即失败（SPEC-0008 第 4 条第 6 项）。

#### 6.4 伪签名

- 授权标识 f 的签名消息 M(f) 取自夹具 `sigmsg`，没有给出时为空。
- 公钥 P（非空的 Bytes）对 f 的有效签名定义为 `BLAKE3-256("Escript/TESTSIG/v1" ‖ P ‖ f ‖ M(f))`，其中 f 为 1 字节，标签按 ASCII 字节。
- 签名列表可以是 `[]Bytes`，也可以是成员全为 Bytes 的 `[]any`。

| 指令 | 规则 | 写入校验域（EVAL 中不写） |
| --- | --- | --- |
| `FN_CHECKSIG(f, sig, P)` | f 为 Byte，sig、P 为 Bytes，P 非空，否则失败。返回 sig 是否等于有效签名 | 序位 0 为验证结果 |
| `FN_MCHECKSIG(f, sigs, pubs, rests)` | f 为 Byte；k = 签名个数 = 公钥个数 ≥ 1；r 为补全集个数，可以为 0；k + r ≤ 255；否则失败。第 i 个签名用第 i 个公钥验证，不排序。即使某个签名不成功，也验证全部签名。全部成功时返回 true | 序位 0 到 k−1 为各自的结果；序位 k 到 k+r−1 为 false |
| `SYS_CHKPASS` | 需要 `chkpass.type` 为 1、`chkpass.flag`，以及见证条目 `<Sig>`、`<PubKey>`。BLAKE3-256(P) 等于 `IN{Receiver}` 且签名有效时成功；缺少任何一项或任一条件不成立即失败 | 成功时序位 0 为 true |

#### 6.5 宿主函数

| 指令 | 实参 | 结果 | 失败 |
| --- | --- | --- | --- |
| `FN_PUBHASH` | P，Bytes | `BLAKE3-256(P)` | P 不是 Bytes |
| `FN_MPUBHASH` | 哈希集（`[]Bytes` 或成员全为 Bytes 的 `[]any`）、m、N | `BLAKE3-256(u8 m ‖ u8 N ‖ 各哈希依次连接)` | m、N 不是整数类实参；不满足 1 ≤ m ≤ N ≤ 255；成员数不等于 N；有成员不是 32 字节 |
| `FN_ADDRESS` | 编码：Bytes x，String p | String：`p ‖ ":" ‖ x 的小写十六进制` | — |
| `FN_ADDRESS` | 解码：String a，Nil 或 String | Bytes：a 中最后一个 `:` 之后的部分按小写十六进制解码 | 没有 `:`；之后的部分不是偶数个小写十六进制数字 |

其它实参类型都是失败。测试宿主不实现 `SYS_AWARD`，对它而言这是未分配码位，含有它的段加载失败（SPEC-0008 第 1 条）。

#### 6.6 成本

记 S = 1000，h(x) = 20 + B(x)（BLAKE3，SPEC-0006 第 2.1 条），K 按 SPEC-0006 计算：

| 指令 | 成本 |
| --- | --- |
| `FN_CHECKSIG` | `K + S` |
| `FN_MCHECKSIG` | `K + (k + r) + k × S` |
| `SYS_CHKPASS` | `1 + S + h(P)` |
| `FN_PUBHASH` | `K + h(P)` |
| `FN_MPUBHASH` | `K + N + h(2 + 32N)` |
| `FN_ADDRESS` | `K + 10` |

这些数值只属于测试宿主，用来检验计费的时点与顺序。外部脚本加载的成本按核心的规定（SPEC-0006）。

#### 6.7 参数、本地输出与私有域

- `MaxInputScriptCost` 取 SPEC-0006 的默认值，向量可以用 `budget` 覆盖。测试宿主不累计交易与区块成本，也不做创建时的检查（这一检查由 `load` 向量覆盖）。
- 私有注册函数：`FN_PRIV[0]` 有 1 个实参、1 个返回值，原样返回实参；`EXT_PRIV[0, 0]`（目标 0、方法 0）有 2 个实参、1 个返回值，返回由两个实参组成的 `[]any`。其它索引没有注册。
- `BUFDUMP` 的处理器记录交给它的内容，供 `expect.private.dumps` 判定。
- `SHELL` 的来源输出数据依次为：`ENV{TxID}`、当前输入的序位（测试宿主只验证一个输入，固定为 Int 0）、`IN{Amount}`、`IN{Receiver}`、`IN{Memo}`。夹具中没有给出的字段为 Nil（第 6.2 条）。

### 7. Evidcoin 宿主向量

`host` 为 `evidcoin` 的文件按 SPEC-0901 配置：

- `fields`、`outputs`、`inouts` 使用 SPEC-0901 第 3 条的字段名；`witness` 使用第 4 条的条目名；`chkpass` 给出签名类别与授权标识；`sigmsg` 给出按 Evidcoin 交易规格构造的消息。
- `scripts` 的 `ref` 为 `[年度, "交易 ID 的 96 位十六进制", 输出序位]`。
- 依赖 SPEC-0901 中待决取值的向量，必须在 `pending` 中列出：签名算法与成本〔P-03〕、字段表与见证条目表〔P-08〕、版本取值〔P-09〕、兑奖规则〔P-10〕。

宿主向量的 `case`：

**`prerelease`**（SPEC-0901 第 2 条）

```json
"input": { "version": 2, "H": 1000, "num": 60, "den": 100,
           "blocks": [ { "coinbase": 32769, "txs": [32769, 1, 32769] }, … ] },
"expect": { "activated": true, "effective": 1005 }
```

- `blocks` 依次是高度 H 到 H+4 的 5 个区块。每个区块给出 Coinbase 与其余各交易头部的 2 字节版本字（十进制）。
- `expect.effective` 是生效高度，未生效时为 `null`。

**`create`**（SPEC-0901 第 9、10 条）

```json
"input": { "height": 1005, "active": [1, 2], "coinbase": false, "inputCost": 1200,
           "outputs": [ { "lock": "…", "versionField": 2 } ] },
"expect": { "valid": true, "txCost": 1234 }
```

- `active` 是在 `height` 已生效的核心版本。`coinbase` 为 true 时，各输出按数组下标作为序位，参与 `SYS_AWARD` 附参的检查。`inputCost` 是各输入验证成本之和。
- `expect.txCost` 在 `valid` 为 true 时必须给出，是 SPEC-0901 第 10 条的交易脚本成本。


## 边界与限制

- 向量只断言共识相关的结论与本篇列出的可观察输出，不断言实现的内部状态。
- 核心测试宿主的签名、哈希与地址是为测试设计的，不具备任何安全性。
- 成本断言的数值依赖 P-02，P-02 决定之前，它们检验的是计费的时点与顺序，而不是最终数值。


## 待决问题

- 成本断言的数值依赖 P-02（以文件的 `costs` 标识区分）。Evidcoin 的签名向量依赖 P-03，字段名依赖 P-08，版本取值依赖 P-09，`SYS_AWARD` 依赖 P-10。
- 本篇的细则：R-50、R-51（待确认），见 SPEC-0001 第 6.4 条。


## 对 Plan 的约束

- 向量由参考实现的测试导出（DEC-0001 第 5 条），随仓库发布。核心向量与宿主向量分文件存放，核心向量只用测试宿主。
- 每个向量用 `refs` 指向它所覆盖的规格条款，以便检查各篇「对 Plan 的约束」中列出的覆盖要求。
- 向量的读取与判定实现为独立的运行器，不依赖生成它们的测试代码；运行器对未知字段报错（第 1 条）。
- 导出时，执行向量与加载向量附带 `text` / `lockText` 等对照文本，由反编译器生成，便于审阅。
