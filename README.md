# Escript 精简但准图灵完备的栈脚本

> 本项目为从区块链 [Evidcoin](https://github.com/cxio/Evidcoin) 中剥离出来，方便通用化使用。

支持简单的合约逻辑。


## 示例

定制的单签名币金支付（不使用 `SYS_CHKPASS` 内置验证）。

```go
// 解锁脚本：
VALUE{OutType}      // 输出类型：1-币金；2-凭信
<flag>              // 授权标识
<sig>               // 签名入栈
<pubKey>            // 公钥入栈

// 锁定脚本：
TOP                 // 引用栈顶项（不弹出）返回入栈
FN_PUBHASH          // 取栈顶项计算公钥哈希后入栈
DATA{46af3fb4...}   // 预置的公钥哈希序列入栈
EQUAL               // 取栈顶2项相等比较，结果（true|false）入栈
PASS                // 取栈顶项检查，TRUE时则通过，否则失败
FN_CHECKSIG         // 取栈顶4项（即最初的 <out_type>, <flag>, <sig>, <pubKey>），验证签名。结果（true|false）入栈
PASS                // 取栈顶项检查，TRUE时则通过，否则失败
```

**解释：**

- `VALUE{OutType}`：从环境取当前输出所属类别（币金|凭信）。结果入栈。**栈状态**：`[<out_type>]`
- `<flag>`：授权标识入栈。**栈状态**：`[<out_type>, <flag>]`
- `<sig>`：自动入栈签名。**栈状态**：`[<out_type>, <flag>, <sig>]`
- `<pubKey>`：自动入栈公钥。**栈状态**：`[<out_type>, <flag>, <sig>, <pubKey>]`
- `TOP`：取栈顶项（`<pubKey>`）返回入栈。**栈状态**：`[<out_type>, <flag>, <sig>, <pubKey>, <pubKey>]`
- `FN_PUBHASH`：取栈顶项（`<pubKey>`）计算公钥哈希后入栈。**栈状态**：`[<out_type>, <flag>, <sig>, <pubKey>, <pubHash>]`
- `DATA{46af3fb4...}`：预置的公钥哈希序列入栈。**栈状态**：`[<out_type>, <flag>, <sig>, <pubKey>, <pubHash>, <preHash>]`
- `EQUAL`：取栈顶2项（`<preHash>`、`<pubHash>`）相等比较，结果（true|false）入栈。**栈状态**：`[<out_type>, <flag>, <sig>, <pubKey>, <true|false>]`
- `PASS`：取栈顶项检查，TRUE时则通过，否则失败。**栈状态**：`[<out_type>, <flag>, <sig>, <pubKey>]`
- `FN_CHECKSIG`：取栈顶4项（当前全部），验证签名。结果（true|false）入栈。**栈状态**：`[<true|false>]`
- `PASS`：取栈顶项检查，TRUE时则通过，否则失败。**栈状态**：`[]`


## 用法

<!-- 待补充 -->