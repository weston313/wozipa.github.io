title: Spark Hint
date: 2019-11-26 15:24:51
categories: spark
tags: SparkHint,源代码
---
## 介绍
SparkHint是在使用SparkSQL开发过程中，针对SQL进行优化的一点小技巧，我们可以通过Hint的方式实现BraodcastJoin优化、Reparttion分区等操作，提供了传统SQL中无法实现的一些功能。


## 语法介绍
SparkSQL的语法定义是通[Antlr4](https://github.com/antlr/antlr4)实现的，Antlr4是一个提供语法定义、语法解析等第三方库，Antlr4语法的定义基本复合正则表达式，因此会正则表达式的同学可以尝试去理解一下下面这段代码。

```
# 来自于Spark源代码的SqlBase.g4文件
# 内容有部分删除，如查看源代码是请注意

# 定义特别查询的语法树节点
querySpecification
    : ((kind=SELECT (hints+=hint)* setQuantifier? namedExpressionSeq fromClause?
       | fromClause (kind=SELECT setQuantifier? namedExpressionSeq)?)
       lateralView*
       (WHERE where=booleanExpression)?
       aggregation?
       (HAVING having=booleanExpression)?
       windows?)
    ;

# Hint基本的格式，开始符号、结束符号，语句列表格式，格式为
# /*+ 多条语句 */
hint
    : '/*+' hintStatements+=hintStatement (','? hintStatements+=hintStatement)* '*/'
    ;
# 单个Hint的组成
# 名称
# 名称(参数, 参数2...)
hintStatement
    : hintName=identifier
    | hintName=identifier '(' parameters+=primaryExpression (',' parameters+=primaryExpression)* ')'
    ;
```

上述Hint的语法树中定义了Hint的使用方式为：

+  SparkHint只能在Select语句中使用
+  SparkHint的结果必须要SELECT关键字以后标识，且结构体为
/*+ ... */

目前SparkHint支持的语法很少，只有两种语法:

+ Broadcast: MAPJOIN/BROADCASTJOIN/BROADCAST
+ Coalesce: REPARTITION/COALESCE

下面的例子简单介绍一下SparkHint是如何使用的：

```
	select /*+ BRAODCASTJOIN(B) */
    	A.COL1
        , B.COL2
    FROM 
    	A LEFT JOIN B USING(COL3); 

```


## 源代码解析

SparkSQL的解析过程大致上可以分为一下几个过程:

+ G4文件定义语法
+ AstBuilder/SparkAstBuilder将语法树种的信息解析成相应的逻辑计划
+ Analyzer层将一些LogicalPlan翻译成另一个LogicalPlan语法
+ Optimizer层在Schema等级别进行优化，生成LogicalPlan
+ SparkPlan将LogicPlan翻译成影响的SparkPlan
+ 执行SparkPlan映射成底层的RDD操作

这里简单介绍一下SparkSQL的解析过程，后续我们再写一篇文章详细介绍SparkSQL解析的整体过程。

### G4文件
当我们使用 sparkSession.sql(string) 执行SparkHint语句时，首先要经历语法解析，在这里会将SparkHint的语法解析成相应的hintStatement语法，并且将HintStatement中的参数进行提取。
```
# Hint基本的格式，开始符号、结束符号，语句列表格式
hint
    : '/*+' hintStatements+=hintStatement (','? hintStatements+=hintStatement)* '*/'
    ;

# 单个Hint的组成
# 名称
# 名称(参数, 参数2...)
hintStatement
    : hintName=identifier
    | hintName=identifier '(' parameters+=primaryExpression (',' parameters+=primaryExpression)* ')'
```

### AstBuilder解析

SparkSQL采用的是Antlr4的Visitor模式，继承Visitor接口实现相应的代码(AstBuilder.java和 SparkAstBuilder.java)，这两个类的主要作用是将一条SQL语句的每一部分翻译成相应的逻辑计划(LogicalPlan)代码。然后SparkSQL的三层将生成的LogicalPlan进行相应的解析、优化和转化成底层的RDD操作。

```scala
  /**
   * Add [[UnresolvedHint]]s to a logical plan.
   */
  private def withHints(
      ctx: HintContext,
      query: LogicalPlan): LogicalPlan = withOrigin(ctx) {
    var plan = query
    // 将hintStatement列表转换成UnresolvedHint对象
    ctx.hintStatements.asScala.reverse.foreach { case stmt =>
      plan = UnresolvedHint(stmt.hintName.getText, stmt.parameters.asScala.map(expression), plan)
    }
    plan
  }
```

这段代码结合G4文件的语法就能得到一下几点:
+ 一个hintStatemenet转换成一个UnresolvedHint
+ 生成UnresolvedHint对象时，将定义的hintName以及params进行了存储


### Analyzer 

Analyzer对象可以理解成是一系列解析过程的集合，每一个解析过程会针对一种LogicalPlan进行解析，并生成后续可以被优化以及翻译的过程LogicalPlan。

Analyzer的解析过程列表，Hint的解析过程排列在第一个：

```scala 
# Spark 2.4 版本代码
lazy val batches: Seq[Batch] = Seq(
    Batch("Hints", fixedPoint,
      new ResolveHints.ResolveBroadcastHints(conf),
      ResolveHints.ResolveCoalesceHints,
      ResolveHints.RemoveAllHints),
    Batch("Simple Sanity Check", Once,
      LookupFunctions),
    Batch("Substitution", fixedPoint,
      CTESubstitution,
      WindowsSubstitution,
      EliminateUnions,
      new SubstituteUnresolvedOrdinals(conf)),
    Batch("Resolution", fixedPoint,
      ResolveTableValuedFunctions ::
      ResolveRelations ::
      ResolveReferences ::
      ResolveCreateNamedStruct ::
      ResolveDeserializer ::
      ResolveNewInstance ::
      ResolveUpCast ::
      ResolveGroupingAnalytics ::
      ResolvePivot ::
      ResolveOrdinalInOrderByAndGroupBy ::
      ResolveAggAliasInGroupBy ::
      ResolveMissingReferences ::
      ExtractGenerator ::
      ResolveGenerate ::
      ResolveFunctions ::
      ResolveAliases ::
      ResolveSubquery ::
      ResolveSubqueryColumnAliases ::
      ResolveWindowOrder ::
      ResolveWindowFrame ::
      ResolveNaturalAndUsingJoin ::
      ResolveOutputRelation ::
      ExtractWindowExpressions ::
      GlobalAggregates ::
      ResolveAggregateFunctions ::
      TimeWindowing ::
      ResolveInlineTables(conf) ::
      ResolveHigherOrderFunctions(catalog) ::
      ResolveLambdaVariables(conf) ::
      ResolveTimeZone(conf) ::
      ResolveRandomSeed ::
      TypeCoercion.typeCoercionRules(conf) ++
      extendedResolutionRules : _*),
    Batch("Post-Hoc Resolution", Once, postHocResolutionRules: _*),
    Batch("Nondeterministic", Once,
      PullOutNondeterministic),
    Batch("UDF", Once,
      HandleNullInputsForUDF),
    Batch("FixNullability", Once,
      FixNullability),
    Batch("Subquery", Once,
      UpdateOuterReferences),
    Batch("Cleanup", fixedPoint,
      CleanupAliases)
  )
```

我们可以看到Hint的Batch里头有三个对象，一个是ResolveBroadcastHints，一个是ResolveCoalesceHints，最后一个是RemoveAllHints，我们来看一下源代码：

+ ResolveCoalesceHints
```
/**
   * COALESCE Hint accepts name "COALESCE" and "REPARTITION".
   * Its parameter includes a partition number.
   */
  object ResolveCoalesceHints extends Rule[LogicalPlan] {
    private val COALESCE_HINT_NAMES = Set("COALESCE", "REPARTITION")
    
    def apply(plan: LogicalPlan): LogicalPlan = plan.resolveOperators {
      // 判断为UnresolvedHint对象并且hintName为REPARTITION/COALESCE
      case h: UnresolvedHint if COALESCE_HINT_NAMES.contains(h.name.toUpperCase(Locale.ROOT)) =>
        // 获取的HintName
        val hintName = h.name.toUpperCase(Locale.ROOT)
        // 是否shuffle
        val shuffle = hintName match {
          case "REPARTITION" => true
          case "COALESCE" => false
        }
        // 分区数量
        val numPartitions = h.parameters match {
          case Seq(IntegerLiteral(numPartitions)) =>
            numPartitions
          case Seq(numPartitions: Int) =>
            numPartitions
          case _ =>
            throw new AnalysisException(s"$hintName Hint expects a partition number as parameter")
        }
        
        // 生成Repartition LogicalPlan对象
        Repartition(numPartitions, shuffle, h.child)
    }
  }
```

```scala
/**
   * Removes all the hints, used to remove invalid hints provided by the user.
   * This must be executed after all the other hint rules are executed.
   */
  object RemoveAllHints extends Rule[LogicalPlan] {
    def apply(plan: LogicalPlan): LogicalPlan = plan resolveOperatorsUp 	{
      // 直接将Dataset链进行返回，不生成任何LogicalPlan，因此为空操作
      case h: UnresolvedHint => h.child
    }
  }
```

```scala
/**
   * For broadcast hint, we accept "BROADCAST", "BROADCASTJOIN", and "MAPJOIN", and a sequence of
   * relation aliases can be specified in the hint. A broadcast hint plan node will be inserted
   * on top of any relation (that is not aliased differently), subquery, or common table expression
   * that match the specified name.
   *
   * The hint resolution works by recursively traversing down the query plan to find a relation or
   * subquery that matches one of the specified broadcast aliases. The traversal does not go past
   * beyond any existing broadcast hints, subquery aliases.
   *
   * This rule must happen before common table expressions.
   */
  class ResolveBroadcastHints(conf: SQLConf) extends Rule[LogicalPlan] {
    private val BROADCAST_HINT_NAMES = Set("BROADCAST", "BROADCASTJOIN", "MAPJOIN")

    def resolver: Resolver = conf.resolver

    private def applyBroadcastHint(plan: LogicalPlan, toBroadcast: Set[String]): LogicalPlan = {
      // Whether to continue recursing down the tree
      var recurse = true

      val newNode = CurrentOrigin.withOrigin(plan.origin) {
        plan match {
          // 将子Dataset进行BroadCast操作
          case u: UnresolvedRelation if toBroadcast.exists(resolver(_, u.tableIdentifier.table)) =>
            ResolvedHint(plan, HintInfo(broadcast = true))
          case r: SubqueryAlias if toBroadcast.exists(resolver(_, r.alias)) =>
            ResolvedHint(plan, HintInfo(broadcast = true))

          case _: ResolvedHint | _: View | _: With | _: SubqueryAlias =>
            // Don't traverse down these nodes.
            // For an existing broadcast hint, there is no point going down (if we do, we either
            // won't change the structure, or will introduce another broadcast hint that is useless.
            // The rest (view, with, subquery) indicates different scopes that we shouldn't traverse
            // down. Note that technically when this rule is executed, we haven't completed view
            // resolution yet and as a result the view part should be deadcode. I'm leaving it here
            // to be more future proof in case we change the view we do view resolution.
            recurse = false
            plan

          case _ =>
            plan
        }
      }

      if ((plan fastEquals newNode) && recurse) {
        newNode.mapChildren(child => applyBroadcastHint(child, toBroadcast))
      } else {
        newNode
      }
    }

    def apply(plan: LogicalPlan): LogicalPlan = plan resolveOperatorsUp {
      // 判断是否为BroadCastJoin操作
      case h: UnresolvedHint if BROADCAST_HINT_NAMES.contains(h.name.toUpperCase(Locale.ROOT)) =>
        // 如果参数为空，则将子Dataset进行broadcast
        if (h.parameters.isEmpty) {
          // If there is no table alias specified, turn the entire subtree into a BroadcastHint.
          ResolvedHint(h.child, HintInfo(broadcast = true))
        } else {
          // Otherwise, find within the subtree query plans that should be broadcasted.
          // 如果参数不为空时，则对表名的Dataset进行BroadCastJoin操作
          applyBroadcastHint(h.child, h.parameters.map {
            case tableName: String => tableName
            case tableId: UnresolvedAttribute => tableId.name
            case unsupported => throw new AnalysisException("Broadcast hint parameter should be " +
              s"an identifier or string but was $unsupported (${unsupported.getClass}")
          }.toSet)
        }
    }
  }
```

所以到这里SparkHint框架基本就结束了，因为SparkHint操作生成的是Unresolved级别的LogicalPlan对象，因此会在Analyzer层被相应的Batch捕捉并处理成别的LogicalPlan对象。

## 总结

SparkHint目前仅提供了BroadCastJoin和REPARTITION两种操作，其他的Hint语法都会被删除忽而略；我们可以在SQL中使用者两种语法来达到优化的目的。如果有什么写的不对的地方，请大家指点出来~








