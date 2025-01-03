---
layout: post
title: Practical Parsing
subtitle: Level Up with Parser Combinators
image: practical-parsing.svg
excerpt_separator: <!--more-->
---

We're going to be talking about parsing - what it is, how to do it, and why it's hard. <!--more--> We won't talk about the parsing you might have done in middle school where you take a sentence in a **natural language** and break it up into different pieces - nouns, verbs, adjectives. We're going to take sentences in a **formal language** and break them up into parts like 'number,' 'term,' 'function call', and 'expression.'


We'll look at parsers which are functions that take an **input sentence** to produce a **data object** that reflects the structure of the input. 


> A parser is a function that maps strings to things.


Because formal languages was not my best course at college and the first word in the title of this presentation is “Practical,” we won't spend too much time with theory, just enough to understand why parser combinators are so cool.

## A Parser Combinator Library

We are going to build a parser combinator library called _Tiny Parse_ and use it to build a parser for simple arithmetic expressions. 

```
sumOp -> + | -
prodOp -> * | /
openParen -> (
closeParen -> )
Expr -> Term Expr′
Expr′ -> sum Term Expr′ | ε
Term -> Factor Term′
Term′ -> product Factor Term′ | ε
Factor -> open Expr close | number
```

If you want to dive right into it you can find the source code at:

