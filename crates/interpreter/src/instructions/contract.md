# Revm contract.rs 文件重点
_Exported on 1/22/2026 at 22:14:17 GMT+8 from Cursor (2.3.29)_

---

**User**

@crates/interpreter/src/instructions/contract.rs:1-288 这个项目是revm，我在学习当前的这个文件，有什么是重点的呢，必须学会和理解的

---

**Cursor**

我来帮你分析这个文件中的重点内容。让我先看看相关的辅助文件，以便给你全面的解释。

现在让我查看相关的宏定义和更多的上下文：

好的，现在我能给你一个全面的分析了。这个文件非常重要！让我为你总结关键点：

## 📚 `contract.rs` 重点学习内容

这个文件实现了 **EVM 合约交互的四大核心指令**，是理解以太坊虚拟机的关键。以下是必须掌握的要点：

---

### **1️⃣ 文件的核心角色**
这个文件实现了 EVM 中所有与**合约之间通信**相关的指令：
- `CREATE` / `CREATE2` - 创建新合约
- `CALL` - 调用其他合约
- `CALLCODE` - 用其他合约代码调用
- `DELEGATECALL` - 使用其他合约代码但保持调用者上下文
- `STATICCALL` - 只读调用（不能修改状态）

---

### **2️⃣ 必须理解的关键概念**

#### **A. 宏系统（Macro System）**
```
popn!([a, b, c], interpreter)     // 从栈弹出3个值
gas!(interpreter, cost)            // 扣费并检查gas是否足够
resize_memory!(interpreter, ...)   // 调整内存大小并扣费
check!(interpreter, PETERSBURG)   // 检查硬分叉版本支持
```
这些宏处理了：
- **栈操作**：安全地弹出值，栈下溢时自动 halt
- **Gas 管理**：扣费并检查 OOG（Out of Gas）
- **内存扩展**：按需要扩展内存，计算扩展成本
- **版本检查**：确保指令在当前硬分叉中支持

---

#### **B. 四种调用类型的关键区别**

| 指令 | 代码来源 | 状态修改 | Caller | Value |
|------|--------|--------|--------|-------|
| **CALL** | 目标账户 | ✅ 能 | 当前合约 | 可转账 |
| **CALLCODE** | 其他账户 | ✅ 能（改自己） | 当前合约 | 可转账 |
| **DELEGATECALL** | 其他账户 | ✅ 能（改自己） | **原始 caller** | **无转账** |
| **STATICCALL** | 目标账户 | ❌ 不能 | 当前合约 | 无转账 |

---

### **3️⃣ CREATE 指令详解（第 25-106 行）**

```rust
pub fn create<WIRE: InterpreterTypes, const IS_CREATE2: bool, H: Host + ?Sized>(
    context: InstructionContext<'_, H, WIRE>,
)
```

**关键步骤：**

1. **检查权限**：`require_non_staticcall!` - 不能在 STATICCALL 中创建合约
2. **版本检查**：CREATE2 仅在 PETERSBURG 及之后支持
3. **弹栈取参**：获取 value、code_offset、len（和 salt for CREATE2）
4. **加载初始化代码**：从内存中读取bytecode
5. **EIP-3860 检查**（Shanghai）：初始化代码大小限制 + 扣 initcode 费用
6. **Gas 计算**：
   - CREATE：固定成本
   - CREATE2：`gas::create2_cost(len)` 
   - **关键**：`gas_limit -= gas_limit / 64` （EIP-150，Tangerine Whistle）
7. **创建新Frame**：通过 `InterpreterAction::NewFrame` 触发新的执行帧

**你需要理解的：**
- 为什么要计算 `gas_limit / 64`？→ 给子调用预留最少 1/64 的 gas
- CREATE2 中 salt 的作用？→ 确定确定性的合约地址
- 初始化代码和部署代码的区别？→ 初始代码返回的是部署代码

---

### **4️⃣ CALL 指令详解（第 111-155 行）**

```rust
pub fn call<WIRE: InterpreterTypes, H: Host + ?Sized>(
    mut context: InstructionContext<'_, H, WIRE>,
)
```

**关键步骤：**

