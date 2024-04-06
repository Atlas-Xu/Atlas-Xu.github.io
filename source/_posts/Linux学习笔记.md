---
title: Linux学习笔记
tags:
  - Linux
abbrlink: b30ceb9f
date: 2024-02-27 00:34:00
---
最近在速通RHCE，也是顺便补一下Linux基础了，简单记录一下，有缘搭GitBook。
<!-- more -->
# Tools
- [tldr](https://tldr.inbrowser.app/)
# Vim
## Basic

### ESC Key

`Ctrl`-`[` is equivalent to `<Esc>`, but easier to press.

You can also rebind the key too (personal preference).

### Movement

`h`, `j`, `k`, `l`: to move around

### Exiting

`q`: **Q**uit without save

`w`: **W**rite

`wq`: **W**rite and **Q**uit

`x`: *Almost the same* as `wq`, but shorter.

### Useful settings

`:set number` `:set nu`: Display line number

`:set relativenumber` `:set rnu`: Display relative line number

## Motions

`motion := [count] <motion-keys>`

### Motion Keys

`h`, `j`, `k`, `l`: directions

`w`: for**w**ard a **w**ord

`W`: ignore punctuation

`b`: **b**ackward/**b**eginning of a word

`B`: ignore punctuation

`e`: **e**nd of a word

`E`: ignore punctuation

`^`: start of the line (regex)

`$`: end of the line (regex)

`f<letter>`: **f**ind to next `<letter>` (inclusive)

`F<letter>`: backwards

`t<letter>`: find un**t**il next `<letter>` (exclusive)

`T<letter>`: backwards

`/<pattern>`: search forward

`?<pattern>`: backwards

### Examples

`10j`: Move down 10 lines

`5w`: Move forward 5 words

`3b`: Move backward 3 words

`fx`: Find and move to next `x`

## Operators

`d`: **D**elete

`c`: **C**hange

`y`: **Y**ank (Copy)

## Duplicated Operator

`command := [count] <operator> <operator>`

**Duplicated operator operates on the current line.**

### Examples

`dd`: Delete current line

`3dd`: Delete 3 lines

`cc`: Change current line

`yy`: Yank/Copy current line

`3yy`: Yank/Copy 3 lines

## Capitalized Operator

`D` = `d$`

`C` = `c$`

`Y` = `y$`

## Operator on motion

`motion := [count] <motion-keys>`

`command := [count] <operator> <motion>`

### Examples

`d3w`: Delete 3 words

`d5l`: Delete 5 characters to the right

`3d3w`: Delete 3 words 3 times!

`d$`: Delete to the end of the line

## Text Objects

`text-object := [count] <modifier> <object-keys>`

### Modifiers

`a`: **A**round

`i`: **I**nner/**I**nside

### Objects

`a`: **A**rgument

`w`: **W**ord

`(` = `)` = `b`: Parenthesis (**b**rackets)

`{` = `}` = `B`: **B**races

`<` = `>`: Diamond brackets

`'`: Single quote

`"`: Double quote

`t`: **T**ag block (XML/HTML)

### Examples

`aw`: Around word

`iw`: Inner word

`a"`: Around double quoted string

`i"`: Inner double quoted string

## Operator on text objects

`text-object := [count] <modifier> <object-keys>`

`command := [count] <operator> <text-object>`

### Examples

`di"`: Delete inner double quoted string

`di'`: Delete inner single quoted string