---
title: C#动态表达式的运算与实战
description: 在实际开发中，动态表达式的运算能力极大提升了C#的灵活性，尤其在规则引擎、报表、动态查询等场景下尤为常见。本文将介绍C#动态表达式的常用实现方式、主流库、效率对比及典型应用。
date: 2025-08-24
slug: Net/2025-08-24
image: 2.jpg
categories:
    - C#
    - 动态表达式
tags:
    - C#
    - 动态表达式
---

# C# 动态表达式的运算与实战

在实际开发中，动态表达式的运算能力极大提升了C#的灵活性，尤其在规则引擎、报表、动态查询等场景下尤为常见。本文将介绍C#动态表达式的常用实现方式、主流库、效率对比及典型应用。

## 1. 动态表达式的应用场景
- **动态查询**：如根据用户输入拼接Where条件。
- **规则引擎**：业务规则可配置，运行时动态解析。
- **报表与公式计算**：支持自定义公式、字段运算。

## 2. 主要实现方式与效率对比

### 2.1 编译代码方式
通过动态生成C#代码并编译执行表达式，适合表达式不频繁变化的场景。

```csharp
using Microsoft.CSharp;
using System.CodeDom.Compiler;

string expression = "a > b && c < d";
if (!expression.Trim().StartsWith("return"))
    expression = "return " + expression + ";";
string className = "Expression";
string methodName = "Compute";
string source = $"using System;sealed class {className}{{public object {methodName}(){{{expression}}}}}";
CompilerParameters options = new CompilerParameters { GenerateInMemory = true, GenerateExecutable = false };
CompilerResults cr = new CSharpCodeProvider().CompileAssemblyFromSource(options, source);
object instance = cr.CompiledAssembly.CreateInstance(className);
var method = instance.GetType().GetMethod(methodName);
var result = method.Invoke(instance, null);
```

- **优点**：支持复杂表达式，灵活性高。
- **缺点**：每次都需编译，效率较低（200-300ms），适合缓存或预编译。

### 2.2 自定义表达式解析器
自己实现表达式解析和运算，灵活但开发成本高，适合特殊需求。

### 2.3 使用第三方表达式计算库

#### ExpressionEvaluator
- 使用方便，支持字符串表达式直接计算。
- 示例：
```csharp
var express = "(1 + 2) > 4";
var expression = new CompiledExpression(express);
var result = expression.Eval();
```
- **优点**：简单易用，速度较快。
- **缺点**：部分库已不维护，复杂表达式支持有限。

#### Z.Expressions.Eval
- 性能优秀，支持多种参数传递方式。
- 示例：
```csharp
using Z.Expressions;
int result = Eval.Execute<int>("X + Y", new { X = 1, Y = 2 });
// 位置参数
result = Eval.Execute<int>("{0} + {1}", 1, 2);
// 动态对象
dynamic expandoObject = new ExpandoObject();
expandoObject.X = 1;
expandoObject.Y = 2;
result = Eval.Execute<int>("X + Y", expandoObject);
// 字典参数
var values = new Dictionary<string, object>() { { "X", 1 }, { "Y", 2 } };
result = Eval.Execute<int>("X + Y", values);
```
- **优点**：速度快，功能强大，支持多种数据源。
- **缺点**：首次执行略慢，后续极快。

#### Dynamic LINQ
- 适合动态查询，字符串表达式转为LINQ。
- 示例：
```csharp
using System.Linq.Dynamic.Core;
var list = new List<int> {1, 5, 10, 20};
var result = list.AsQueryable().Where("it > 5").ToList();
```

#### Roslyn脚本引擎
- 支持完整C#脚本动态编译与执行，适合复杂表达式。
- 示例：
```csharp
using Microsoft.CodeAnalysis.CSharp.Scripting;
var result = await CSharpScript.EvaluateAsync<int>("1 + 2 * 3");
```

## 3. 性能与安全注意事项
- 表达式树和Z.Expressions.Eval性能较高，适合频繁调用。
- 动态编译和Roslyn功能强大但需注意沙箱和安全隔离。
- 动态表达式涉及用户输入时，务必做好校验防止注入风险。

## 4. 总结
C#动态表达式为业务开发带来极大灵活性。可根据场景选择动态编译、第三方库、Dynamic LINQ或Roslyn等方案，提升代码扩展性和可维护性。
![[image.png]]


> 参考：[动态执行用户输入表达式的策略与效率对比-CSDN博客](https://blog.csdn.net/qq_21162349/article/details/128645022)