---
layout: post
title: The Roman Numerals kata
lead: Sandi Metz's <em>Make Everything the Same</em> post got me thinking about my own solution to the Roman Numerals kata and how it differed from so many others'
---

I recently read Sandi Metz's [Make Everything the Same](http://www.sandimetz.com/blog/2016/6/9/make-everything-the-same)
post and her words really resonated with me. I too had recently
completed the Roman Numerals kata in Elixir and, while I was happy with my
solution, I was struck by how different it was from most others.

### Special cases in disguise

Most solutions I explored had some flavor of this kind of configuration:

```elixir
@numerals [
    [1000, "M"],
    [ 900, "CM"],
    [ 500, "D"],
    [ 400, "CD"],
    [ 100, "C"],
    [  90, "XC"],
    [  50, "L"],
    [  40, "XL"],
    [  10, "X"],
    [   9, "IX"],
    [   5, "V"],
    [   4, "IV"],
    [   1, "I"]
  ]
```

There are two things that bother me about this: the configuration contains
duplication and it makes assumptions about the implementation that uses it.

The implementations that make use of this kind of configuration were indeed
short (it really only take [three more lines of code](https://github.com/exercism/xelixir/blob/master/exercises/roman-numerals/example.exs)) and fast, but I found
them to be opaque. When approaching the Roman Numerals kata, I tried to write
a program in the way my brain processes the conversion instead of a clever mathematical solution.
As Sandi says, code is read more than it's written—so I prefer a solutions
that are easily understandable.

To me, the only real needed configuration is this:

```elixir
@numerals [
    [1000, "M"],
    [ 500, "D"],
    [ 100, "C"],
    [  50, "L"],
    [  10, "X"],
    [   5, "V"],
    [   1, "I"]
  ]
```

The four and nine cases are just special combinations of these base symbols.
I'd argue that the first example contains *special cases*—they're just in
disguise.

### Make everything the same, functionally

The next step for me was to abstract the application of these symbols for any
given digit in a decimal number. A digit in a decimal number represents two things:
a numeral and a magnitude (a `3` in the ones' place is different than a `3` in the hundreds'
place). In Roman numeral notation the symbol changes at different magnitudes, but
the pattern is always the same. This pattern is what I chose to abstract.

```elixir
def template(digit, one, five, ten) do
  case digit do
  1 -> one
  2 -> one  <> one
  3 -> one  <> one <> one
  4 -> one  <> five
  5 -> five
  6 -> five <> one
  7 -> five <> one <> one
  8 -> five <> one <> one <> one
  9 -> one  <> ten
  _ -> ""
end
```

Give the template
a digit and the Roman symbols for that magnitude, and it gives you the Roman numeral
representation of that digit.

The magnitude here is simply the position of the Arabic numeral you're currently
processing:

```elixir
def magnitude(decimal_place) do
  :math.pow(10, decimal_place) |> round
end
```

And then that magnitude maps nicely onto our configuration to reveal the Roman symbols
the template is after.

```elixir
template(digit, @numerals[1 * magnitude],
                @numerals[5 * magnitude],
                @numerals[10 * magnitude])
```

Now yes, the `template` function contains the verboten `case` statement, but its intention here
is not to branch off to other behaviors in the code. It's simply formatting the supplied
numerals. To me at least, this *feels* different.

My most recent version is [this solution](https://gist.github.com/cdinger/9b6ac68c090fb2299afd1997061cfa56)
and I'm *mostly* happy with it. Though I'm already tempted to revisit
this kata. It'd be interesting to refactor away the `case` statement
altogether, which might teach me a new thing or two about Elixir in the process.
I suppose that's the point!