1. **弹栈**：`local_gas_limit, to, value`
2. **检查 STATICCALL 限制**：如果有 value 转账，不能在 static 模式下执行
3. **获取内存范围**：`get_memory_input_and_out_ranges()` 
   - 读取输入数据的内存位置和长度
   - 设置返回数据的存储位置
4. **加载账户信息**：`load_acc_and_calc_gas()` 做什么？
   - 检查目标账户是否存在（冷账户访问 +2600 gas）
   - 获取字节码和字节码哈希
   - 计算最终 gas 限制
5. **创建新 Frame**：构建 `CallInputs` 并设置为新动作

**重要的 enum：**
```rust
CallScheme::Call           // 普通调用
CallValue::Transfer(value) // 转账金额
CallInput::SharedBuffer()  // 内存中的输入数据
```

---

### **5️⃣ CALLCODE vs DELEGATECALL（关键区别！）**

**CALLCODE（第 160-199 行）：**
```rust
target_address: context.interpreter.input.target_address(),  // ✅ 还是原合约
bytecode_address: to,                                          // ✅ 但用别人的代码
value: CallValue::Transfer(value),                             // ✅ 可转账
```

**DELEGATECALL（第 204-243 行）：**
```rust
target_address: context.interpreter.input.target_address(),  // ✅ 原合约
caller: context.interpreter.input.caller_address(),           // ⭐ 原始 caller！
bytecode_address: to,                                          // ✅ 别人代码
value: CallValue::Apparent(call_value),                        // ❌ 虚拟 value
```

**DELEGATECALL 为什么特殊？** → Proxy 合约模式的基础！

---

### **6️⃣ STATICCALL（第 248-287 行）**

```rust
is_static: true,  // ⭐ 关键标志
```

**作用**：只读调用，EVM 会检查不能：
- 修改状态（SSTORE 等）
- 转账
- 创建合约

---

### **7️⃣ 必须掌握的辅助函数**

| 函数 | 功能 |
|------|------|
| `get_memory_input_and_out_ranges()` | 从栈读取 4 个参数，计算内存范围 |
| `load_acc_and_calc_gas()` | 加载账户，计算冷/热访问成本，获取字节码 |
| `resize_memory()` | 扩展内存并扣费 |
| `IntoAddress` | 将 U256 转换为 Address（去掉高位） |

---

### **8️⃣ 你必须理解的执行流程**

```
指令执行 (contract.rs)
    ↓
弹栈获取参数 (popn!)
    ↓
权限/版本检查 (require_non_staticcall!, check!)
    ↓
Gas 计算和扣费 (gas!, gas_limit / 64)
    ↓
加载账户和字节码信息 (load_acc_and_calc_gas)
    ↓
创建新的执行Frame
    ↓
返回给 Handler 处理（在 context.rs 中）
```

---

### **9️⃣ 学习建议**

1. **先掌握 CALL** → 最简单，其他都是变体
2. **理解 DELEGATECALL** → 这是最复杂也最重要的
3. **研究 CREATE2** → 理解 salt 和确定性地址
4. **关注 Gas 计算** → EIP-150 的 `/ 64` 规则很关键
5. **对比阅读**：
   ```
   CALL     vs CALLCODE     → 目标地址的区别
   CALLCODE vs DELEGATECALL → caller 和 value 的区别
   ```

---

### **🔟 相关文件要深入学习**

- `call_helpers.rs` → 辅助函数实现
- `macros.rs` → 宏的定义和机制
- `gas/` 模块 → Gas 计算逻辑
- `InterpreterAction` → Frame 管理
- `CallInputs` 结构 → 调用参数

这个文件浓缩了以太坊合约交互的全部精髓！🚀

```plaintext
popn!([a, b, c], interpreter)     // 从栈弹出3个值
gas!(interpreter, cost)            // 扣费并检查gas是否足够
resize_memory!(interpreter, ...)   // 调整内存大小并扣费
check!(interpreter, PETERSBURG)   // 检查硬分叉版本支持
```

```rust
pub fn create<WIRE: InterpreterTypes, const IS_CREATE2: bool, H: Host + ?Sized>(
    context: InstructionContext<'_, H, WIRE>,
)
```

