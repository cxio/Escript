# DEC-0002 核心 VM 与宿主接口

- 状态：Accepted
- 日期：2026-10-28
- 修订：2026-11-01（并入作者回答；补公钥哈希与地址函数的归属、核心提供的环境字段）
- 覆盖评审条目：P2-1、P1-6（另含 P1-5 的宿主侧要求、Q7 的结论）


## 背景

README 把 Escript 定位为「从 Evidcoin 剥离出来，方便通用化使用」。但指令集里有不少 Evidcoin 专有的概念：币金 / 凭信 / 存证、`SYS_AWARD` 兑奖槽、`SYS_CHKPASS` 内置验证、输入输出的字段（Creator、Title、Attachment 等）、「年度 + 48 字节交易 ID + 序位」的外部引用格式、每天 240 个区块、公钥哈希与地址的算法等。

如果不先划出边界，Spec 就只能写成「Evidcoin 脚本规格」，其它宿主无法复用；核心逻辑的测试也会被宿主细节缠住。


## 决策

### 1. 拆成核心 VM 与宿主两层

**核心 VM** 是纯粹的执行引擎：输入「字节码 + 版本 + 宿主提供的环境 + 成本预算」，输出「结果 + 实际成本 + 执行标记」。内部没有 I/O，不知道 Evidcoin 的存在。

| 归属 | 内容 |
| --- | --- |
| 核心 | 值类型与编码；解码与加载期校验；数据栈、实参区、局部域与全局域；控制流与表达式；运算、比较、逻辑、转换；集合；MODEL；EVAL；通关状态机；校验域记录（`SIGNED` 读取的验证记录）；成本计量与限额；版本门控；系统占位符的执行（压入见证条目）；公共 / 私有两种执行模式；`ENV` 中描述执行状态的字段（`Passed`、`InGoto`、`InEmbed`、`InScript`、`InCode`、`Gotos`、`Embed1`–`Embed4`，由核心计算，索引区间由 Spec 划分） |
| 宿主定义 | 环境字段表（`GVAL`、`ENV` 的宿主字段、`IN`、`OUT`、`INOUT`、`XFROM` 的字段与取值）；外部脚本的引用格式与取数（`GOTO`、`EMBED`、`SCRIPT`）；签名验证算法与签名消息的构造（`FN_CHECKSIG`、`FN_MCHECKSIG`、`SYS_CHKPASS`）；公钥哈希与地址方案（`FN_PUBHASH`、`FN_MPUBHASH`、`FN_ADDRESS`）；见证条目表（系统占位符的附参所指的 `<Sig>`、`<PubKey>`、`<Sigs>`、`<PubKeys>`、`<Rests>`）；宿主专属的系统指令（`SYS_AWARD`）；私有域通道（`INPUT`、`OUTPUT`、`BUFDUMP`、`PRINT`、`SYS_TIME`、`SHELL`、`*_PRIV`）；各项参数的取值（区块成本上限、描述长度等） |

指令码的编号不变。规格中把一部分指令的语义标为「由宿主定义」：核心规格写明它们的附参、实参个数、返回值个数和失败条件，算法与取值由宿主规格给出。宿主没有实现的宿主指令（例如一个没有兑奖机制的宿主中的 `SYS_AWARD`），对该宿主而言就是未分配码位，按 DEC-0004 处理。

### 2. Evidcoin 作为第一个宿主规格（Host Profile）

Evidcoin 宿主规格负责：

- 上述各张字段表，以及 `OutKind` 等取值的含义；
- 外部引用格式「年度 + 48 字节交易 ID + 输出序位」，以及引用目标的取数与默克尔证明验证；
- 签名算法、签名消息的授权种类（`SIGIN_*`、`SIGOUT_*` 等）、见证信息的结构；
- 公钥哈希与地址方案：单签公钥哈希 `SHA3-256(BLAKE2b-512(pubKey))`、多签复合公钥哈希（`BLAKE3` 初级哈希、排序串联、前置 `m/N`）、地址的前缀、Base58 编码与校验码；
- 见证条目表：`<Sig>`、`<PubKey>` 对应 `FN_CHECKSIG`，`<Sigs>`、`<PubKeys>`、`<Rests>` 对应 `FN_MCHECKSIG`，附参取值与条目类型由宿主规格给出；
- `SYS_CHKPASS` 的完整语义（花费者地址核实、单签 / 多签）、`SYS_AWARD`；
- 交易的前置检查（币金 / 凭信的区别对待）。它在脚本执行之前完成，不属于脚本系统；
- 参数：每天 240 个区块、`MaxBlockScriptCost`、各签名算法的成本（见 DEC-0007）。

**授权标识。** 没有授权标识的占位符。标准验证中，授权标识随见证信息导入环境；定制验证中，它作为普通值写在解锁脚本里，由锁定脚本经 `FN_CHECKSIG` / `FN_MCHECKSIG` 的实参传给宿主。两种情形下，签名消息都包含授权标识本身（`Evidcoin设计参考.md`「签名消息」）。

### 3. 接口的形态

