# SPEC-0105 模块与扩展：v1 方法表

- 状态：草案，待作者审定
- 日期：2026-11-06
- 修订：2026-11-11（P-05、P-06 经第六轮回答确认）


## 来源追溯

- `conception/Instruction/17.模块指令.md`：静态索引与动态方法名两种调用方式；内置对象（Time、RegExp、BigInt）没有构造方法；多个返回值封装为切片；`MO_MATH` 的 1–7、`MO_FMT` 的 1–3 与示例（构想层原有）；其余方法为本轮回填（原为「待补充」）
- `conception/Instruction/14.工具指令.md`：`CALL` 及其示例（`Day`、`Clock`、`Format`、`Abs`、`Pow`、`Sprint`）
- `conception/Instruction/18.扩展指令.md`：`EXT_MO`
- DEC-0001 第 3 条（时间：公共域开放数值读取，只认 UTC 与固定偏移；`Format`、`Local`、`In` 私有）
- DEC-0004 第 2 条（方法表属于版本的取值表，随版本冻结）
- DEC-0005 开放问题（2026-11-06 已关闭：BigInt 的取模与幂由 `MO_BINT` 提供）
- SPEC-0002 第 6.16 条（模块指令的格式）、SPEC-0004 第 2 条（Time）、SPEC-0104 第 3.9 条（`CALL`）


## 概述

本篇给出 v1 的模块方法表：`MO_TIME`、`MO_RE`、`MO_BINT`、`MO_MATH`、`MO_FMT`，以及 `CALL` 作用于 Dict 时的方法。方法表属于版本的取值表〔P-05〕：静态调用的索引、`CALL` 可用的方法名，都按指令所在段的版本确定；未分配的索引在加载期即失败（SPEC-0007 第 4 条）。

构想层原标为「待补充」，作者要求由评审直接给出合适的结果，并已在第六轮回答中确认。方法名与顺序已回填到构想层 `17.模块指令.md`。成本系数仍属 P-02。


## 规格正文

### 1. 调用

#### 1.1 两种调用方式

| 方式 | 写法 | 规则 |
| --- | --- | --- |
| 静态 | `MO_X[i]`，i ≥ 1 | 按方法表中该方法的实参记号取得实参（SPEC-0005 第 4 条）；返回值按方法表 |
| 构造 | `MO_X[0]` | 仅 `MO_MATH`、`MO_FMT`：实参 0，返回一个模块对象 |
| 动态 | `CALL` | 目标与方法名在运行时给出（SPEC-0104 第 3.9 条） |

- `MO_TIME`、`MO_RE`、`MO_BINT` 的索引 0 未分配〔P-06〕：使用即加载失败。它们的「对象」就是 Time、RegExp、BigInt 值本身，`CALL` 以这些值为目标（构想层）。
- 方法返回多个值时，封装为一个切片作为单个返回值（构想层约定）。静态调用与 `CALL` 相同。
- 方法名区分大小写。方法表中的名称就是 `CALL` 使用的方法名，也是文本中静态调用的别名（`MO_MATH{Abs}`，SPEC-0201）。

#### 1.2 `CALL` 的方法集

| `CALL` 的目标 | 方法集 | 目标是否作为方法的实参 1 |
| --- | --- | --- |
| Time | `MO_TIME` | 是 |
| RegExp | `MO_RE` | 是 |
| BigInt | `MO_BINT` | 是 |
| `MO_MATH[0]` 创建的模块对象 | `MO_MATH` | 否 |
| `MO_FMT[0]` 创建的模块对象 | `MO_FMT` | 否 |
| Dict | 第 7 条 | 否（作为接收者） |

例（构想层）：`DATE{2026-03-21T19:08:50Z} TOP "Day" &CALL[2]` 调用 `Day(t)`，得 21；`MO_MATH "Abs" {-100} CALL[3]` 调用 `Abs(-100)`，得 100。

#### 1.3 表的写法

- **实参**：`t` 为 Time；`re` 为 RegExp；`s` 为 String 或 Bytes；`x` 为 BigInt；`I` 为整数类或 BigInt，整数类先提升为 BigInt；`i`、`n` 为整数类（SPEC-0101 第 1.2 条）；`T` 见第 5 条。
- **记号**：方法的实参记号。未写的为固定个数，即签名中的实参个数；`-1` 等变体的含义同 SPEC-0002 第 4 条。
- **P**：私有方法。公共执行中调用即失败（静态调用在执行到时失败，与私有专属指令相同）。
- **成本**：按 SPEC-0006 第 2.1 条的记号，基数为 `K`，再加所属类别的附加项（SPEC-0006 第 3 条）。静态调用中 a 为方法的实参数；`CALL` 中 a 为 `CALL` 取得的全部实参数（含目标与方法名），见 SPEC-0104 第 3.9 条。
- 每个方法另有本表之外的失败情形时，在「说明」中写出；实参类型不符一律失败。

