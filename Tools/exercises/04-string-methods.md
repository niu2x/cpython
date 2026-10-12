# 第四题：给字符串对象加方法支持

在 `test/1d5735e8462` 分支完成。共同交付要求和评审标准见 [README](README.md)。第三题通过评审后再开始。

## 目标

当前字符串对象是“无属性对象”（`"abc".upper()` 直接报
`TypeError: attribute-less object`）。给字符串类型加上属性/方法访问，并实现两个方法：

- `s.find(sub)` —— 返回子串首次出现的下标，找不到返回 `-1`。
- `s.split()` / `s.split(sep)` —— 默认按任意空白切分，或按 `sep` 切分，返回 list。

训练重点：类型对象的 `tp_getattr`、`methodlist`/`findmethod` 约定、返回对象的引用所有权、
字符串与列表的构造。

## 涉及层次

- `Objects/stringobject.c`：仿照 `list_getattr`（`Objects/listobject.c`）实现
  `string_getattr` 与 `string_methods[]`，并把 `Stringtype` 里的 `tp_getattr`
  从 `0` 改为该函数。
- 字符串查找、切分逻辑可在本文件内实现（也可复用 `strop` 中的等价逻辑，但需说明取舍）。

## 语义规定

- 属性名不存在时抛 `AttributeError`。
- `s.find(sub)`：参数个数或类型错误抛 `TypeError`；`sub` 为空串时返回 `0`。
- `s.split()`（无参）：按任意空白切分，丢弃空段；`s.split(sep)`：按 `sep` 切分，
  允许空段。
- 返回类型：`find` 返回 int；`split` 返回新 list，元素为新的字符串对象。
- 不得改变字符串现有的比较、拼接、`%` 格式化等行为。

## 实现约束

- 只改 `Objects/stringobject.c`（必要时加 `Include` 声明）。不新增文件、不改语法/opcode。
- 复用现有类型对象/方法表约定，不要为字符串另造一套属性机制。

## 验收要求

1. `s.find(sub)` 命中/未命中/空子串行为正确。
2. `s.split()` 与 `s.split(sep)` 结果正确，含只有空白、前导/尾随分隔符等边界。
3. 属性名错误抛 `AttributeError`；方法参数错误抛 `TypeError`。
4. 返回值对象的引用计数正确：反复调用、丢弃返回值后无泄漏、无悬空。
5. 现有字符串行为不回归。

## 测试要求

必须覆盖：

| 情况 | 要求 |
| --- | --- |
| `find` 命中 / 未命中 / 空子串 | 下标或 `-1`/`0` |
| `split()` 各种空白组合 | 段数与本内容正确 |
| `split(sep)` 空段 | 保留空段 |
| 未知属性 / 错误参数 | `AttributeError` / `TypeError` |
| 引用计数 | 反复调用后仍可用 |
| 回归 | 拼接、比较、`%` 格式化不变 |

测试必须有自动判定与非零失败退出状态。

## 设计说明与评审追问

提交时必须：

1. 说明为什么字符串现在找不到属性（`tp_getattr` 为 `0` 意味着什么）。
2. 说明 `methodlist`/`findmethod` 的约定，以及方法对象如何绑定 `self`。
3. 说明 `split` 返回的 list 及其元素的引用计数如何转移。

评审追问：

1. 加 `tp_getattr` 后，`"abc".__class__` 之类的访问会怎样？
2. `find` 用返回 `-1` 而非抛异常，在错误处理上有何利弊？
3. `split` 切分出的每个字符串是共享原串内存还是独立新建？本版本如何权衡？