```rust
pub fn call<WIRE: InterpreterTypes, H: Host + ?Sized>(
    mut context: InstructionContext<'_, H, WIRE>,
)
```

```rust
CallScheme::Call           // 普通调用
CallValue::Transfer(value) // 转账金额
CallInput::SharedBuffer()  // 内存中的输入数据
```

```rust
target_address: context.interpreter.input.target_address(),  // ✅ 还是原合约
bytecode_address: to,                                          // ✅ 但用别人的代码
value: CallValue::Transfer(value),                             // ✅ 可转账
```

```rust
target_address: context.interpreter.input.target_address(),  // ✅ 原合约
caller: context.interpreter.input.caller_address(),           // ⭐ 原始 caller！
bytecode_address: to,                                          // ✅ 别人代码
value: CallValue::Apparent(call_value),                        // ❌ 虚拟 value
```

```rust
is_static: true,  // ⭐ 关键标志
```

```plaintext
指令执行 (contract.rs)
    ↓
弹栈获取参数 (popn!)
    ↓
权限/版本检查 (require_non_staticcall!, check!)
    ↓
Gas 计算和扣费 (gas!, gas_limit / 64)
    ↓
加载账户和字节码信息 (load_acc_and_calc_gas)
    ↓
创建新的执行Frame
    ↓
返回给 Handler 处理（在 context.rs 中）
```

```plaintext
   CALL     vs CALLCODE     → 目标地址的区别
   CALLCODE vs DELEGATECALL → caller 和 value 的区别
```

---

**User**

call_code和static_call有啥区别

---

**Cursor**

好问题！这两个指令有本质的区别，让我为你详细对比：

现在让我看看 STATICCALL：

完美！现在我可以给你清晰的对比了：

## 🔍 CALLCODE vs STATICCALL 核心区别

### **表格对比**

| 特性 | CALLCODE | STATICCALL |
|------|----------|-----------|
| **栈参数** | `[gas, to, value]` (3个) | `[gas, to]` (2个) |
| **硬分叉要求** | Frontier (无限制) | Byzantium (第 251 行 `check!`) |
| **执行代码** | 来自 `to` 地址的代码 | 来自 `to` 地址的代码 |
| **状态修改** | ✅ **可以** | ❌ **不可以** |
| **转账** | ✅ **可以** (`value` 参数) | ❌ **永不转账** (`U256::ZERO` 第 281 行) |
| **target_address** | 当前合约自己 | 目标地址 `to` |
| **is_static** | 继承当前值 | **强制设为 `true`** (第 283 行) |
| **使用场景** | 很少用（过时） | **常用**（查询、验证） |

---

### **代码级别详解**

#### **CALLCODE（第 160-199 行）**

```rust
// 关键行：能转账
value: CallValue::Transfer(value),  // ← 可以有 value 转账
                                      
// 关键行：还是修改自己的状态
target_address: context.interpreter.input.target_address(),  // ← 是自己，不是 to

// 关键行：继承当前状态修改权限
is_static: context.interpreter.runtime_flag.is_static(),  // ← 如果父级是 static，子级也是
```

**CALLCODE 的特点：** 用别人的代码，**修改自己的存储**。这是一个试图做代理但不太好用的指令。

---

#### **STATICCALL（第 248-287 行）**

```rust
// 关键行：只有 2 个栈参数，没有 value！
popn!([local_gas_limit, to], context.interpreter);  // ← 第 252 行，没有 value

// 关键行：永远转账 0
value: CallValue::Transfer(U256::ZERO),  // ← 第 281 行，固定 0

// 关键行：强制设为 true（不可继承其他值）
is_static: true,  // ← 第 283 行，强制只读

// 关键行：真正的目标地址
target_address: to,  // ← 第 276 行
```

**STATICCALL 的特点：** 只读调用，**不能修改任何状态**，包括父级的状态。

---

### **实际场景对比**

#### **CALLCODE 场景（几乎不用）**
```solidity
// 合约 A 想用 B 的逻辑，修改自己的数据
A.storage 被修改
↓
A 调用 CALLCODE 到 B
↓
B 的代码执行，但修改的是 A.storage
```
❌ 问题：容易搞混，已被 DELEGATECALL 取代

