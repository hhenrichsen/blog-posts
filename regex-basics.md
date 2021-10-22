---
title: "Regex: The Basics"
date: 2021-10-21T12:00:00-06:00
draft: false
tags:
 - Programming
 - Regex
 - Computability
---

{{< mathjax >}}

## Introduction

Regex is an interesting language.

On one hand, it's a great way to recognize specific patterns of characters and
is one of the foundations of modern compilers. On the other hand, an oft-quoted
saying in programming is that "You have a problem. You use regex. You now have
two problems."

One of the main reasons behind this is because many people try to use regex to
parse entire languages, or structured languages that have existing parsers like
HTML/XML. This is not a good use of regex, since there are other tools that are
much better suited to that sort of problem.

## How Regex Works

Regex works by way of a state machine. Each character that it's supposed to
recognize is turned into a node that can transition to other states, or fail.
For example, if I wanted to recognize a string of the form "abc def", I'd need a
state machine that had 9 different states: a start, 5 intermediates, an
accepting state, and a rejecting "sink" state.

{{< blogsvg "regex-basics/automata.svg" >}}

There's a lot of notation here, but basically the idea is when you recieve an
input character that matches one of the characters on the arrows, you follow
that arrow. With a few simple operations on these state machines, you can
represent a lot of pretty complex patterns.

Let's start simple: we want to match the word hello. What's the regex for that?

```text {linenos=table} 
hello
```

Mindblowing, I know.

### Matching Multiple Times

Things get a bit more complicated when we allow loops, or want to draw arrows to
nodes that are further back in the graph. I'll save that for a different post
focusing on regex and state machines. But the fact of the matter is, if we can
loop around again to things we can pick up repeating patterns. What if we wanted
to allow any number of exclamation points after hello?

```text {linenos=table}
hello!*
```

That `*` is saying that the character before it can repeat 0 to infinity times,
in this case the exclamation point. That means each of the following inputs
succeed:

{{< output >}}
hello
hello!
hello!!!!!
{{< /output >}}

But things like `hi` or `hey` don't succeed, because they don't match each step
along the way.

### Optional Characters

What if we wanted only one exclamation? Then we'd use the `?` operation, which
makes the character before it as optional, so allowing 0 or 1.

```text
hello!?
```

{{< output >}}
hello
hello!
{{< /output >}}

### Matching Multiple Characters

What if we wanted to also accept `hallo`? 

```text {linenos=table}
h[ea]llo!*
```

The `[ea]` is saying that either `e` or `a` can successfully move onto the next
step, which means now we succeed on any of the following inputs:

{{< output >}}
hello
hallo
hello!
hallo!!!!
{{< /output >}}

This can be combined with the aforementioned `*` operation to allow any number
of `e`s and `a`s can be successful inputs:

```text {linenos=table}
h[ea]*llo!*
```

{{< output >}}
hello
heeeeeello!
haaaallo!!!!
heallo
hllo!!
{{< /output >}}

Those last two don't look right. Let's address the last one first.

### Matching one or more
So we know that we want one or more of the above letters, but the `*` that we
already have is zero or more. Allow me to introduce the `+` operation!

The `+` is the operator that means one or more, so maybe what we want is:

```text
h[ea]+llo!*
```

And here are our outputs:
{{< output >}}
hello
heeeeeello!
haaaallo!!!!
heallo
{{< /output >}}

And that still doesn't work. Bummer. Let's try something else.

### Groups and Alternation

Let's group those two together, so we can operate on them a little differently.

```text
h(e|a)+llo!*
```

This is currently nearly the same as the `[ea]`. The `()` group things together,
and then the `|` says that either option in the current group (or current
expression, if not inside a group) is acceptable. It's a little more
flexible, because we can change either side:

```text
h(e+|a+)llo!*
```

And here are our outputs:

{{< output >}}
hello
heeeeeello!
haaaallo!!!!
{{< /output >}}

Horray!

### Quantifiers

Say we only wanted to allow 1-5 `e`s or 2-4 `a`s. (I understand this is getting
a little silly at this point, but stay with me). We can use `{1,5}` instead of a
`+` on our `e`, and `{2,4}` instead of `+` on the `a`.

```text
h(e{1,5}|a{2,4})llo!*
```

This gives us these outputs:

