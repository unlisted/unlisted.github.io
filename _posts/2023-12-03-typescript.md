---
layout: post
title:  "Typescript"
date: 2023-12-03
categories: code javascript typescript
---
Having conversations with startup founders, developers and technologists it's becoming clear that [TypeScript](https://www.typescriptlang.org/) is now the language of choice. It makes a ton of sense. With Javascript you can code up and down the stack and the addition of Typescript gives you type saftey and compiler assistance.

Python, my language of choice for the past few years, has a useful and maturing type system (See [PEP-484](https://peps.python.org/pep-0484/)). But it's optional and so are the static type checkers like [mypy](https://www.mypy-lang.org/). I find it super useful, you can even annotate [generic classes](https://peps.python.org/pep-0484/#instantiating-generic-classes-and-type-erasure). But it is optional. Also, Javascript is faster, if speed is what you're interested in. Although [3.11](https://docs.python.org/3/whatsnew/3.11.html) introduced some major speed improvements.

I've written Javascript sparingly, it was always something to be avoided in the earlier days. But with standardization via [ECMA-262](https://ecma-international.org/publications-and-standards/standards/ecma-262/), and the addition of Typescript JS seems to have matured and as mentioned above, become the language of choice. OK, it's clear, time to learn some. 

Typescripe compiles to Javascript and Bing Chat told me that it's better to learn JS first. When I learn something new I like to spend a little time running through docs and taking notes. Here's what I have for my 20-30 hour with JS [docs from MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)


>Interpreted,
free form,
object-oriented trough prototypes instead of class inheritance
Type-safe
Fast
>
ECMA-script standard defined in ECMA-264
DOM is not defined by ECMA, how javascript interacts with browser
Firefox engine is SpiderMonkey, Google Chrome engine is V8
>
Case sensitive and uses unicode encoding
Statements separated by semi-colon, not necessary for single line statements, best practice to always include
ASI - Automatic semi-colon Insertion https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#automatic_semicolon_insertion
>
>
var declares a value
let declares a block-scoped value
const declares a block-scoped read-only value
>
javascript identifier usually starts with _ or $
>
desctructering assignment
const { bar } = foo
>
without an initializer assigned the variable has undefined value
>
declaring a variable outside any function it is a global variable, inside a function its a local variable
var declared variables are hoisted
referencing let/const before declaration results in ReferenceError
function declarations are hoisted entirely
>
+  operator with string and int operands converts the int to string