### 2. `MO_TIME`（221）

所有方法的实参 1 是 Time。

公私划分〔R-33〕：DEC-0001 第 3 条在公共域开放年、月、日、时、分、秒、毫秒与 Unix 时间戳的数值读取，`Format`、`Local`、`In` 为私有。本表 1–14 都是数值读取；15–19 是固定偏移的换算与时刻运算，超出该条的列举，但只做整数运算，不依赖格式化与时区数据库，结果确定，符合该条划分公私的理由，因此也列为公共方法。

| 索引 | 方法 | 返回 | 成本 | 说明 |
| --- | --- | --- | --- | --- |
| 1 | `Year(t)` | Int | `K` | 年，可以为 0 或负数（天文纪年） |
| 2 | `Month(t)` | Int | `K` | 1–12 |
| 3 | `Day(t)` | Int | `K` | 1–31 |
| 4 | `Hour(t)` | Int | `K` | 0–23 |
| 5 | `Minute(t)` | Int | `K` | 0–59 |
| 6 | `Second(t)` | Int | `K` | 0–59 |
| 7 | `Millisecond(t)` | Int | `K` | 0–999 |
| 8 | `WeekDay(t)` | Int | `K` | 0–6，0 为星期日 |
| 9 | `YearDay(t)` | Int | `K` | 1–366 |
| 10 | `Date(t)` | `[]Int` | `K + m` | [年, 月, 日] |
| 11 | `Clock(t)` | `[]Int` | `K + m` | [时, 分, 秒] |
| 12 | `Unix(t)` | Int | `K` | ⌊时刻 / 1000⌋，秒 |
| 13 | `UnixMilli(t)` | Int | `K` | 时刻，毫秒 |
| 14 | `Offset(t)` | Int | `K` | 偏移，秒 |
| 15 | `UTC(t)` | Time | `K` | 时刻不变，偏移为 0 |
| 16 | `InOffset(t, i)` | Time | `K` | 时刻不变，偏移为 i 秒。\|i\| < 86400 且为 60 的倍数，否则失败 |
| 17 | `AddMilli(t, i)` | Time | `K` | 时刻加 i 毫秒，偏移不变；溢出即失败 |
| 18 | `AddDate(t, i, i, i)` | Time | `K` | 加年、月、日，见 2.2 |
| 19 | `Sub(t, t)` | Int | `K` | 实参 1 的时刻减实参 2 的时刻，毫秒；溢出即失败 |
| 20 | `Format(t, String)` P | String | `K` | 按 Go `time.Time.Format` 的布局 |
| 21 | `Local(t)` P | Time | `K` | 转为本地时区（节点设置） |
| 22 | `In(t, String)` P | Time | `K` | 转为 IANA 时区名所指的时区（节点的时区数据库） |

#### 2.1 日历计算

日历取值（1–11）都按 Time 的偏移计算（SPEC-0004 第 2 条〔R-05〕）：

```
L      = 时刻 + 偏移 × 1000            // 本地毫秒；溢出即失败
days   = ⌊L / 86400000⌋                 // 自 1970-01-01 起的本地日数
msDay  = L − days × 86400000            // 0 ～ 86399999
年月日 = 由 days 按外推格里历（proleptic Gregorian）换算
时、分、秒、毫秒 = 由 msDay 换算
WeekDay = (days + 4) mod 7              // 1970-01-01 是星期四；mod 取非负余数
YearDay = days − (该年 1 月 1 日的 days) + 1
```

- 外推格里历：闰年为能被 4 整除且不能被 100 整除，或能被 400 整除的年份，对公元 1582 年以前与 0 年、负数年同样适用。
- `Unix` 向负无穷取整：时刻 −1 毫秒的 `Unix` 为 −1。
- 不使用时区数据库，不处理闰秒（DEC-0001 第 3 条）。

#### 2.2 `AddDate(t, y, m, d)`

与 Go `time.Time.AddDate` 相同，在本地日历上计算：

1. 取 t 的本地年 Y、月 M、日 D 与 msDay（2.1）。
2. M₀ = (M − 1) + m；Y′ = Y + y + ⌊M₀ / 12⌋；M′ = (M₀ mod 12) + 1。
3. 本地日数 = (Y′ 年 M′ 月 1 日的 days) + (D − 1) + d。日超出该月时顺延，例如 10 月 31 日加 1 个月得 12 月 1 日。
4. 结果的时刻 = 本地日数 × 86400000 + msDay − 偏移 × 1000，偏移不变。