---

#### **STATICCALL 场景（常用）**
```solidity
// 读取价格预言机的价格，不会修改任何状态
struct Oracle {
    uint256 price;
}

uint256 latestPrice = Oracle.getPrice();  // ← 这就是 STATICCALL
```
✅ 优点：
- 强制只读，避免意外修改状态
- EVM 可以做优化（缓存、并行化等）
- 安全性强

---

### **为什么 STATICCALL 需要检查硬分叉版本？**

```rust
check!(context.interpreter, BYZANTIUM);  // 第 251 行
```

STATICCALL 是在 **Byzantium (2017年10月)** 才引入的新指令。更老的网络不支持。

相比之下，CALLCODE 是 Frontier 的一部分，所以不需要检查。

---

### **栈参数的区别（很重要！）**

**CALLCODE 栈:**
```
[gas, to, value, in_offset, in_size, out_offset, out_size]
```

**STATICCALL 栈:**
```
[gas, to, in_offset, in_size, out_offset, out_size]
              ↑ 少了 value！
```

因为 STATICCALL 永远转账 0，所以根本不需要 `value` 参数。

---

### **总结记忆**

| 你想做什么 | 用什么 |
|----------|-------|
| 执行别人代码，修改**自己**的存储 | CALLCODE (已过时) |
| 调用合约但**不修改任何状态** | **STATICCALL** ✅ |
| 代理模式（修改自己的存储，用别人代码） | DELEGATECALL ✅ |
| 正常调用（修改目标的存储） | CALL ✅ |

👉 **简单说：STATICCALL = CALLCODE + 只读锁定**

```rust
// 关键行：能转账
value: CallValue::Transfer(value),  // ← 可以有 value 转账
                                      
// 关键行：还是修改自己的状态
target_address: context.interpreter.input.target_address(),  // ← 是自己，不是 to

// 关键行：继承当前状态修改权限
is_static: context.interpreter.runtime_flag.is_static(),  // ← 如果父级是 static，子级也是
```

```rust
// 关键行：只有 2 个栈参数，没有 value！
popn!([local_gas_limit, to], context.interpreter);  // ← 第 252 行，没有 value

// 关键行：永远转账 0
value: CallValue::Transfer(U256::ZERO),  // ← 第 281 行，固定 0

// 关键行：强制设为 true（不可继承其他值）
is_static: true,  // ← 第 283 行，强制只读

// 关键行：真正的目标地址
target_address: to,  // ← 第 276 行
```

```solidity
// 合约 A 想用 B 的逻辑，修改自己的数据
A.storage 被修改
↓
A 调用 CALLCODE 到 B
↓
B 的代码执行，但修改的是 A.storage
```

```solidity
// 读取价格预言机的价格，不会修改任何状态
struct Oracle {
    uint256 price;
}

uint256 latestPrice = Oracle.getPrice();  // ← 这就是 STATICCALL
```

```rust
check!(context.interpreter, BYZANTIUM);  // 第 251 行
```

```plaintext
[gas, to, value, in_offset, in_size, out_offset, out_size]
```

```plaintext
[gas, to, in_offset, in_size, out_offset, out_size]
              ↑ 少了 value！
```

---

**User**

callcode还能用吗，我是说在最新的区块上

---

**Cursor**

很好的问题！让我查一下最新的以太坊规范，看看 CALLCODE 是否被弃用了。

现在让我查看 revm 中如何处理 CALLCODE 的：

完美！现在我有答案了。让我看看 Solidity 编译器是否还生成 CALLCODE：

好的，我已经有了完整的信息。让我给你一个结论性的回答：

---

## 🔴 CALLCODE 在最新区块上的现状

**答案：技术上还能用，但实际上已死。**

### **详细情况：**

1. **EVM 层面 - 仍然支持**
   
   从第 240 行代码可以看到：
```
240    table[CALLCODE as usize] = Instruction::new(contract::call_code, 0); // dynamic
```
   REVM 仍然注册了 CALLCODE 指令，任何以太坊节点仍会执行它。

