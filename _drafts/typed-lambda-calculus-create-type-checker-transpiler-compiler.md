---
title: Type Checker for Extended Simply Typed Lambda Calculus
author: minko_gechev
layout: post
categories:
  - Programming languages
  - Lambda calculus
  - Type theory
  - Compilers
tags:
  - Programming languages
  - Lambda calculus
  - Type theory
  - Compilers
---

In this blog post we'll go through a sample implementation of a type checker, interpreter and a transpiler for a primitive purely functional programming language, which is based on the lambda calculus. The article is inspired by the book "[Types and Programming Languages](https://www.cis.upenn.edu/~bcpierce/tapl/)".

# Syntax

The syntax of the language is going to be quite simple.

We will have two types:

```
T ::= Int | Bool
```

As you can see we don't ave a syntactical construct for declaring type of a function (like we do in Haskell - `T1 → T2`, for instance). This is because we're going to apply type inference in order to guess the function type based on the type of the function's argument and body.

Our programs should belong to the language declared by the following grammar:

```
t ::=
  x   # variable
  λ x: T → t   # abstraction
  t t   # application
  true   # true literal
  false   # false literal
  if t then t else t   # conditional expression
  succ t   # returns the next natural number when applied to `t`
  prev t   # returns the previous natural number when applied to `t`
  iszero t # returns if the argument after evaluation equals `0`
  num   # a natural number
```

Where `num ∈ ℕ, num ≥ 0`. Note that there's no syntax construct for expressing negative numbers in our language. In order to be consistent the result of `pred 0` will return `0`.

# Semantics

Before going any further, lets show a few samples of the programming language that we're going to develop.

```
(λ a: Int → succ succ a) 0
```

In the example above we declare an anonymous function which accepts a single argument called `a` of type `Int`. In the body of the function we apply twice the predefined function `succ` to `a`. The anonymous function we apply to `0`, which is going to produce `2` as final result.

The equivalent of this program in JavaScript will be:

```
(a => succ(succ(a)))(0)
```

Where `succ` is defined as:

```
succ = a => a + 1
```

A more complicated program in our programming language will look like this:

```
succ (
  λ f: Int →
    (λ g: Int → g) 0
) 0
```

Lets explain the evaluation step by step:

1. Apply the `succ` predefined function to the result of the evaluation of the code within the parenthesis. The semantics of this code is:
2. Define an anonymous function which accepts a single argument of type `Int`, called `f`, and returns another anonymous function.
3. The second anonymous function has a single argument called `g` of type `Int`.
4. Apply the innermost function to `0`, which as result is going to produce `0`.
5. Pass `0` as argument to the outer function which will return `0` as result of the computation (the body of the outermost function evaluates to `0`).
6. Increment `0` and get `1`.

Formally, we can show the small-step semantics of our language in the following way:

## Small-step semantics

In this section I'll show the [small-step semantics](https://en.wikipedia.org/wiki/Operational_semantics#Small-step_semantics) of the language.

Lets suppose `σ` is the current state of our program, which keeps what value each individual variable has.

### Variables

```
1)
    ────────────────
    (x, σ) → σ(x)
```

1) simply means that for variable `x` we return its value kept in the state `σ`.

### Built-in functions

The built-in functions here are `succ` and `pred`. Here's their small-step semantics:

```
1)
        t1 → t2
    ─────────────────
    succ t1 → succ t2

2)
    pred 0 → 0

3)
    pred succ v → v

4)
        t1 → t2
    ─────────────────
    pred t1 → pred t2
```

1) simply means that if expression `t1`, passed to `succ` evaluates to `t2`, `succ` evaluates to `succ t2`.

2) we define that the result of `pred 0` equals `0` in order to keep the language consistent and disallow negative numbers not only syntactically but also as result of evaluation.

We're going to define `iszero` the following way:

```
1)
    iszero 0 → true

2)
          t1 → t2
    ────────────────────
    iszero t1 → iszero t2
```

This means that `iszero` applied to `0` returns `true`. If `t1` evaluates to `t2`, then `iszero t1` equals the result of the evaluation `iszero t2`.

### Conditional expressions

If the condition of the conditional expression is `true` then we return the expression from the `then` part, otherwise, we return the one from the `else` part.

```
1)
    if true then t2 else t3 → t1
    if false then t2 else t3 → t3
```

If given expression `t1` evaluates to `t*` and this expression is passed as condition of the conditional expression, the evaluation of the conditional expression equals to the evaluation of the conditional expression with `t*` passed as condition.

```
2)
                      t1 → t*
    ─────────────────────────────────────────────
    if t1 then t2 else t3 → if t* then t2 else t3
```