任一步溢出 int64 即失败。

### 3. `MO_RE`（222）

所有方法的实参 1 是 RegExp。查找规则与全部匹配的算法见 SPEC-0104 第 1 条。实参 `s` 为 String 时，结果中的文本为 String；为 Bytes 时为 Bytes。M 为一次正则匹配的成本 ⌈s 的字节数 × 正则字节数 / 8⌉。

| 索引 | 方法 | 返回 | 成本 | 说明 |
| --- | --- | --- | --- | --- |
| 1 | `Match(re, s)` | Bool | `K + M` | 是否存在匹配 |
| 2 | `Find(re, s)` | String / Bytes 或 Nil | `K + M` | 首个匹配；没有匹配为 Nil |
| 3 | `FindIndex(re, s)` | `[]Int` 或 Nil | `K + M` | [起点, 终点)，字节偏移 |
| 4 | `FindSubmatch(re, s)` | `[]any` 或 Nil | `K + m + M` | 全匹配与各组；未参与匹配的组为 Nil |
| 5 | `FindSubmatchIndex(re, s)` | `[]Int` 或 Nil | `K + m + M` | 每组两个偏移；未参与匹配的组为 −1、−1 |
| 6 | `FindAll(re, s, n)` | `[]String` / `[]Bytes` | `K + m + M` | 全部匹配的全文 |
| 7 | `FindAllIndex(re, s, n)` | `[]any` | `K + m + M` | 每个成员为一个 `[]Int` |
| 8 | `FindAllSubmatch(re, s, n)` | `[]any` | `K + m + M` | 每个成员同 `FindSubmatch` |
| 9 | `FindAllSubmatchIndex(re, s, n)` | `[]any` | `K + m + M` | 每个成员同 `FindSubmatchIndex` |
| 10 | `ReplaceAll(re, s, s)` | String / Bytes | `K + B(s) + M` | 替换全部匹配；实参 3 为模板，展开规则同 `REPLACE`（SPEC-0104 第 3.6 条） |
| 11 | `ReplaceAllLiteral(re, s, s)` | String / Bytes | `K + B(s) + M` | 同上，实参 3 按原文 |
| 12 | `NumSubexp(re)` | Int | `K` | 捕获组的个数 |
| 13 | `SubexpNames(re)` | `[]String` | `K + m` | 第 0 个为 `""`，未命名的组为 `""` |
| 14 | `SubexpIndex(re, String)` | Int | `K` | 该名称的组号；不存在为 −1 |
| 15 | `String(re)` | String | `K` | 源文本 |

- `FindAll*` 的 n 为整数类：n < 0 表示全部，n = 0 得到空切片，n > 0 至多取 n 个。结果超过 1024 个成员即失败（SPEC-0006 第 1 条）；实现可以在找到第 1025 个时停止。
- `ReplaceAll`、`ReplaceAllLiteral` 的实参 3 与实参 2 的类型必须相同。
- 不提供 `Split`、`Longest` 等改变匹配语义或结果形态难以统一的方法。

### 4. `MO_BINT`（223）

所有方法的实参 1 是 BigInt（整数类即失败，先用 `BIGINT` 转换）。`I` 类实参接受整数类与 BigInt。结果为 BigInt 时，绝对值达到 2¹⁰¹⁶ 即失败。w 为操作数与结果中最大者的 64 位字数（1–16）。

