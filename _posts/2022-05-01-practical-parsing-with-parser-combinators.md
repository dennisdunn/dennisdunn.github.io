---
layout: post
title: Practical Parsing
subtitle: Level Up with Parser Combinators
image: practical-parsing.svg
comment: These are the speaker notes for a talk I planned to give at Stir Trek 2022
excerpt_separator: <!--more-->
---

We're going to be talking about parsing - what it is, how to do it, and why it's hard.<!--more--> Since we're at a software developers conference we won't talk about the parsing you might have done in middle school where you take a sentence in a **natural language** and break it up into different pieces - nouns, verbs, adjectives. We're going to take sentences in a **formal language** and break them up into parts like 'number,' 'term,' 'function call', and 'expression.'


We'll look at parsers which are functions that take an **input sentence** to produce a **data object** that reflects the structure of the input. The best definition that I've run across is


> A parser is a function that maps strings to things.


Because formal languages was not my best course at college and the first word in the title of this presentation is “Practical,” we won't spend too much time with theory, just enough to understand why parser combinators are so cool.

## Kevin's Problem

At the start of the pandemic lockdowns my friend Kevin was working with the Pennington County School District. They wanted to expand their home-school offerings and the first module they wanted to extend was the math module with the ability for the kids to type in arithmetic sentences and have them evaluated.


At first, Kevin used the tools he knew about and that had worked well for him in the past. His first attempt at solving the problem with regular expressions failed miserably. He would get to a certain point in parsing these math sentences and then couldn't get any further. He did what any competent software developer would do and he consulted the oracle, StackOverflow.


As it turns out, back in 1956 Noam Chomsky defined 4 types of formal languages. 


* Type 0 - Unrestricted
* Type 1 - Context sensitive
* Type 2 - Context free
* Type 3 - Regular

The major difference between these different types of formal language is the amount of information that needs to be retained to properly parse sentences in that language. An example of a Type 3 language is the comma-separated-values format; the data is 2-dimensional, without any nesting, and can be processed with a simple finite state automaton. I'm sure that most of you, like Kevin, have used regular expressions to process a CSV file. It's no coincidence that regular expressions are used to process regular languages!

The thing that stumped Kevin was that the sentences he was dealing with contained nested elements. It was like trying to parse an HTML file with regular expressions; what do you do when you come to an opening tag before you find the closing tag for the one you are currently on? Kevin discovered that he was dealing with a Type 2 context free language.

Kevin needed to level up his parsing skills!

## What's the solution?

There are plenty of tools to build parsers for context-free and context-sensitive languages such as

* ANTLR
* Bison
* Nearly.js

These tools use a domain-specific-language to describe the language that you want to parse and then use that description to generate the code for your parser. While they are powerful tools, they are also complicated tools and take time to learn.

Another option is to hand-craft your parser by first writing a lexer to transform your sentence into tokens and then a parser to construct your data object from the stream of tokens. This too can be time consuming and error prone.

We are going to use parser combinators to build a parser for Kevin's context-free language. A parser combinator is a higher-order function that takes one or more parsers as input and returns a new parser that is a combination of the input parsers.

## Introduction to Grammars

Before we do that, however, we need a blueprint to show us how sentences in our context-free language arrange their parts. That blueprint is known as a grammar. 

Here is a simple grammar written in Backus-Naur form for arithmetic expressions. Each line of the grammar is a production rule. The left side of the `::=` is a symbol and the right side is a rule for how to produce the thing on the left side. Non-terminals are the parts between the angle brackets and terminals are the characters such as the math operators and the open- and close- parentheses.

```
<exp> ::= <exp> + <term> | <exp> - <term> | <term>
<term> ::= <term> * <factor> | <term> / <factor> | <factor>
<factor> ::= ( <exp> ) | <number>
```

The first line says that an expression is an expression, a plus symbol, and a term OR an expression, a minus symbol, and a term OR a term. Using these rules we can break down a sentence like `1+2` into its parts.

Let's simplify the notation a little bit

```
Expr -> Expr + Term | Expr - Term | Term
Term -> Term * Factor | Term / Factor | Factor
Factor -> ( Expr ) | number
```

and introduce symbols for our character literals

```
sum -> + | -
product -> * | /
open -> (
close -> )
Expr -> Expr sum Term | Term
Term -> Term product Factor | Factor
Factor -> open Expr close | number
```

You might notice that we have a small problem. Expr can be produced by an Expr followed by a sum operator followed by Term. Our naive grammar is left recursive and before we can use it we must get rid of the recursion.

This time, Kevin consults the other oracle, Wikipedia. 

