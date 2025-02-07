---
title: 新 SQL 规划器框架设计
description: 新 SQL 规划器框架设计 RFC
---

- 开始日期: 2021/09/13
- 跟踪问题: https://github.com/databendlabs/databend/issues/1217

# 概述

为了支持更复杂的 SQL 查询，例如包含 `JOIN` 和相关子查询的查询，我们需要重新设计 SQL 规划器组件。

当前实现中我们将在本 RFC 中讨论的主要问题如下：

- 不支持 `JOIN` 和相关子查询
- 不具备严格的语义检查能力，例如类型检查和名称解析，这给 SQL 优化和执行过程中的正确性保障带来了不必要的复杂性
- 没有通用的 SQL 优化框架

让我们从一个简单的例子开始。

在 SQL 中，允许元组中字段名称重复。在 PostgreSQL 中，结果集可以包含名称相同的不同列：

```
postgres=# create table t(a INT);
CREATE TABLE
postgres=# insert into t values(1),(2);
INSERT 0 2
postgres=# SELECT * from t t1 cross join t t2;
 a | a
---+---
 1 | 1
 1 | 2
 2 | 1
 2 | 2
(4 rows)
```

我们可以看到有两个名为 `a` 的字段，一个来自派生表 `t1`，另一个来自派生表 `t2`。

如果你尝试引用重复名称 `a` 的列，它将返回一个错误：

```
postgres=# SELECT a from t t1, t t2;
ERROR:  column reference "a" is ambiguous
LINE 1: SELECT a from t t1, t t2;
```

而你可以使用规范名称如 `t.a` 来引用列，因为在查询上下文中表名必须是唯一的。

目前，`databend` 使用 `DataSchema` 来表示输入和输出关系模式，这无法提供足够的信息来处理上述情况。在 `DataSchema` 中，每一列都用 `DataField` 表示，其定义如下：

```rust
pub struct DataField {
    name: String,
    data_type: DataType,
    nullable: bool,
}
```

`DataSchema` 中的每个 `DataField` 都用唯一的 `name` 字符串标识。目前，`name` 仅表示列名，因此很难用这种抽象实现 `JOIN`。我们将在后面的章节中详细讨论这个问题的解决方案。

第二个问题是关于语义检查。

以类型检查为例，表达式中的每个变量（例如列引用、常量值）都有自己的数据类型。每个标量表达式对其参数的数据类型有要求，例如 `+` 表达式要求其参数为数值类型。

为了确保查询有效且正确，我们需要在执行查询之前进行类型检查。

由于优化器和执行器都需要类型检查，最好用单一组件来解决这个问题，这样可以使其更易于维护。

最后一个问题是关于查询优化框架。

许多现代优化器都是以 Volcano/Cascades 风格实现的，这是一种高度模块化的方法。

典型的 Cascades 优化器由独立的模块组成：

- 转换规则
- 实现规则
- 探索引擎
- 成本模型

有趣的是，规则系统（转换和实现）与探索引擎和成本模型解耦，这意味着可以轻松构建启发式优化器而无需基于成本的优化（CBO）。一旦我们准备实现 CBO，规则系统可以重用。

实际上，这是实际的做法。在一些工业级 Cascades 实现（例如 SQL Server 和 CockroachDB）中，总是有一个启发式优化阶段，例如 SQL Server 中的 `pre-exploration` 和 CockroachDB 中的 `normalization`，它们通常与探索引擎共享相同的规则系统。

总之，本 RFC 将：

- 引入一个新框架以支持规划 `JOIN` 和相关子查询
- 引入一个规则系统，使开发人员能够轻松编写转换规则

# 设计细节

## 架构

在当前实现中，SQL 查询将按以下方式处理：

1. `PlanParser` 将 SQL 文本解析为 AST（抽象语法树）
2. `PlanParser` 还将从 AST 构建一个用 `PlanNode` 表示的规范计划树
3. 构建计划树后，`Optimizer` 将对计划进行一些规范优化，并生成最终的 `PlanNode`
4. `Interpreter` 将 `PlanNode` 作为输入，并将其解释为由 `Processor` 组成的可执行 `Pipeline`
5. 执行器将使用特定运行时执行 `Pipeline`

在我们的新框架中，`PlanParser` 将被重构为两个组件：