| 索引 | 方法 | 返回 | 成本 | 说明 |
| --- | --- | --- | --- | --- |
| 1 | `Abs(x)` | BigInt | `K + w` | 绝对值 |
| 2 | `Sign(x)` | Int | `K` | −1、0、1 |
| 3 | `Cmp(x, I)` | Int | `K + w` | x < y 为 −1，相等为 0，大于为 1 |
| 4 | `Quo(x, I)` | BigInt | `K + w²` | 商，向零截断；y = 0 即失败 |
| 5 | `Rem(x, I)` | BigInt | `K + w²` | 截断除法的余数，与 x 同号 |
| 6 | `QuoRem(x, I)` | `[]BigInt` | `K + m + w²` | [Quo, Rem] |
| 7 | `Div(x, I)` | BigInt | `K + w²` | 欧几里得除法的商：使余数满足 0 ≤ r < \|y\| |
| 8 | `Mod(x, I)` | BigInt | `K + w²` | 欧几里得余数，0 ≤ r < \|y\| |
| 9 | `DivMod(x, I)` | `[]BigInt` | `K + m + w²` | [Div, Mod] |
| 10 | `Pow(x, i)` | BigInt | `K + ⌈w² × bitlen(e) / 8⌉` | xᵉ，e ≥ 0；x⁰ = 1（含 0⁰） |
| 11 | `ModPow(x, I, I)` | BigInt | `K + ⌈w² × bitlen(e) / 8⌉` | xᵉ mod m，e ≥ 0，m ≥ 1；结果在 [0, m) |
| 12 | `ModInverse(x, I)` | BigInt | `K + 4w²` | 模 m 的逆元，m ≥ 1；结果在 [0, m)；gcd(x, m) ≠ 1 即失败 |
| 13 | `GCD(x, I)` | BigInt | `K + 4w²` | 非负的最大公约数；gcd(0, 0) = 0 |
| 14 | `Sqrt(x)` | BigInt | `K + 10w²` | ⌊√x⌋；x < 0 即失败 |
| 15 | `BitLen(x)` | Int | `K` | \|x\| 的位数；0 为 0 |
| 16 | `Bit(x, i)` | Int | `K` | x 的二进制补码表示中第 i 位（0 或 1），i ≥ 0 |
| 17 | `IsInt64(x)` | Bool | `K` | 是否在 int64 范围内 |
| 18 | `FillBytes(x, i)` | Bytes | `K` | \|x\| 的大端表示，左侧补零到 i 字节；放不下即失败；0 ≤ i ≤ 8191 |

- 「欧几里得除法」与 Go `big.Int` 的 `Div`、`Mod`、`DivMod` 相同；「截断除法」与 `Quo`、`Rem`、`QuoRem` 相同。
- `Pow`：e 为整数类且 ≥ 0。|x| ≥ 2 时，应在计算之前按 (bitlen(|x|) − 1) × e ≥ 1016 判断结果必然超限并失败。
- `ModPow`、`ModInverse` 的 m 为模数；`ModPow` 中 m = 1 的结果为 0。
- BigInt 的加、减、乘、截断除法、取负、位运算与移位由运算指令提供（SPEC-0102 第 3 条），转回 Int 用 `INT`（SPEC-0102 第 2.1 条）。本模块补齐 BigInt 的取模与幂（DEC-0005 开放问题，已关闭）。

### 5. `MO_MATH`（224）

T 为 Int 或 Float（私有）；Byte、Rune 先提升为 Int；BigInt 即失败。返回值的类型与实参相同（构想层）；两个实参分别为 Int 与 Float 时，Int 先就近舍入为 Float，结果为 Float。

| 索引 | 方法 | 返回 | 成本 | 说明 |
| --- | --- | --- | --- | --- |
| 0 | 构造 | 模块对象 | `K` | |
| 1 | `Abs(T)` | T | `K` | Int：`Abs(MinInt64)` 即失败；Float：同 Go `math.Abs` |
| 2 | `Ceil(T)` | T | `K` | Int 为原值；Float：同 `math.Ceil` |
| 3 | `Floor(T)` | T | `K` | Int 为原值；Float：同 `math.Floor` |
| 4 | `Pow(T, T)` | T | `K + 2` | Int：同 `POW`（指数为负即失败，0⁰ = 1，溢出即失败）；Float：同 `math.Pow` |
| 5 | `Max(T, T)` | T | `K` | Float：同 `math.Max`（有 NaN 即 NaN） |
| 6 | `Min(T, T)` | T | `K` | Float：同 `math.Min` |
| 7 | `Mod(T, T)` | T | `K` | 同 `MOD`：Int 余数与被除数同号，除数为 0 即失败；Float：同 `math.Mod` |
| 8 | `Round(T)` | T | `K` | Int 为原值；Float：四舍五入，0.5 远离零（同 `math.Round`） |
| 9 | `Trunc(T)` | T | `K` | Int 为原值；Float：同 `math.Trunc` |
| 10 | `Sign(T)` | T | `K` | Int：−1、0、1；Float：−1.0、1.0，±0 与 NaN 返回原值 |
| 11 | `Sqrt(T)` | T | `K + 2` | Int：⌊√x⌋，x < 0 即失败；Float：同 `math.Sqrt` |
| 12 | `MulDiv(Int, Int, Int)` | Int | `K + 1` | a × b / c：乘积按 128 位精确计算，商向零截断；c = 0 或结果超出 int64 即失败 |
| 13 | `Exp(T)` P | Float | `K + 2` | 实参为 Int 时先转为 Float；下同 |
| 14 | `Log(T)` P | Float | `K + 2` | 自然对数 |
| 15 | `Log2(T)` P | Float | `K + 2` | |
| 16 | `Log10(T)` P | Float | `K + 2` | |
| 17 | `Sin(T)` P | Float | `K + 2` | |
| 18 | `Cos(T)` P | Float | `K + 2` | |
| 19 | `Tan(T)` P | Float | `K + 2` | |
| 20 | `Atan(T)` P | Float | `K + 2` | |
| 21 | `Atan2(T, T)` P | Float | `K + 2` | 实参依次为 y、x |
| 22 | `Hypot(T, T)` P | Float | `K + 2` | |

