# Revm apparent 字段解释
_Exported on 1/22/2026 at 23:34:54 GMT+8 from Cursor (2.3.29)_

---

**User**

@crates/interpreter/src/interpreter_action/call_inputs.rs:1-250 这是revm的代码。其中的apparent是啥啊

---

**Cursor**

看这段 revm 代码，我来解释 `apparent` 的含义：

## Apparent 是什么？

`apparent` 是一个用于 **DELEGATECALL** 场景的特殊概念。它表示一个"表观值"（apparent value），而不是实际转移的值。

### 关键点：

1. **CallValue 枚举** (第 195-202 行)：
   - `Transfer(U256)` - 实际从调用者转移给被调用者的值
   - `Apparent(U256)` - 只是"看起来"的值，**不会实际转移**

2. **为什么存在 Apparent？**
   
   在 **DELEGATECALL** 中：
   - 被调用的代码在调用者的上下文中执行（共享存储、调用者地址等）
   - 当被调用代码执行 `CALLVALUE` 操作码时，它需要获取原始调用的 value
   - 但这个 value **不会真正转移**给任何账户
   - 这就是 `Apparent` 的用处

3. **使用示例**：

```rust
// 如果是普通 CALL，value 会被转移
CallValue::Transfer(U256::from(100))

// 如果是 DELEGATECALL，value 是"表观的"，不转移
CallValue::Apparent(U256::from(100))
```

4. **相关方法**：
   - `transfer()` - 只在 `Transfer` 时返回 `Some`
   - `apparent()` - 只在 `Apparent` 时返回 `Some`
   - `is_transfer()` / `is_apparent()` - 判断类型
   - `get()` - 无论类型如何，都返回实际的数值

### 简单理解：

**Apparent value = DELEGATECALL 中的虚拟金额，用来欺骗被调用合约让它"以为"有价值转移，但实际上没有钱动过。**

```rust
// 如果是普通 CALL，value 会被转移
CallValue::Transfer(U256::from(100))

// 如果是 DELEGATECALL，value 是"表观的"，不转移
CallValue::Apparent(U256::from(100))
```

---

**User**

举个简单例子来说明

---

**Cursor**

好，我用一个具体的例子来说明 `apparent` 和 `transfer` 的区别：

## 场景演示

假设现在有三个账户：
- **账户 A**（余额：100 ETH）
- **账户 B**（合约，余额：10 ETH）
- **账户 C**（合约）

### 场景 1：普通 CALL（使用 Transfer）

```
A 调用 B，传送 5 ETH

CallValue::Transfer(5 ETH)

结果：
- A 的余额：95 ETH（失去 5 ETH）
- B 的余额：15 ETH（获得 5 ETH）
- 5 ETH 真实转移了
```

### 场景 2：DELEGATECALL（使用 Apparent）

```
A 调用 B，B 的代码中使用 DELEGATECALL 调用 C，并传送 5 ETH

B 在调用 C 时：
CallValue::Apparent(5 ETH)

结果：
- A 的余额：95 ETH（失去 5 ETH）
- B 的余额：10 ETH（没变化）
- C 的余额：不变（没获得任何 ETH）
- C 的代码中执行 CALLVALUE 能读到 5 ETH
- 但这 5 ETH 其实没有真正转移给 C，只是"幻想"出来的
```

## 为什么要这样设计？

DELEGATECALL 的特点是被调用代码在**调用者的上下文**中执行，包括调用者的存储、余额等。所以：

- **Transfer**：真实金钱流动，用于 CALL、CALLCODE 等正常调用
- **Apparent**：虚拟金额，让被调用代码"感觉"有价值转移，但实际账户余额不动，仅用于 DELEGATECALL

这样 DELEGATECALL 中的代码执行 `CALLVALUE` 操码时能得到正确的"原始调用值"，同时不会真的转移资金。

```plaintext
A 调用 B，传送 5 ETH

CallValue::Transfer(5 ETH)

结果：
- A 的余额：95 ETH（失去 5 ETH）
- B 的余额：15 ETH（获得 5 ETH）
- 5 ETH 真实转移了
```

```plaintext
A 调用 B，B 的代码中使用 DELEGATECALL 调用 C，并传送 5 ETH

B 在调用 C 时：
CallValue::Apparent(5 ETH)

结果：
- A 的余额：95 ETH（失去 5 ETH）
- B 的余额：10 ETH（没变化）
- C 的余额：不变（没获得任何 ETH）
- C 的代码中执行 CALLVALUE 能读到 5 ETH
- 但这 5 ETH 其实没有真正转移给 C，只是"幻想"出来的
```

---

**User**

不对啊，你场景2中，整体来看减少了5ETH，应该守恒才对啊

---

**Cursor**

你说得完全对！我之前的例子有问题。让我重新来：

## 更正后的场景 2：DELEGATECALL（使用 Apparent）

```
A 调用 B，发送 5 ETH
B 的代码中使用 DELEGATECALL 调用 C，传送 5 ETH

结果：
- A 的余额：95 ETH（失去 5 ETH）
- B 的余额：15 ETH（获得了 A 转来的 5 ETH）
- C 的余额：不变（因为 DELEGATECALL 中 C 是代理执行，代码在 B 的上下文运行）
- C 的代码中执行 CALLVALUE 能读到 5 ETH
- 总金额：95 + 15 + 0 = 110 ETH ✓（守恒）
```

## 关键理解

**DELEGATECALL 中的 Apparent 值**：
- 是从 **A 发给 B** 的原始 value
- 当 B 用 DELEGATECALL 调用 C 时，虽然**没有新的 ETH 转移**给 C
- 但 C 的代码能通过 `CALLVALUE` 读取这个 5 ETH（apparent value）
- 这是为了让 C 的代码能"看到"原始交易的价值

本质上：**DELEGATECALL 中 apparent 是一个虚拟的"传递"，告诉被代理执行的代码原始调用有多少钱，但不会真实转移资金。** 真实的金钱流动早就在 A→B 的时候完成了。

感谢指正！你的能量守恒直觉是对的。

```plaintext
A 调用 B，发送 5 ETH
B 的代码中使用 DELEGATECALL 调用 C，传送 5 ETH

结果：
- A 的余额：95 ETH（失去 5 ETH）
- B 的余额：15 ETH（获得了 A 转来的 5 ETH）
- C 的余额：不变（因为 DELEGATECALL 中 C 是代理执行，代码在 B 的上下文运行）
- C 的代码中执行 CALLVALUE 能读到 5 ETH
- 总金额：95 + 15 + 0 = 110 ETH ✓（守恒）
```