- `Parser`：将 SQL 文本解析为统一的 AST 表示，这在 [此 PR](https://github.com/databendlabs/databend/pull/1478) 中已经引入
- `Binder`：将 AST 中出现的变量与数据库中的对象（例如表、列等）绑定，并执行语义检查（名称解析、类型检查）。将生成计划树的逻辑表示

此外，将引入一个新的优化器，用 `规则系统` 取代当前的优化器。

## Binder

由于 SQL 查询中可能有许多语法上下文，我们需要一种方法来跟踪它们之间的依赖关系以及不同上下文中 `name` 的可见性。

这里我们提出了抽象 `Metadata`，它存储了查询优化所需的所有元数据，包括来自目录的基本表、派生表（子查询和连接）和列。每个表和列都将被分配一个唯一标识符。

```rust
pub struct Metadata {
    pub tables: Vec<TableEntry>,
}

pub struct TableEntry {
    pub index: IndexType, // `Metadata` 中表的索引
    pub columns: Vec<ColumnEntry>,
    // 以及表的元数据，例如名称、数据库名称等
}

pub struct ColumnEntry {
    pub index: ColumnIndex,
    // 以及列的元数据，例如名称、数据类型等
}

pub type IndexType = usize;

pub struct ColumnIndex {
    pub table_index: IndexType, // 此列所属表的索引
    pub column_index: IndexType, // 表中列的索引
}
```

因此，在名称解析之后，每个变量将绑定到一个唯一的 `ColumnIndex` 而不是字符串，因此我们不必担心重复名称的问题。

在绑定过程中，我们需要跟踪绑定的状态。状态可能包含以下信息：

- 当前上下文中可见的列，用于处理通配符结果集（`SELECT * FROM t`）
- 上下文中的列是否为分组键
- 列是否为派生列（例如 `a+1 AS a` 这样的投影）
- 变量是否来自当前子查询，以识别相关列引用

为了维护状态，我们提出了数据结构 `BindContext`（这个名字受到 CMU Peloton 的 `BinderContext` 启发，我认为这是一个非常恰当的名字）。

`BindContext` 是一个类似栈的结构，栈中的每个 `BindContext` 节点记录相应语法上下文的状态。SQL 绑定是一个自底向上的过程，这意味着它将递归处理 AST，并将数据源（例如表扫描、连接、子查询）产生的列添加到

简而言之，`BindContext` 是一组列引用。为了清楚起见，我们将使用图表来解释这个机制的工作原理。

以这个例子为例：

```sql
create table t (a int);

SELECT * from t -- 表: 上下文 1 [t.a]
cross join t t1 -- 表: 上下文 2 [t1.a]
-- 连接: 上下文 3 [t.a, t1.a]
where t.a = 1
```

根据 SQL 的语义，我们可以将绑定过程描述如下：

1. 为表 `t` 创建一个空的 `BindContext` 上下文 1，并用 `t` 中的列填充它
2. 为表 `t` 创建一个空的 `BindContext` 上下文 2，用 `t` 中的列填充它，并将表重命名为 `t1`
3. 为 `t cross join t1` 创建一个空的 `BindContext` 上下文 3，并用 `t` 和 `t1` 中的列填充它
4. 对谓词 `t.a = 1` 执行名称解析
5. 查找上下文 3，并找到变量 `t.a` 对应的 `ColumnEntry`

让我们看看 `BindContext` 如何处理相关子查询。

相关子查询表示子查询依赖于外部上下文中的列。

有一个规范的 `Apply` 操作符来执行相关子查询，它将为每个元组（类似于交叉连接）评估子查询表达式。然而，大多数相关子查询可以解相关为连接（例如半连接、反半连接等）。

以这个查询为例：

```sql
create table t (a int);

SELECT * from t -- 表: 上下文 1 [t.a]
where exists (
    SELECT * from t t1 -- 表: 上下文 2 带有父上下文 1 [t1.a]
    where t1.a = t.a -- t.a 是相关列，因为它来自外部查询中的 t
);
```

在绑定 `exists` 子查询之前，我们将为其创建一个新的 `BindContext`，并将外部 `BindContext` 作为其父上下文。

当我们在子查询中绑定相关列引用 `t.a` 时，我们将首先查找当前 `BindContext` 以查看是否存在适当的列，如果不存在，我们将继续在父上下文中查找，直到找到相应的列或遍历所有父上下文。

如果在父上下文中找到列，那么我们可以确认这个子查询是相关子查询，并且绑定到父上下文的列引用是相关列。

该过程可以总结如下：

1. 为表 `t` 创建一个空的 `BindContext` 上下文 1，并用 `t` 中的列填充它
2. 为表 `t` 创建一个空的 `BindContext` 上下文 2，并将上下文 1 作为其父上下文，用 `t` 中的列填充它，并将表重命名为 `t1`
3. 对谓词 `t1.a = t.a` 执行名称解析
4. 查找上下文 2，并找到变量 `t1.a` 对应的 `ColumnEntry`，但找不到 `t.a`。因此我们将继续执行步骤 5
5. 查找上下文 2 的父上下文（上下文 1），并找到变量 `t.a` 对应的 `ColumnEntry`。由于在外部上下文中找到了变量，它将被标记为相关列引用，子查询将被标记为相关

## 优化器

### Cascades 优化器的简要介绍

SQL 优化基于关系代数的等价性。有许多不同的定理和引理可以帮助我们识别两个关系代数树是否逻辑上等价。通过一组等价的关系表达式，我们可以用成本模型评估它们并找到最优表达式。

Cascades 优化器是由 Goetz Graefe 在他的 [论文](https://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/Papers/Cascades-graefe.pdf) 中引入的查询优化框架。

在 Cascades 查询优化器中，SQL 查询将被翻译成树状结构 `Expression`，关系操作符（`Operator`）作为其节点。

有三种 `Operator`：

- 逻辑操作符：表示关系代数，例如 `Select`、`Get`、`Inner Join`
- 物理操作符：表示带有实现的关系代数，例如 `Hash Join`
- 标量操作符：表示标量表达式，例如 `Select` 操作符中的谓词表达式（`AND`、`OR`、`+`）

转换规则用于对 `Expression` 执行代数转换。

每个转换规则都有关于它可以应用于的关系操作符的描述，我们称之为 `规则模式`。优化器将提供一种方案来遍历 `Expression`，检查是否有任何规则可以应用于子 `Expression`，然后对匹配的规则应用转换。

`Expression` 的变体在转换过程中生成。例如：

```sql
SELECT * FROM t INNER JOIN t1 ON t.a = t1.a;
```

通过 `JoinCommutativity` 规则，上述 SQL 可以转换为等价的 SQL：

```sql
SELECT * FROM t1 INNER JOIN t ON t.a = t1.a;
```

为了减少转换生成的重复 `Expression`，Cascades 使用自顶向下的方法来枚举变体。

引入了一个结构 `Memo` 来存储变体。每个 `Memo` 由 `Group` 组成，每个 `Group` 是一组等价的 `Expression`。

与前面提到的 `Expression` 不同，`Memo` 中的 `Expression` 将 `Group` 作为其子节点而不是 `Expression`，因此等价的 `Expression` 可以共享子节点候选。

以 `JoinCommutativity` 为例，原始 SQL 的 `Memo` 可以表示为：

```
Group 1: [Get(t)]

Group 2: [Get(t1)]

Group 3: [Join(1, 2, "t.a = t1.a")]
```

应用 `JoinCommutativity` 转换后，`Memo` 将变为：

```
Group 1: [Get(t)]

Group 2: [Get(t1)]

Group 3: [Join(1, 2, "t.a = t1.a"), Join(2, 1, "t.a = t1.a")]
```

现在你已经基本了解了 Cascades 优化器框架。尽管 Cascades 还有许多其他重要概念，但这个简要介绍足以让你理解 RFC。

### Databend 的新优化器

在新优化器框架中，有几个核心结构。

`Plan`，逻辑操作符和物理操作符的枚举。与规范的 Cascades 不同，我们不将标量操作符作为 `Plan` 的一部分。

```rust
enum Plan {
    // ...
}
```

`SExpr`，单表达式的缩写，表示 `Plan` 的树。

```rust
struct SExpr {
    pub plan: Plan,
    pub children: Vec<Plan>,
}
```

`Memo`，`Group` 的集合，如 Cascades 中的 `Memo`。

```rust
struct Memo {
    pub groups: Vec<Group>,
}
```

`Group`，`MExpr` 的集合，如 Cascades 中的 `Group`。

```rust
struct Group {
    pub expressions: Vec<MExpr>,
}
```

`MExpr`，`Memo` 中 `Expression` 的表示。

```rust
struct MExpr {
    pub plan: Plan,
    pub children: Vec<GroupIndex>,
}
```

`Rule`，转换规则的特征。`Rule` 可以分为探索规则（生成等价的逻辑表达式）和实现规则（生成物理表达式）。

```rust
trait Rule {
    fn pattern(&self) -> &SExpr;

    fn apply(&self, expr: &SExpr, state: &mut TransformState) -> Result<()>;
}
```

`HeuristicOptimizer`，优化器对 `SExpr` 应用转换，不生成变体。

`CascadesOptimizer`，优化器对 `SExpr` 应用转换，生成 `Memo`，并最终从 `Memo` 中提取最优的 `SExpr`。

让我们看看将 SQL 查询翻译成 `Processor` 的整个过程：

1. SQL 查询文本被解析为 AST
2. `Binder` 将 AST 翻译为带有 `Catalog` 的规范逻辑 `SExpr`
3. `HeuristicOptimizer` 优化规范逻辑 `SExpr`
4. `CascadesOptimizer` 获取 `HeuristicOptimizer` 的输出，构建 `Memo`，应用规则，并返回最优的 `SExpr`
5. `Executor` 获取 `CascadesOptimizer` 生成的物理 `SExpr`，并构建可执行的 `Processor`

# 里程碑

经过这次重构，我们希望：

- 为 `JOIN` 提供朴素实现（等值连接的哈希连接和交叉连接的嵌套循环连接），包括规划和执行
- 支持使用 `databend` 运行 TPCH 基准测试中的大多数查询（包含不同类型的连接和相关子查询）
- 实现几个简单的优化规则，例如外连接消除、解相关、谓词下推等
- 迁移到新的规划器框架

同时，我们不会：

- 重视性能，相关工作应在下一阶段进行
- 实现基于成本的优化，这项工作依赖于统计系统的设计