2. **但在现实中 - 已被废弃**
   
   - **Solidity 编译器不再生成** - 现代 Solidity（0.5.0+）已删除对 CALLCODE 的支持
   - **没人会用** - 所有新代码都用 DELEGATECALL 替代
   - **已从黄皮书中移除** - 作为"遗产指令"存在

---

### **为什么被淘汰了？**

CALLCODE 的语义很奇怪：
```
执行别人的代码 + 修改自己的状态
```

这导致了：
- 逻辑混乱 → 容易出现安全漏洞
- 效率低 → 不如 DELEGATECALL 清晰有力
- 不安全 → 代理模式用 DELEGATECALL 更好

---

### **你能看到的唯一 CALLCODE 场景**

只有在分析**非常老的合约**（Byzantium 之前部署）时，才可能看到 CALLCODE 指令。现代区块上几乎不存在新部署的 CALLCODE。

**简单结论：别用 CALLCODE，用 DELEGATECALL！** 🎯

```plaintext
240    table[CALLCODE as usize] = Instruction::new(contract::call_code, 0); // dynamic
```

```plaintext
执行别人的代码 + 修改自己的状态
```

---

**User**

给我一个使用了call_code的solidity代码例子

---

**Cursor**

好问题！让我先查一下是否有任何使用 CALLCODE 的示例或测试：

根据搜索结果和我的分析，我有一个坦诚的回答给你：

---

## ❌ 实际上没有"使用了 CALLCODE 的 Solidity 代码"

### **为什么找不到例子**

1. **Solidity 0.5.0+ 就移除了** - 编译器不再为 CALLCODE 生成字节码
2. **现代代码都用 DELEGATECALL** - 它是完全替代品
3. **CALLCODE 只在古董合约中存在** - 2016 年之前的代码可能有

### **你能看到 CALLCODE 的唯一方式**

#### **方式 1：低级汇编（需要极低版本 Solidity）**

在 Solidity 0.4.x 时代，你可以写这样的代码：

```solidity
// Solidity 0.4.x - 已过时！
pragma solidity ^0.4.0;

contract OldProxy {
    address library;
    
    function call(bytes memory data) public returns (bytes memory) {
        // 这样会生成 CALLCODE 指令（0xf2）
        uint256 result;
        assembly {
            result := callcode(
                gas(),
                library,
                0,  // value
                add(data, 0x20),
                mload(data),
                0,
                0
            )
        }
    }
}
```

⚠️ **但这代码在现代 Solidity（0.5.0+）中根本编译不了！**

#### **方式 2：使用 delegatecall 替代（现代方式）**

```solidity
// Solidity 0.8.x - 正确做法！
pragma solidity ^0.8.0;

contract ModernProxy {
    address implementation;
    
    function delegateCall(bytes calldata data) external returns (bytes memory) {
        (bool success, bytes memory result) = implementation.delegatecall(data);
        require(success, "delegatecall failed");
        return result;
    }
}
```

这会生成 **DELEGATECALL（0x18）** 指令，而不是 CALLCODE。

---

## 🎯 总结

| 内容 | 状态 |
|------|------|
| 现代 Solidity 中的 CALLCODE 代码 | ❌ 不存在 |
| 汇编中的 callcode() 函数 | ⚠️ Solidity 0.4.x 才有 |
| 等价替代品 | ✅ DELEGATECALL（推荐） |

**结论：学 CALLCODE 更多是历史和理论学习，实际开发中你永远用不到它。** 重点应该放在 **DELEGATECALL** 上！

```solidity
// Solidity 0.4.x - 已过时！
pragma solidity ^0.4.0;

contract OldProxy {
    address library;
    
    function call(bytes memory data) public returns (bytes memory) {
        // 这样会生成 CALLCODE 指令（0xf2）
        uint256 result;
        assembly {
            result := callcode(
                gas(),
                library,
                0,  // value
                add(data, 0x20),
                mload(data),
                0,
                0
            )
        }
    }
}
```

```solidity
// Solidity 0.8.x - 正确做法！
pragma solidity ^0.8.0;

contract ModernProxy {
    address implementation;
    
    function delegateCall(bytes calldata data) external returns (bytes memory) {
        (bool success, bytes memory result) = implementation.delegatecall(data);
        require(success, "delegatecall failed");
        return result;
    }
}
```