We have rules of the form:

![Eq 1](/images/practical-parsing/eq1.png)

We need to make some substitutions like this:

![Eq 2](/images/practical-parsing/eq2.svg)

![Eq 3](/images/practical-parsing/eq3.png)

That little **ε** is epsilon or the **empty-rule** and is the key for the transformation. It stops the production from recursively eating all of space and time. After all of these substitutions Kevins final blueprint looks like this

```
sum -> + | -
product -> * | /
open -> (
close -> )
Expr -> Term Expr′
Expr′ -> sum Term Expr′ | ε
Term -> Factor Term′
Term′ -> product Factor Term′ | ε
Factor -> open Expr close | number
```

Enough hand-waving - let's build a parser using parser combinators so you can see how powerful they really are!

## A Parser Combinator Library

We are going to build a parser combinator library called Tiny Parse for Kevin and he'll use it to build a parser for the simple arithmetic expressions that we just built the grammar for. 

If you want to dive right into it you can find the source code at:

https://github.com/dennisdunn/tiny-parse-js.git

It is also available as an npm package if you want to play around with the library. Note that this is not production-ready code, it's a pedantic tool so don't use it in anger! Just follow the instructions in the README to add Tiny Parse to your package.json.

## The problem of State

Remember that we are dealing with a Type 2 context-free language, that means that our parser will need to hold more state than
can be accomadated in a simple state machine. First, our input string will have a pointer to the next character to be read. We'll encapsulate the string and the pointer into a Stream class with seek(position), peek() and read() methods. Read() will return the next character from the stream and advance the pointer. Peek() returns the next character but does not advance the pointer. Seek() moves the pointer to the designated position and returns nothing.

Secondly, the nested nature of the processing will be tracked on the call stack of the functions making up the parser. We'll need to be aware of this as we build our parser or we will absolutely blow the stack on a simple input sentence. 

## Code the simplest thing possible

Our parser combinators are functions that combine … parsers. A parser is a function that takes an input string, in our case in the form of a Stream instance, and returns the portion of the string that the parser matched.

The simplest thing that we can parse is a single character like the mathematical operators of our grammar. If the next character in the sentence is the one we expected then return it to the caller and advance the position of the input stream. 

```javascript
char = c => stream =>  stream.peek() === c ? stream.read() : null; [^1]
```

Note that char() is not a parser but a parser generator, it takes some text and returns a parser for that text. This one generator allows us to create parsers for two of the rules of our grammar, namely the open and close parentheses.


* open = char('(') [^2]
* close = char(')')


To parse something like the sum terminal of our grammar we want to determine if the next character is one of a set of characters. We need a higher-order function that takes a couple of `char()` parsers and returns a parser.

```javascript
or = parsers => stream => {
        for(parser of parsers) {
                result = parser(stream)
                if(result) {
                        return result;
                }
        }
    return null;
}
```

Using just the `char()` generator and the `or()` combinator we can parse the math operator terminals of our grammar.

* sum = or(char('+'), char('-'))
* prod = or(char('*'), char('/'))

Albert Einstein is credited with the aphorism  “Make things as simple as possible, but no simpler.” We have a powerful programming language so let's use it accordingly. You will find that some parser combinator libraries actually use regular expressions as the basis for their parser generators.

The basic parser generators of our combinator library are `anyOfChar()` and `str()`. We've changed the signatures of the stream methods to accept the number or characters to peek or read.

```javascript
anyOfChar = text => stream => text.indexOf(stream.peek(1)) >= 0 
    ? stream.read(1) 
    : null;

str = text => stream => stream.peek(text.length) === text
    ? stream.read(text.length)
    : null;
```

So far, Kevin has come up with the following parsers for his grammar:
        
* sum = anyOfChar('+-')
* prod = anyOfChar('*/')
* open = str('(')
* close = str(')')

To create parsers for the number terminal and the non-terminals Kevin needs a few more combinators.

## Code the Essential Combinators

* Sequence
   * Try to match all of the parsers and return the results as an array.
* Many
   * Try to match one parser as many times as possible and return the results as an array. Can possibly return an empty array.
* Choice
   * Try to match each parser in sequence and return the first that succeeds.
* Optional
   * Try to match the parser, possibly returning null if there is no match.
* Between
   * Try to match each parser in sequence and return the result of the middle parser.
* Map
   * Try to match the parser and if successful, apply a function to the result.

## Parsing Numbers

First Kevin defines some parsers for the components of a number:

```javascript
digit = anyOfChar('0123456789')
digits = sequence(digit, many(digit))
sign = anyOfChar('+-')
fractional = sequence(str('.'), digits)
```

