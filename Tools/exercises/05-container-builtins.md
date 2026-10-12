# 第五题：新增 `list()` 与 `dict()` 内建构造器

在 `test/1d5735e8462` 分支完成。共同交付要求和评审标准见 [README](README.md)。第四题通过评审后再开始。

## 目标

当前内建函数里有 `tuple()`，却没有 `list()` 和 `dict()`。在 `Python/bltinmodule.c` 中新增：

- `list(seq)` —— 由序列构造新的 list。
- `dict()` —— 新的空 dict；`dict(mapping)` —— 由映射对象构造 dict。

训练重点：内建函数注册、序列/映射协议的通用使用、C 层构造对象时的错误处理与引用计数。

## 涉及层次

- `Python/bltinmodule.c`：新增 `builtin_list`、`builtin_dict`，注册进 `builtin_methods[]`。
- 通过对象协议（`tp_as_sequence` / `tp_as_mapping`）通用处理，而不是只认具体类型。

## 语义规定

- `list(seq)`：非序列抛 `TypeError`；请在设计中明确是否允许 `list()` 无参（返回空 list），
  并与后续版本保持一致。
- `dict()`：返回空 dict。
- `dict(mapping)`：`mapping` 必须能提供 `keys()`；否则抛 `TypeError`。
- 不得改变 `tuple()` 及现有内建函数的行为。

## 实现约束

- 只改 `Python/bltinmodule.c`。不新增文件、不改语法/opcode。
- 构造过程复用现有对象接口（如 `newlistobject`、`mapping`/`sequence` 协议），
  不要在 C 里手工重写容器内部结构。

## 验收要求

1. `list(tuple)`、`list(list)`、`list(string)` 结果正确。
2. `list(non-sequence)` 抛 `TypeError`。
3. `dict()`、`dict(mapping)` 结果正确，键值对完整。
4. 每个内建函数的参数个数/类型错误抛 `TypeError`。
5. 构造中途失败时不泄漏已分配对象、不崩溃。

## 测试要求

必须覆盖：

| 情况 | 要求 |
| --- | --- |
| `list` 各类序列 | 结果正确且是新对象 |
| `list` 非序列 | `TypeError` |
| `dict()` / `dict(mapping)` | 结果正确 |
| `dict` 无 `keys` 的实参 | `TypeError` |
| 参数个数错误 | `TypeError` |
| 引用计数 | 反复构造释放后仍可用 |

测试必须有自动判定与非零失败退出状态。

## 设计说明与评审追问

提交时必须：

1. 说明用序列协议构造 list 的过程，以及中途出错时已分配对象的清理方式。
2. 说明 `dict(mapping)` 为何依赖 `keys()`，能否改为直接遍历映射。
3. 说明无参 `list()` / `dict()` 的语义选择及理由。

评审追问：

1. 用协议槽通用处理与直接 `is_listobject` 判断各有何取舍？
2. `list(seq)` 返回的 list 与 `seq` 是否共享元素对象？引用计数如何处理？
3. 内建函数表 `builtin_methods[]` 的注册约定是什么？