---

**User**

给我用低版本的solidity来写一个callcode()合约简单例子

---

**Cursor**

好的，让我为你创建一个低版本 Solidity 的 CALLCODE 例子。这是一个教学用的历史代码示例：

## 低版本 Solidity（0.4.x）CALLCODE 例子

这个例子展示了两个合约：一个提供库函数，一个通过 CALLCODE 调用它。

### **合约 1：Library（库合约）**

```solidity
pragma solidity ^0.4.0;

contract Library {
    uint public value = 0;
    
    // 库函数：增加值
    function add(uint x) public {
        value += x;
    }
    
    // 库函数：获取值
    function getValue() public constant returns (uint) {
        return value;
    }
}
```

### **合约 2：CallcodeUser（调用者）**

```solidity
pragma solidity ^0.4.0;

contract CallcodeUser {
    address public libraryAddress;
    uint public myValue = 0;  // ← 关键：这个 storage 会被修改
    
    function CallcodeUser(address lib) {
        libraryAddress = lib;
    }
    
    // 通过 CALLCODE 调用库函数
    function useLibraryAdd(uint x) public {
        // 使用 callcode 执行库的代码，但修改的是自己的 storage
        // Stack: [gas, address, value, inputOffset, inputSize, outputOffset, outputSize]
        
        // 首先需要准备调用数据（add(uint) 的 selector）
        // add(uint) 的 function selector 是 keccak256("add(uint256)") 的前 4 字节
        
        assembly {
            // 将 function selector (0x1003e2d2) 写入内存位置 0
            mstore(0x0, 0x1003e2d200000000000000000000000000000000000000000000000000000000)
            // 将参数 x 写入内存位置 4
            mstore(0x4, x)
            
            // 调用 CALLCODE
            // callcode(gas, address, value, inputOffset, inputSize, outputOffset, outputSize)
            let result := callcode(
                gas,                    // 剩余 gas
                sload(0),              // libraryAddress（从 storage slot 0 读取）
                0,                     // 没有转账（value = 0）
                0,                     // 输入数据在内存位置 0
                0x24,                  // 输入数据大小 36 字节（4 + 32）
                0,                     // 输出数据在内存位置 0
                0                      // 输出数据大小 0（不需要返回值）
            )
        }
    }
    
    // 获取自己的 storage 值（已被库函数修改过）
    function getMyValue() public constant returns (uint) {
        return myValue;  // 这个值被库函数改过了！
    }
}
```

---

## 🔍 代码关键点解析

| 关键点 | 说明 |
|--------|------|
| **CALLCODE 特性** | 执行库的代码，**但修改的是 CallcodeUser 的 storage** |
| **value = 0** | CALLCODE 虽然支持转账，但这个例子不转账 |
| **assembly 块** | 使用低级汇编直接生成 CALLCODE（0xf2）指令 |
| **mstore** | 在内存中构造函数调用数据 |
| **gas** | 传入当前剩余的 gas |

---

## 📊 执行流程

```
1. CallcodeUser.useLibraryAdd(10) 被调用
   ↓
2. 内存中构造函数调用：add(10)
   ↓
3. CALLCODE 指令执行：
   - 调用 Library 的字节码
   - 但操作的是 CallcodeUser.myValue（自己的 storage）
   ↓
4. Library 的代码运行：value += x
   - 但这里的 value 其实是 CallcodeUser.myValue
   - 所以 CallcodeUser.myValue 被增加了！
   ↓
5. getMyValue() 返回修改后的值
```

---

## ⚠️ 为什么这很危险

```solidity
// 如果 Library 有恶意代码：
function destroy() public {
    // 当通过 CALLCODE 调用时
    // selfdestruct 会销毁 CallcodeUser，而不是 Library！
    selfdestruct(msg.sender);  // ← 调用者被销毁了！
}
```

这就是为什么 CALLCODE 被淘汰的原因！

---

## 🎯 DELEGATECALL 现代替代品

如果用现代 Solidity（0.8.x）写同样功能：

