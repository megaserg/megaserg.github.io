---
layout: post
title: "Static vs. dynamic typing"
tags: typing, ide, repl
---
Freedom doesn't scale

static languages simplify static analysis

claim: dynamic languages only good for small systems. say 1000 lines.

dynamic languages provide you freedom. you say: you don't need the compiler help, I can hold it in my head. but it doesnt scale

freedom is okay, but as a programmer you want correctness. the only way of provable correctness in mainstream is static typing
it's not guaranteed to work correct but the whole class of incorrectness is eliminated

example: number threaded through several functions.

without it, you can remember it. it increases cognitive load.

actually every time you use a variable, you have to remember its type

there is a limit of how much you can remember. also prone to forget

number of variables grows linearly. number of connections at least quadratically.

new hire: how to understand the type of variable in dyn lang? names, tests, iniitalization, otheruses

every new reader performs a static analysis in their head.

and work with hands instead of IDE (e.g. renaming)

-------
and it's fine! WHEN THE PROJECT IS SMALL.
so because of theese limits, when the size of projects grows to 10k, 100k lines - the development speed drops dramaticallu.

that's why it's good for prototyping. you can prototype fast. but know where to stop.

==========================================
dynamicist complain that getting the types correctly slows them down.
thye tend to use vim, emacs, sublime, atom, notepad++. these editors have nice features like syntax highlighting, you can install plugins and even get some autocompletion. but they have no idea of scoping, so you have irrelevant autocomplete suggestions. find usages is just a substring search, rename refactoring is a luxury, and there is afaik no support for pulling a block out of a method
and when you try to use static language in this environment... the editors don't help you. no wonder they complain! to them, it's no increase in productivity (because the editors lack the necessary features), moreover, it's a slowdown because you have to be verbose with all these types and signatures.