Then he combines them into a parser for numbers:

```javascript
number = sequence(optional(sign), digits, optional(fractional))
```

## Parsing the Non-Terminals

The grammar has five non-terminals. To build parsers for each non-terminal, Kevin maps directly from the grammar to the various combinators.

* Expr = sequence(Term, Expr_Prime) [^3]
* Expr_Prime = optional(sequence(sum, Term, Expr_Prime))
* Term = sequence(Factor, Term_Prime)
* Term_Prime = optional(sequence(product, Factor, Term_Prime))
* Factor = choice(number, sequence(open, Expr, close)) [^4]

The parser for Kevins Type 2 context-free language comes to ten lines of code that map directly to the grammar we looked at earlier and a few helper functions. So how does Kevin use his brand new parser?

## How To Use The Parser

If you want to follow along with the code examples in this section, clone https://github.com/dennisdunn/StirTrek2022-practical-parsing.git and follow the directions in the README.

### Parse Trees

The start symbol of the grammar, `G.start()`, gives us the function to call to parse our expressions. Calling `G.start('(1 + 2) * 3')` results in an array-of-arrays which is the parse tree of our expression. The parse tree contains all of the elements of the text with very little other structure.

### Token Stream

Let's tag the elements so that we know which part of the expression they represent. We'll take our terminal parsers and map the result to a function that tags the element. To do that, we'll use the `map()` combinator.

First, create a function that curries the token type argument of a token generator function:

```javascript
token = type => value => ({type, value})
```

Next, let's create some new parsers out of our existing parsers that return tokens instead of matched strings:
        
```javascript
open = map(open, token('open-paren'))
close = map(close, token('close-paren'))
sum = map(sum, token('sum-op'))
prod = map(prod, token('prod-op'))
number = map(number, token('number'))
```

If we do an inorder-traversal of the resulting tree we get a stream of tokens that can be passed to the shunting-yard algorithm for evaluation.

### Abstract Syntax Tree

Instead of creating tokens, let's create an abstract syntax tree of the source text by instantiating instances of an AST_Node class. Each terminal symbol of the grammar will have an associated AST_Node subclass and the non-terminals are handled by static methods on the AST_Node class. The class hierarchy is actually simpler than the token we used in the previous example because the sum and prod rules are both mapped to an AST_BinaryOp.

Recall that our grammar has non-terminals of the form Α and Α′. Α is referred to as the head and Α′ is called the tail. The map() combinator is used to apply the static head_handler() and tail_handler() methods to the result of a parse of the non-terminal to build the tree. The handlers will look at the parse tree returned by the parser and set the tree properties of the node to the correct values.

```javascript
map(sequence(Term, Expr_Prime), AST.head_handler)
map(optional(sequence(sop, Term, Expr_Prime)), AST.tail_handler)
```

Here is the result of parsing the expression (1 + 2) * 3

```javascript
AST_BinaryOp {
  value: '*',
  right: AST_Number { value: '3' },
  left: AST_BinaryOp {
    value: '+',
    right: AST_Number { value: '2' },
    left: AST_Number { value: '1' }
  }
}
```

### Evaluating the AST

Now that we have an AST, let's evaluate the tree to compute its value. Since we're such good software developers, we'll take the lazy way out and add an `eval()` method to each AST_Node subclass whose purpose is to evaluate that node. `AST_Number.eval()` converts its text content to a number. `AST_BinaryOp.eval()` applies the operator specified in its text content to the results of evaluating its operand properties.

```javascript
class AST_BinaryOp extends AST_Node {
    constructor(value) {
        super(value);
    }

    eval() {
        const operand_a = this.left.eval();
        const operand_b = this.right.eval();
        switch (this.value) {
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


Kevin went on to add other functionality to his grammar. First he added **modulo** and **power** binary operators to the grammar. Next he added unary operators like **factorial** as well as the trigonometric functions **sine**, **cosine**, and **tangent**.

If you use parser combinators to build your parsers and evaluators, the hardest part is getting the grammar correct. Building a parser generator that takes a grammar and uses a parser combinator library to build a parser is left as an exercise for the reader. :)

#### Footnotes

[^1]: Explain the difference between javascript function expressions and function definitions.

[^2]: Mention that `open()` and `close()` are functions that take a stream.

[^3]: If you look at the code in the repository you'll see that the non-terminal functions use javascript function definition syntax, not function expression syntax. This is because we have recursive functions and function definitions are hoisted.

[^4]: Note that we test for a number first. Remember that comment about blowing the stack? This is why.