### Abstraction & Application

In this section we'll explain the function evaluation.

```
1)
    (λ x: T → t1) v2 → { x → v2 } t1

2)
        t1 → t*
    ────────────────
     t1 t2 → t* t2

3)
        t2 → t*
    ────────────────
      v1 t2 → t1 t*
```

1) means that if we have the abstraction `(λ x: T → t1)`, where `T` is the type of `x`, and apply it to `v2`, we need to substitute all the occurrences of `x` in `t1` with `v2`.

In 2) if `t1` evaluates to `t*`, `t1 t2` evaluates to `t*` applied to `t2`.

The semantics of 3) is that if `t2` evaluates to `t*`, `v1 t2` evaluates to `t1` applied to `t*`.

# Type Relations

Although the small-step semantics laws above are quite descriptive and by using them we already can build an evaluator for our programming language, we still can construct some ridiculous programs. For instance the following is invalid:

```
if 1 then true else 2
```

The condition of the conditional expression is expected to be of type boolean, however, above we pass a natural number. Anther problem with the expression is that the result of both branches of the expression should return result of the same time but this is not the case in our example.

In order to handle such invalid programs we can introduce a mechanism of program verification through **type checking**. This way, we will assign types to the individual constructs in our program and **as part of the compilation process**, verify if some properties of the program are hold or not.

Notice that **type checking will be performed compile-time**. The alternative is to provide runtime type checking, which will not prevent us from launching/writing invalid programs.

TBD

# Lexer and Parser

Although the implementation of a lexer and parser for this tiny language will be quite simple, we're going to generate them using PEG.js.

Here's the Peg grammar:

```
Application = l:ExprAbs r:Application* {
  if (!r || !r.length) {
    return l;
  } else {
    r = r.pop();
    return { type: 'application', left: l, right: r};
  }
};

ExprAbs = Expr / Abstraction;

Abstraction = _ '('? _ 'λ' _ id:Identifier ':' t:Type '→' f:Application _ ')'? _ {
  return { type: 'abstraction', arg: { type: t, id: id }, body: f };
}

Expr = IfThen / IsZeroCheck / ArithmeticOperation / Zero / True / False / Identifier / ParanExpression

ArithmeticOperation = o:Operator e:Application {
  return { type: 'arithmetic', operator: o, expression: e };
};

IsZeroCheck = IsZero e:Application {
  return { type: 'is_zero', expression: e };
}

Operator = Succ / Pred

IfThen = If expr:Application Then then:Application Else el:Application {
  return { type: 'conditional_expression', condition: expr, then: then, el: el };
}

Type =  Int / Bool

_  = [ \t\r\n]*

__ = [ \t\r\n]+

Identifier = !ReservedWord _ id:[a-z]+ _ {
  return { name: id.join(''), type: 'identifier' };
}

ParanExpression = _'('_ expr:Expr _')'_ {
  return expr;
}

True = _ 'true' _ {
  return { type: 'literal', value: true };
}

False = _ 'false' _ {
  return { type: 'literal', value: false };
}

Zero = _ '0' _ {
  return { type: 'literal', value: 0 };
}

ReservedWord = If / Then / Else / Pred / Succ / Int / Bool / IsZero / False

If = _'if'_

Then = _'then'_

Else = _'else'_

Pred = _'pred'_ {
  return 'pred';
}

Succ = _'succ'_ {
  return 'succ';
}

Int = _'Int'_ {
  return 'Int';
}

Bool = _'Bool'_ {
  return 'Bool';
}

IsZero = _'iszero'_ {
  return 'iszero';
}
```

Lets take a look at some specific terms:


### Application

```
Application = l:ExprAbs r:Application* {
  if (!r || !r.length) {
    return l;
  } else {
    r = r.pop();
    return { type: 'application', left: l, right: r};
  }
};
```

The function application `t t` from above can be expressed with this Peg rule. In general, we can have one expression or abstraction followed by 0 or more other applications.

We name the left term `l` and the right one `r`, after that, in case of a match, we return an object (AST node) with type `abstraction`, `left` and `right` branches.

Another interesting term is the abstraction:

```
Abstraction = _ '('? _ 'λ' _ id:Identifier ':' t:Type '→' f:Application _ ')'? _ {
  return { type: 'abstraction', arg: { type: t, id: id }, body: f };
}
```

We can see that it starts with an optional opening parenthesis, after that we have the lambda (`λ`) symbol, followed by the parameter declaration and its type. After that, we have arrow (`→`) and the abstraction's body, which is an application.

# Developing a Type Checker

# Developing an Interpreter

# Developing a Transpiler

# Conclusion
