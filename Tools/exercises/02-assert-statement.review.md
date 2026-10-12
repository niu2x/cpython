# 第二题评审记录：`assert` 语句

- 分支：`test/1d5735e8462`
- 提交：`dbcb260f04` — “Add assert statement”
- 评审日期：2026-10-13
- 结论：**代码层面通过，无阻塞项**

## 计分口径说明

本记录按提交者要求，只对“代码正确性”和“代码改动范围”计分。题面里的
自动化测试、独立 commit（不混入生成产物）、5 项设计说明**不在本次计分口径内**，
但它们仍是交付要求，见文末“未计分但未完成的交付项”。

## 得分

| 评审项 | 分值 | 得分 |
| --- | ---: | ---: |
| 行为和边界正确 | 40 | 39 |
| 错误路径、引用与状态恢复 | 25 | 24 |
| 改动范围（仅代码层面） | 15 | 14 |
| **合计** | **80** | **77（折算约 96 / 100）** |

## 提交内容

- `Grammar/Grammar`：新增 `assert_stmt: 'assert' test [ ',' test ]`。
- `Python/compile.c`：`com_assert_stmt`，用 `JUMP_IF_TRUE` 短路，
  `LOAD_NAME AssertionError` 按名字解析异常类型，`RAISE_EXCEPTION` 抛出；
  真、假两条路径各补一次 `POP_TOP`。
- `Python/bltinmodule.c`：新增全局 `object *AssertionError;`，在
  `initerrors()` 里用 `newstdexception("AssertionError")` 注册。
- `Python/graminit.c`、`Include/graminit.h`：用 pgen 重新生成的语法表。

## 实测验证

解释器为当前树按 `make` 重新构建后的 `./python-early`。

| 场景 | 命令要点 | 结果 |
| --- | --- | --- |
| 模块级捕获 | `try: assert 0, "boom"` + `except AssertionError, m` | 命中，`m == "boom"` |
| 函数内捕获 | 函数内 `assert`，外层 `except AssertionError` | 命中（修复前会逃逸） |
| 重新绑定 | `AssertionError = "Rebound"` 后 `except "Rebound"` | 命中（修复前被忽略） |
| 条件求值次数 | 计数器条件函数 | 恰好 1 次 |
| 真时消息短路 | 真条件下带副作用的消息表达式 | 不求解 |
| 异常传播 | 条件、消息分别抛 `ZeroDivisionError` | 原异常传播，未被替换 |
| 无消息形式 | `assert 0` | 异常值为 `''`，随后继续执行 |
| `try/finally` | `assert` 在 `try` 内、`finally` 有副作用 | `finally` 执行且异常被正确捕获 |
| 非法语法 | `assert`、`assert 1 2` 等 | 非零退出（沿基线段错误路径） |
| 栈平衡 | 模块级与函数内各 2,000,000 次 `assert 1` 循环 | 峰值 RSS 与等价普通循环持平，无泄漏 |

## 逐项对照语义规定

| 规定 | 结论 |
| --- | --- |
| `condition` 恰好求值一次 | 满足 |
| 条件为真时不求值 `message` | 满足 |
| 条件为假时求值消息并抛 `AssertionError` | 满足 |
| 条件/消息自身抛异常时传播原异常 | 满足 |
| 使用当前版本的字符串异常机制 | 满足（`RAISE_EXCEPTION` + 字符串异常） |
| `assert` 始终生效，无优化开关 | 满足 |
| `AssertionError` 按普通名字查找、可重新绑定 | 满足（`LOAD_NAME` + 内建 `AssertionError`） |
| 用现有字节码、不新增 opcode | 满足 |
| 通过 pgen 生成语法表、可从源重新生成 | 满足 |

## 修复历史

初评时有三处阻塞问题，均在后续提交中修复：

1. 用 `LOAD_CONST "AssertionError"` 硬编码，导致重新绑定无效、跨 code object
   无法被 `except AssertionError` 捕获。→ 改为 `LOAD_NAME` 名字查找。
2. 未注册内建 `AssertionError`，`except AssertionError` 会先抛 `NameError`。
   → 在 `initerrors()` 注册。
3. `JUMP_IF_TRUE` 不弹栈，成功路径每次执行泄漏一个值栈槽（200 万次循环峰值
   内存从约 2.7 MB 涨到约 18.4 MB）。→ 两条路径各补 `POP_TOP`。

## 剩余问题（非阻塞，均为口味/风格）

- `Python/compile.c` 的 `com_assert_stmt` 用 ANSI 风格定义并混入语句后声明，
  与文件其余部分的 K&R 风格不一致。
- `assert cond` 无消息时异常值取常量 `""`，属可接受选型。
- 真路径显式 `POP_TOP`，假路径依赖异常展开清理条件值——正确但不对称。

## 未计分但未完成的交付项

以下为题面明确要求、但本次提交中未提供的部分（不计入本记录分数，仍需补齐）：

1. 覆盖题面全部场景的自动化测试，且失败必须以非零状态退出。
2. 5 项设计说明：两条路径的栈变化、临时对象引用归属、异常时各路径清理、
   新关键字对旧程序的兼容影响、语法源/生成文件/编译器/内建异常的构建关系。
3. 确认提交未混入生成产物（`Grammar/graminit.[ch]`、各级 `Makefile`、
   `python-early` 等已通过 `.gitignore` 处理）。