```solidity
pragma solidity ^0.8.0;

contract Library {
    uint public value = 0;
    
    function add(uint x) public {
        value += x;
    }
}

contract ProxyUser {
    address public library;
    uint public myValue = 0;
    
    constructor(address lib) {
        library = lib;
    }
    
    // 现代方式：用 delegatecall
    function useLibraryAdd(uint x) public {
        // delegatecall 是 CALLCODE 的改进版
        (bool success, ) = library.delegatecall(
            abi.encodeWithSignature("add(uint256)", x)
        );
        require(success);
    }
}
```

**差异**：
- ❌ CALLCODE - 奇怪的语义，容易出错
- ✅ DELEGATECALL - 清晰的代理模式语义

---

## 总结

这个例子展示了为什么 CALLCODE 被废弃：它的语义太奇怪，太容易造成安全问题。现代开发**永远应该用 DELEGATECALL**！

```solidity
pragma solidity ^0.4.0;

contract Library {
    uint public value = 0;
    
    // 库函数：增加值
    function add(uint x) public {
        value += x;
    }
    
    // 库函数：获取值
    function getValue() public constant returns (uint) {
        return value;
    }
}
```

```solidity
pragma solidity ^0.4.0;

contract CallcodeUser {
    address public libraryAddress;
    uint public myValue = 0;  // ← 关键：这个 storage 会被修改
    
    function CallcodeUser(address lib) {
        libraryAddress = lib;
    }
    
    // 通过 CALLCODE 调用库函数
    function useLibraryAdd(uint x) public {
        // 使用 callcode 执行库的代码，但修改的是自己的 storage
        // Stack: [gas, address, value, inputOffset, inputSize, outputOffset, outputSize]
        
        // 首先需要准备调用数据（add(uint) 的 selector）
        // add(uint) 的 function selector 是 keccak256("add(uint256)") 的前 4 字节
        
        assembly {
            // 将 function selector (0x1003e2d2) 写入内存位置 0
            mstore(0x0, 0x1003e2d200000000000000000000000000000000000000000000000000000000)
            // 将参数 x 写入内存位置 4
            mstore(0x4, x)
            
            // 调用 CALLCODE
            // callcode(gas, address, value, inputOffset, inputSize, outputOffset, outputSize)
            let result := callcode(
                gas,                    // 剩余 gas
                sload(0),              // libraryAddress（从 storage slot 0 读取）
                0,                     // 没有转账（value = 0）
                0,                     // 输入数据在内存位置 0
                0x24,                  // 输入数据大小 36 字节（4 + 32）
                0,                     // 输出数据在内存位置 0
                0                      // 输出数据大小 0（不需要返回值）
            )
        }
    }
    
    // 获取自己的 storage 值（已被库函数修改过）
    function getMyValue() public constant returns (uint) {
        return myValue;  // 这个值被库函数改过了！
    }
}
```

```plaintext
1. CallcodeUser.useLibraryAdd(10) 被调用
   ↓
2. 内存中构造函数调用：add(10)
   ↓
3. CALLCODE 指令执行：
   - 调用 Library 的字节码
   - 但操作的是 CallcodeUser.myValue（自己的 storage）
   ↓
4. Library 的代码运行：value += x
   - 但这里的 value 其实是 CallcodeUser.myValue
   - 所以 CallcodeUser.myValue 被增加了！
   ↓
5. getMyValue() 返回修改后的值
```

```solidity
// 如果 Library 有恶意代码：
function destroy() public {
    // 当通过 CALLCODE 调用时
    // selfdestruct 会销毁 CallcodeUser，而不是 Library！
    selfdestruct(msg.sender);  // ← 调用者被销毁了！
}
```

```solidity
pragma solidity ^0.8.0;

contract Library {
    uint public value = 0;
    
    function add(uint x) public {
        value += x;
    }
}

contract ProxyUser {
    address public library;
    uint public myValue = 0;
    
    constructor(address lib) {
        library = lib;
    }
    
    // 现代方式：用 delegatecall
    function useLibraryAdd(uint x) public {
        // delegatecall 是 CALLCODE 的改进版
        (bool success, ) = library.delegatecall(
            abi.encodeWithSignature("add(uint256)", x)
        );
        require(success);
    }
}
```