接口只承载粗粒度、低频的操作。高频的环境取值在执行前由宿主一次性填好，按附参下标直接读取。

```go
// 示意，非规范。
type Host interface {
    LoadScript(ref []byte) (code []byte, ver uint8, err error)  // GOTO / EMBED / SCRIPT；取不到时返回 ErrUnavailable
    VerifySig(req SigRequest) (SigResult, error)                // FN_CHECKSIG / FN_MCHECKSIG / SYS_CHKPASS；结果含各序位的验证状态
    SysCall(op byte, aux []byte, args []Value) ([]Value, error) // SYS_AWARD、FN_PUBHASH / FN_MPUBHASH / FN_ADDRESS 等宿主定义的指令
}

type Env struct {
    GVal, Env, In, XFrom [256]Value // 按附参索引取值
    Outs, InOuts         []OutputView
    Witness              []Value    // 系统占位符按附参取用
}
```

`GOTO`、`EMBED`、`SCRIPT` 的目标都写在附参里，是静态的。宿主可以在执行前扫描字节码，预先取齐全部外部脚本。

### 4. 执行结果有三种

| 结果 | 含义 | 宿主的处理 |
| --- | --- | --- |
| 通过 | 验证通过 | 收录 |
| 失败 | 验证不通过 | 拒绝；可以降低转发者的信任 |
| 数据不可用 | 外部脚本或环境数据取不到 | 稍后重试；不丢弃交易，不惩罚转发者 |

「数据不可用」不等于失败（P1-6）。没有数据的节点不会把这类交易打包进区块，区块同步后数据自然齐全。

### 5. 执行标记

核心 VM 在结果中附带执行标记，供宿主使用。目前一项：**本次执行是否读取过 `Height`、`BlockTime`、`TipHeight`**（P1-5）。带这个标记的交易，其有效性会随高度变化，宿主的内存池在每个新区块之后只需重验这类交易。

### 6. 宿主必须遵守的私有域要求

- 私有节点默认关闭 `SHELL`，必须显式配置白名单（例如按脚本哈希，或按签名者授权）才执行。任何人都能发一笔交易让私有节点运行其中的命令，这是强制要求，不是建议。
- 私有域通道的数据只影响本地，不参与共识。


## 理由

- **性能影响可以忽略。** 热路径（栈操作、运算、控制流、MODEL）全部在核心内部，不经过接口。环境字段是数组访问。只有加载外部脚本、签名验证、系统调用走接口：Go 的接口调用是纳秒级，一次签名验证是几十到上百微秒，接口开销不到千分之一。
- **可测试。** 核心可以单独生成测试向量（DEC-0001），宿主的变化不影响核心的向量集。
- **通用化。** 换一个宿主只需要写一份新的宿主规格，核心不变。公钥哈希与地址方案各链不同（Bitcoin 用 HASH160 与 Base58Check），因此与签名算法一起归宿主。

**不做：** 不为不同宿主分配不同的指令码区间。码位统一，语义由宿主规格填充，这样反编译器和 MODEL 不需要知道宿主是谁。


## 影响

- Spec 分成两部分：「核心规格」和「Evidcoin 宿主规格」。核心规格中由宿主定义的指令只写接口契约（输入、输出、失败条件、成本由谁给出）。
- Plan 的包边界按此划分，例如核心包不得依赖 Evidcoin 的任何类型。
- TxID 的确定性由宿主保证：Evidcoin 的 TxID 既不含见证信息，也不含解锁脚本（Q7）。因此解锁区保留 `[0-45]` 的灵活性不会造成交易延展性，见 DEC-0003。


## 构想层依据

- `README.md`：通用化定位
- `conception/Instruction/AGENTS.md`：历史交易的携带与缓存
- `conception/Instruction/13.环境指令.md`：`ENV`、`IN`、`OUT`、`INOUT`、`XFROM` 字段
- `conception/Instruction/14.工具指令.md`：`SHELL`
- `conception/Instruction/15.系统指令.md`：`SYS_AWARD`、`SYS_CHKPASS`
- `conception/Instruction/16.函数指令.md`：`FN_ADDRESS`、`FN_PUBHASH`、`FN_MPUBHASH`
- `conception/Design.md`：附：系统占位符；附：`SYS_CHKPASS` 超级赎回
- `working/Evidcoin设计参考.md`：签名、签名消息、交易的前置检查、输出项配置
- 评审问答：P2-1、P1-6、P1-5、Q7，第四轮


## 开放问题

1. **超级赎回**（`Design.md` 附录）不进入本 Decision（作者已同意）。如果将来采纳，它属于 Evidcoin 宿主规格中 `SYS_CHKPASS` 的语义。它让原本无法花费的输出变得可以花费，属于放宽规则，是硬分叉，不适用 DEC-0004 的软分叉路径。

已关闭（2026-11-01）：见证条目表的名称已在构想层定为 `<Sig>`、`<PubKey>`、`<Sigs>`、`<PubKeys>`、`<Rests>`，附参取值与类型由宿主规格给出；授权标识不设占位符，签名消息包含授权标识（第 2 条）。