{{< output >}}
hello
haaaallo!!!!
{{< /output >}}

This also means the the previous `?` could be represented as `{0,1}`.

### Character Ranges (Sets)

If we wanted to match any letter from a to z, that would get to be pretty
verbose, looking something like this:

```text
[abcdefghijklmnopqrstuvwxyz]
```

Instead, we're given character ranges, that let us represent this as a very easy
shorthand, `a-z`:

```text
[a-z]
```

This works on numbers, too, and cares about cases:

```text
[A-z]+
[0-9]+
```

### Escaping

As one might expect, you can also escape characters to allow matching on them:

```text
\([a-z]+\)
```

*This pattern matches any number of lowercase letters in parenthesis.*

### Example: Building a Simple US Phone Number Pattern

Let's start out simple with some options of what we want to try to match:

{{< output >}}
(123) 456-7890
123-456-7890
123 456-7890
{{< /output >}}

Let's start with the first three digits:

```text
(\([0-9]{3}\))|([0-9]{3})
```

This is a bit complicated, but it sets us up to match either (123) or 123, but
not a mix of the two. Next, let's work on the dash between the first and second
block of digits. We want a space, if we have the parenthesis, or either a space
and a dash afterwards:

```text
(\([0-9]{3}\) )|([0-9]{3}[ -])
```

Now for the second and third blocks of digits, combined by a dash:


```text
(\([0-9]{3}\) )|([0-9]{3}[ -])[0-9]{3}-[0-9]{4}
```

### Character Classes

It'd be a shame if we had to type out the [0-9] every time we wanted a digit, or
[A-z] every time we wanted a text character. That's why regex introduces us
short escape codes called Character Classes that represent common groups of
characters:

* `\d` is for digit, `[0-9]`.
* `\w` is for word, `[A-z0-9\-_]`
* `\s` is for space, `[ \n\r]`

### Negating Sets

If you instead prefer to match anything *but* what's in the braces, you can
negate a set with `[^...]`:

```text
[^0-4]{1,6}
```

This would match anything 1-6 characters long that didn't contain any digit from
0 to 4, inclusive.

### Negated Character Classes

Just as you can negate a set, you can use a capital letter to negate a character
class:


* `\D` is for digit, `[^0-9]`.
* `\W` is for word, `[^A-z0-9\-_]`
* `\S` is for space, `[^ \n\r]`

### Groups and Non-Capturing Groups

We've used groups already, but we should be aware that there are two types of
groups: capturing and non-capturing. Capturing groups are the parenthesis-based
groups that we've used before. Regex engines will allow us to pull specific
pieces out of a match based on the order of the capturing groups.

The other type of group is a non-capturing group, which still allows grouping
functionality without pulling anything into captured groups. These are started
with a few extra characters: `(?:...)`. Let's update our phone regex to capture
each group of digits, and nothing else:

```text
(?:(\([0-9]{3}\)) )|(?:([0-9]{3})[ -])([0-9]{3})-([0-9]{4})
```

This isn't as nice as it could be, but here are what we get back from some
inputs:

| Input            | Group 0          | Group 1   | Group 2   | Group 3   | Group 4   |
| ---------------- | ---------------- | --------- | --------- | --------- | --------- |
| `(123)-456-7890` | `<empty>`        | `<empty>` | `<empty>` | `<empty>` | `<empty>` |
| `(123) 456-7890` | `(123) 456-7890` | `123`     | `<empty>` | `456`     | `7890`    |
| `123 456-7890`   | `123 456-7890`   | `<empty>` | `123`     | `456`     | `7890`    |
| `123-456-7890`   | `123 456-7890`   | `<empty>` | `123`     | `456`     | `7890`    |

This makes it super easy to put things in a consisten format, even when they
might be entered in a non-standard format. These are often the most important
result of doing anything in regex, because they allow you to specify formats 
and then extract pertinent information.

## Conclusion

Hopefully this gives you the resources you need to start writing at least basic
regex. It can be a powerful tool if used in the right circumstances, and a 
powerful, high-caliber footgun if used incorrectly. Feel free to let me know 
your thoughts on {{< discord >}}.

## Other Resources

* [Regexr](https://regexr.com/) is incredibly useful. It lets you write and test
  your regex on a variety of text, and helps to explain and visualize it as you
  go. It was instrumental in creating this post.