- [Tiny Parse C# project](https://github.com/dennisdunn/TinyParse-cs.git)

If you would like to see how I came up with the grammar, read:

- [So You Want To Parse Something - An Introduction to Grammars](https://dennisdunn.github.io/introduction-to-grammars/)

## The problem of State

Our arithmetic expressions are a Type 2 context-free language, that means that our parser will need to hold more state than
can be accommodated in a simple state machine. First, our input string will have a pointer to the next character to be read. We'll encapsulate the string and the pointer with an `IText` interface with `Seek(position)`, `Peek()` and `Read()` methods as well as a Position property. `Read()` will return the next character from the input and advances the pointer. `Peek()` returns the next  character but does not advance the pointer. `Seek(position)` moves the pointer to the designated position in the input and returns nothing. The `Position` property returns the current position of the pointer. 

Secondly, the nested nature of the processing will be tracked on the call stack of the functions making up the parser. We'll need to be aware of this as we build our parser or we will absolutely blow the stack on a simple input sentence. 

## Code the simplest thing possible

Our parser combinators are functions that combine … parsers. A parser is a function that takes an input string, in our case in the form of an `IText` instance, and returns the portion of the string that the parser matched.

The simplest thing that we can parse is a single character like the mathematical operators of our grammar. If the next character in the sentence is the one we expected then return it to the caller and advance the position of the input stream. 

```csharp
Parser Char(string expected)
{
    return text =>
    {
        return text.Peek() == expected
        ? text.Read()
        : throw new SyntaxError();
    };
}
```

Note that `Char()` is not a parser but a parser generator, it takes some text and returns a parser for that text. This one generator allows us to create parsers for two of the rules of our grammar, namely the open and close parentheses.


```csharp
Parser open => Char("(");
Parser close => Char(")");
```

To parse something like the sumOp terminal of our grammar we want to determine if the next character is one of a set of characters. We need a higher-order function that takes a couple of `Char()` parsers and returns a new parser.

```csharp
Parser Or(params Parser[] parsers) {
    return text => {
        var position = text.Position
        foreach(Parser parser in parsers) {
            try {
                return parser(text);
            } catch() {
                text.Seek(position);
            }
        }
        throw new SyntaxError(); 
    }
}
```

Using just the `Char()` generator and the `Or()` combinator we can parse the math operator terminals of our grammar.

```csharp
Parser sumOp = Or(Char("+"), Char("-"));
Parser prodOp = Or(Char("*"), Char("/"));
```

Albert Einstein is credited with the aphorism  _“Make things as simple as possible, but no simpler.”_ We have a powerful programming language so let's use it accordingly. You will find that some parser combinator libraries actually use regular expressions as the basis for their parser generators.

The basic parser generators of our combinator library are `AnyOf()` and `Str()`. We've changed the signatures of the IText methods to accept the number of characters to peek or read.

```csharp
Parser Str(string expected)
{
    return text =>
    {
        return text.Peek(expected.Length) == expected
        ? text.Read(expected.Length)
        : throw new SyntaxError();
    };
}
```

```csharp
Parser AnyOf(string expexted) {
    return text => {
        var str = text.Peek();
        return expected.Contains(str)
        ? text.Read()
        : throw new SyntaxError();
    }
}
```

So far, we have come up with the following parsers for this grammar:

```csharp
Parser SumOp = AnyOf("+-");
Parser ProdOp = AnyOf("*/");
Parser OpenParen = Str("(");
Parser CloseParen = Str(")");
```  

To create parsers for the number terminal and the non-terminals we need a few more combinators.

## The Essential Combinators

- Any 
    - Returns the first parser that succeeds.
- All
    - Returns a parser which matches all of the arguments in order.
- Many
    - Matches the argument 1 or more times.
- Optional
    - Matches the argument 0 or 1 time. Always succeeds,
    potentially returning `null` as a result.
- Ignore
    - Tries to match the argument and ignores any errors. 
    Always returns `null` as a result.
- Sequence
    - Matches each of the arguments and returns the results as a list.
- Apply
    - Tries to match the first argument and if successful, applies the second argument to the result.

## Parsing Numbers

First we define some parsers for the components of a number:

```csharp
Parser Digits =  Many(AnyOf("0123456789"));
Parser Sign = AnyOf("+-");
Parser Fractional = Sequence(Str("."), Digits);
```

Then we combine them into a parser for numbers:

```csharp
Parser Number = Sequence(Optional(Sign), Digits, Optional(Fractional))
```

## Parsing the Non-Terminals

The grammar has five non-terminals. To build parsers for each non-terminal, we map directly from the grammar to the various combinators. Because of the mutual recursion of the non-terminals, we'll structure
them as methods instead of delegates.

```csharp
dynamic Expr(IText text) => Sequence(Term, ExprPrime)(text);
dynamic ExprPrime(IText text) => Optional(Sequence(SumOp, Term, Expr_Prime))(text);
dynamic Term(IText text) => Sequence(Factor, TermPrime)(text);
dynamic TermPrime(IText text) => Optional(Sequence(ProdOp, Factor, Term_Prime))(text);
dynamic Factor(IText text) => Choice(Number, Seuence(OpenParen, Expr, CloseParen))(text);
```

The parser for our Type 2 context-free language comes to ten lines of code that map directly to the grammar we looked at earlier and a few helper functions. So how do we use this brand new parser?

## How To Use The Parser

If you want to follow along with the code examples in this section, clone https://github.com/dennisdunn/PracticalParsing-cs.git and follow the directions in the README.

### Parse Trees

The start symbol of the grammar, `G.Start()`, gives us the function to call to parse our expressions. Calling `G.Start('(1 + 2) * 3')` results in an array-of-arrays which is the parse tree of our expression. The parse tree contains all of the elements of the text with very little other structure.

### Token Stream

Let's tag the elements so that we know which part of the expression they represent. We'll take our terminal parsers and map the result to a function that tags the element. To do that, we'll use the `Apply()` combinator.

First, create a class to hold the token information:

```csharp
public class Token {
    public string Type { get; init; }
    public dynamic Value { get; init; }
}
```

Next, let's create some new parsers out of our existing parsers that return tokens instead of matched strings:
        
```csharp
Parser OpenParen = Apply(OpenParen,  v => new Token{ Type = "OpenParen", Value = v });
Parser CloseParen = Apply(CloseParen,  v => new Token{ Type = "CloseParen", Value = v });
Parser SumOp = Apply(SumOp,  v => new Token{ Type = "SumOp", Value = v });
Parser ProdOp = Apply(ProdOp,  v => new Token{ Type = "ProdOp", Value = v });
Parser Number = Apply(Number,  v => new Token{ Type = "Number", Value = v });
```

We didn't even need to change the non-terminal productions of the grammar to get a tree of tokens instead of a tree of strings. If we do an inorder-traversal of the token tree we get a stream of tokens that can be passed to the shunting-yard algorithm for evaluation.

### Abstract Syntax Tree

Instead of creating tokens, let's create an abstract syntax tree of the source text by instantiating instances of an AST_Node class. Each terminal symbol of the grammar will have an associated AST_Node subclass and the non-terminals are handled by static methods on the AST_Node class. The class hierarchy is actually simpler than the token we used in the previous example because the sum and prod rules are both mapped to an AST_BinaryOp.

Recall that our grammar has non-terminals of the form Α and Α′. Α is referred to as the head and Α′ is called the tail. The `Apply()` combinator is used to apply the static `HeadHandler()` and `TailHandler()` methods to the result of a parse of the non-terminal to build the tree. The handlers will look at the parse tree returned by the parser and set the properties of the node to the correct values.

```csharp
Apply(Sequence(Term, ExprPrime), AST.HeadHandler)
Apply(Optional(Sequence(SumOp, Term, Expr_Prime)), AST.TailHandler)
```

Here is the result of parsing the expression (1 + 2) * 3

```csharp
AST_BinaryOp {
  Value: '*',
  Right: AST_Number { Value: '3' },
  Left: AST_BinaryOp {
    Value: '+',
    Right: AST_Number { Value: '2' },
    Left: AST_Number { Value: '1' }
  }
}
```

### Evaluating the AST

Now that we have an AST, let's evaluate the tree to compute its value. Since we're such good software developers, we'll take the lazy way out and add an `Eval()` method to each `AST_Node` subclass whose purpose is to evaluate that node. `AST_Number.Eval()` converts its text content to a number. `AST_BinaryOp.Eval()` applies the operator specified in its text content to the results of evaluating its operand properties.

```csharp
public class AST_BinaryOp : AST_Node {

    Eval() {
        const operand_a = Left.Eval();
        const operand_b = Right.Eval();
        switch (Value) {
            case '+':
                return operand_a + operand_b;
            case '-':
                return operand_a - operand_b;
            case '*':
                return operand_a * operand_b;
            case '/':
                return operand_a / operand_b;
        }
    }
}
```

Pretty cool!

It is fairly easy to add other functionality to the grammar. Adding other binary operators such as **modulo** and **power** involves adding the relevant terminals and extending the `AST_BinaryOp.Eval()`. It is not much harder to add unary operators like **factorial** and trigonometric functions like **sine**, **cosine**, and **tangent**.

If you use parser combinators to build your parsers and evaluators, the hardest part is getting the grammar correct. Building a parser generator that takes a grammar and uses a parser combinator library to build a parser is left as an exercise for the reader. :)