- 索引 1–7 是构想层原有的方法与顺序（示例假设 `Abs` 的索引为 1），8 起为本轮增补。
- 构想层示例 `{1} {99.0} MO_MATH{Max}` 得 99.0，这是私有执行中的 Float；公共执行中 `MO_MATH` 只有 Int 形式可用。
- `MulDiv` 用于金额按比例计算（如 a × 分子 / 分母）而不经过 BigInt。
- 13–22 的结果总是 Float，因此标为私有；它们的结果依赖浮点数学库的实现，不属于共识。

### 6. `MO_FMT`（250）

整个模块是私有的〔R-10〕：构造方法与全部静态方法在公共执行中执行到即失败。

| 索引 | 方法 | 记号 | 返回 | 成本 | 说明 |
| --- | --- | --- | --- | --- | --- |
| 0 | 构造 P | 0 | 模块对象 | `K` | |
| 1 | `Sprint(any...)` P | `-1` | String | `K` | 同 Go `fmt.Sprint` |
| 2 | `Sprintf(String, any...)` P | `-1≥1` | String | `K` | 同 Go `fmt.Sprintf`；实参 1 为格式串 |
| 3 | `Sprintln(any...)` P | `-1` | String | `K` | 同 Go `fmt.Sprintln` |

值传给 Go `fmt` 时的对应关系，以及默认的文本形式，见 SPEC-0106 第 7 条。

### 7. `CALL` 作用于 Dict

| 方法 | 返回 | 成本 | 说明 |
| --- | --- | --- | --- |
| `Has(String)` | Bool | `K` | 是否含有该键 |
| `Get(String, any)` | any | `K` | 该键的值；不存在时返回实参 2（默认值） |

- 这两个方法只能经 `CALL` 调用，没有静态索引。Dict 本身是接收者，不计为方法的实参：`dict "Get" "k" NIL CALL[4]`。
- 读取不存在的键得 Nil 用 `ITEM` 即可（SPEC-0101 第 5.8 条）；`Get` 用于需要区分「不存在」与「值为 Nil」，或需要其它默认值的场合。

### 8. `EXT_MO`（251）

v1 没有已分配的扩展模块，任何使用在加载期即失败。扩展模块的分配随新版本发布（SPEC-0007）。


## 边界与限制

- 方法表随版本冻结。新版本可以在未分配的索引上增加方法，不能改变已分配方法的语义（SPEC-0007 第 3 条）。
- `Format`、`Local`、`In` 依赖 Go 的布局语法或节点的时区数据库，因此是私有方法。公共执行中需要的日期文本，可以由取值方法与 `STRING` 组合得到。
- Float 方法的结果依赖实现的浮点数学库，私有执行中不同实现的结果可能有末位差异。


## 待决问题

- P-05、P-06 已关闭（SPEC-0001 第 6.1 条）。成本系数仍是 P-02。
- P-02：成本系数。
- 本篇涉及的细则：R-10、R-33（已确认），见 SPEC-0001 第 6.2、6.3 条。


## 对 Plan 的约束

- 方法表实现为数据表（模块 → 索引 → 名称、实参记号与类型、返回、私有标记、成本函数），静态调用、`CALL`、反编译器与文本别名共用这一张表，按版本选取。
- `MO_TIME` 的日历计算按第 2.1 条自行实现（或使用 Go `time` 包中不涉及时区数据库的部分，并以向量验证），覆盖负时刻、0 年与负数年、闰年 2 月 29 日加 1 年、月末加月等情形。
- `MO_BINT` 的每个方法都要有正、负、零与边界（2¹⁰¹⁶ − 1）的向量；`Pow`、`ModPow` 的超限要在计算之前判定。
- 构想层 `14.工具指令.md` 与 `17.模块指令.md` 的全部示例都要成为测试向量（私有方法的示例在私有执行中验证）。
