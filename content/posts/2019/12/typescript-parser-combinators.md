---
title: "Let's Write Parser Combinators"
date: 2019-12-02T20:07:56-05:00
draft: true
tags: [parsers, typescript]
---

# Intro

Let's write a parser! That may sound like a tall order, but it's actually simpler than you'd think.
Parsers have been around forever, and there are many techniques to create them - you could write one by hand,
you could use a parser generator tool (like [yacc](http://dinosaur.compilertools.net/) or [jison](https://zaa.ch/jison/)),
or you could use a parser combinator library. If you've never heard that term before, that's fine - this is the approach
we'll be going with in this post (or series of posts, I haven't decided yet), so I'll explain what that means later on.

As we develop the core of the library, we'll also be building up an actual _parser_ which uses our library, namely a parser
for JSON. A JSON parser is relatively easy to write particularly because of its simplicity (the spec can 
[fit onto a business card](http://seriot.ch/json/json_business_card.png)). It's also pretty ubiquitous, and a great choice for
a first parser.

Let's get into it!

## Step 0: Setup

There's no shortage of javascript parser combinators out there (see [parsimmon](https://github.com/jneen/parsimmon),
[parjs](https://github.com/GregRos/parjs), or [bennu](http://bennu-js.com/) for some prior art), but we're going to
write ours in Typescript! Mostly because I'm a huge fan of static typing, but also because of the added challenge of
figuring out the definitions for some fairly complex types.

Start by creating the project directory: `mkdir <project_name> && cd <project_name>` (I called mine `part-and-parsel`,
but you can call it whatever you want) and initializing: `npm init -y`.

We're going to try and be extremely light on dependencies here, in fact we're only going to have 2, and they're dev
dependencies, so let's get them out of the way right now: `npm i -D typescript ts-node`. Then we can initialize the 
typescript project with `npx tsc --init`. This will create our `tsconfig.json` file, which we'll just leave as-is.

With the setup nearly done, we're almost ready to roll! Let's create three files, which will contain the entirety of our
project: `src/parser.ts`, `src/result.ts`, and `src/index.ts`.

We should have a structure that looks like this:
```
├── package-lock.json
├── package.json
├── src
│   ├── index.ts
│   ├── parser.ts
│   └── result.ts
└── tsconfig.json
```

If you've followed along, we're ready to go! Otherwise, feel free to clone 
[the repo](https://github.com/kengorab/part-and-parsel/tree/step-00-setup), at the `step-00-setup` tag.

### What is a Parser Anyway?

You can think of a `Parser` as a function which accepts code as input and produces some kind of data structure as output.
In fact, let's go ahead and define that using Typescript:
```ts
type Parser<T> = (input: string) => T
```
To those who may be unfamiliar with generic types, this is Typescript-speak for "a `Parser<T>` is a function which
accepts a `string` as input, and produces `T` as output". We say a `Parser` of `T` _produces_ a `T`, whatever we define
that type variable `T` to be. For example, this would be a totally valid parser:
```ts
const lameParser: Parser<string> = (input: string) => input
```
Sure, it didn't _do_ anything to the input, but technically it satisfies the type requirement. If it sounds like I'm 
harping on this a lot now, it's because later on we start doing some pretty crazy things with these `Parser`s, and
the best way to help the ensuing brain-melty-ness is to remember that a `Parser<T>` is a _function_ which accepts a
`string` as input and returns a `T`.

Cool, so we have the basic understanding of what a parser function is and what it returns, but simply returning some
parsed data structure isn't really enough for us. We're building a _parser combinator_ library, which means that we're
going to build a big parser function out of smaller parser functions. And those small parser functions will be made of
even smaller functions! These functions are going to need some way of passing around some kind of state, and we do that
through the return value. 

What kinds of things do parsers need to return? Well, we definitely need to return the `T` that the function was able to
parse from the input, but we also need to make sure we don't throw away the rest of the input that we haven't parsed yet.
Let's say I have a function which parses numbers, and I have an input `'1234 five'`; my `Parser<number>` function should
_definitely_ return the `1234`, but it also needs to return the remaining string `' five'`, so it can be passed along to
future parsers in the chain.

In Typescript, let's represent this with a pair, or a two-element array. We can use a type alias to ensure that this _must_
be a two-element array, with fixed positions:
```ts
type Pair<L, R> = [L, R]
type Parser<T> = (input: string) => [string, T]
```
So now our parser function accepts a `string` as input, and returns a pair consisting of the remaining value left in the
input and the `T` that it was able to parse out. This still isn't _really_ sufficient, since this currently doesn't leave us
with any way to surface errors, but let's come back to that. You didn't come here to listen to me drone on about type
definitions, you came here to see some parsing. For now, we'll just say that the parser will either return that tuple value
if parsing succeeds, or `null` if not. This can be expressed like this in typescript, and is the first two lines of
`src/parser.ts`:
```ts
type Pair<L, R> = [L, R]
export type Parser<T> = (input: string) => [string, T] | null
```

## Step 1: Parsing Null

The simplest value to start with when parsing JSON is the `null` value, so that's where we'll begin. In `src/index.ts`
let's add the following:
```ts
import { Parser } from './parser'

const parseNull: Parser<null> = (input: string) => null

const input = 'null'
console.log(parseNull(input))
```
If we run this file (`npx ts-node src/index.ts`), we should see an output of `null`. _Coool_. But of course that works,
and it _technically_ satisfies the contract of a `Parser<null>`, namely "a function which accepts a string and returns null".
But, what we really want is to _parse_ that input, meaning our parser function should only return `null` _if_ the input is
the string `'null'`. And in order to parse a string, we need to be able to parse a character. This is our very first
combinator, a parser which is capable of parsing only a single character!

Let's head over to `src/parser.ts` and add the scaffolding for our first parser function, which we'll call `pChar`. Since
combinator libraries rely on combining a lot of functions together, it's common to see these functions with relatively
short names. Let's take a look:
```ts
export function pChar(ch: string): Parser<string> {
  return input => {
    
  }
}
```
Even though we're using `string` here, keep in mind that we're talking about a single character (javascript, and 
therefore typescript, by extension) lacks an explicit type for a single character. But `pChar` is a function which
expects a single character, and returns a function which _parses_ that character. Don't be overwhelmed by the whole
functions-returning-functions thing, remember the type definition of `Parser<T>`.

For now, this doesn't typecheck since the returned parser function doesn't return a proper value. Let's fix this!
A parser that only parses a single character should succeed if the first character of the input matches that character,
and it should fail otherwise. Remember that we represent a failure with a `null` value, and a success with a tuple.
This is our function now, in its final form:
```ts
export function pChar(ch: string): Parser<string> {
  return input => {
    if (input.charAt(0) === ch) {
      return [input.substring(1), ch]
    }
    return null
  }
}
```

This works! And we can check it. In `src/parser.ts`, temporarily add the following and then run the file
(`npx ts-node src/parser.ts`):
```ts
const parseA = pChar('a')

console.log(parseA('a'))
console.log(parseA('b'))
```
We should see two lines output: `['', 'a']` first, and then `null` second. The first one succeeded, since its
first character matched the required character `'a'`. The second one failed because it didn't match. If we were to
do `console.log(parseA('aaa'))`, what do you think would be printed?

If you guessed `['aa', 'a']`, give yourself a gold star. If you didn't, remember that the first element returned in the
tuple is the _remaining_ string from the input. Our `pChar('a')` parser only parses the single character `'a'`, and only
the very first one that it sees! The rest are left up to other parsers to deal with. Which means that we're still pretty
far from being able to parse the string `'null'`, aren't we? Let's get into the real meat of this then, the idea of
_combining parsers together_.

### The `and` Combinator

How would we parse the string `'no'`? Well that's pretty simple: we'd want to parse the character `'n'`, followed by 
the character `'o'`. If we were given an input of `'na'`, the first character should parse correctly (since it's an `'n'`),
but the second should not, and the parser should fail. Same goes if we were given an input of `'to'`; the first parser
should fail because `'t' !== 'n'`. So it's important to have some mechanism of chaining parsing operations together! This
results in a new parser that succeeds if both the first and second succeed, and fails if either of them fail. Let's create
this appropriately-named `and` function now, in `src/parser.ts`:
```ts
export function and<L, R, V>(p1: Parser<L>, p2: Parser<R>, combiner: (l: L, r: R) => V): Parser<V> {
  return input => {
    
  }
}
```
First of all, there's a _lot_ going on in this type signature. If you're unfamiliar with Typescript and/or the concept of
generics, this is quite the mouthful (it is even if you _are_ familiar with these concepts, to be honest). But basically, 
all this is saying is given 2 parsers `p1` and `p2`, each of which produces some value (either `L` or `R`, respectively), 
`and` will return a new parser that produces some value of type `V`.
It's important to note that it's entirely possible for `L`, `R`, and `V` to all be the same type! But this type signature
allows for a lot of flexibility, and generics are a super powerful tool for expressing that flexibility.

There _is_ one additional parameter to this function though: `combiner`. Basically, `p1` and `p2` are parser functions, 
which are each going to parse out some value. Since our goal in `and` is to produce a _new_ parser which chains together
these two, this parser needs to know how those 2 values get combined together.

For example, if we were parsing the word `no`, we'd have `p1 = pChar('n')` and `p2 = pChar('o')`. If `p1` succeeded, it'd
return the successful value of `['o', 'n']` - the remaining value in the input (`'o'`), and the value that was successfully
parsed out (`'n'`). The remainder is then passed to `p2`, which will return a successful value of `['', 'o']`. Our mega-parser
which combines together `p1` and `p2` needs to return `['', 'no']` as its success value, so we need to combine together the
results of `p1` and `p2` in order to achieve that. 

_Why not just hard-code a `combiner` function of `(l: string, r: string) => l + r`?_, you might ask? Well, we're not certain
that `L` and `R` will always be `string`s! They could be arrays, or objects, or anything really! That's the beauty of 
generics - we get to write a function whose logic is generically applicable, and customize it from the outside depending on
the types involved in that particular case.

Let's encode the logic in the paragraph above, and have a completed `and` function:
```ts
export function and<L, R, V>(p1: Parser<L>, p2: Parser<R>, combiner: (l: L, r: R) => V): Parser<V> {
  return input => {
    const p1Result = p1(input)
    if (!p1Result) return null

    const [remainder1, p1Value] = p1Result

    const p2Result = p2(remainder1)
    if (!p2Result) return null

    const [remainder2, p2Value] = p2Result
    return [remainder2, combiner(p1Value, p2Value)]
  }
}
```
Sweet, what's going on here? So we already know that we're returning a new parser from this `and` function
(aka, returning a _function_ that accepts a `string` input and returns either `null` (in the case of an error) or a 
tuple (in the case of success)). This parser should succeed only if the two provided parsers `p1` and `p2` succeed. 
So, we run `p1` with the given input. If it fails, we're done and we return `null`. If it succeeds, we extract out 
the remainder and the parsed-out value. We pass this remainder into `p2`; if that fails we're done and we return 
`null`. If it succeeds, we again extract out the remainder and the parsed-out value. This second remainder represents
what's left of the input after both parsers have run, so it's this value that we must provide as the first
element of the tuple. The second element is the result of combining together the parsed-out values of `p1`
and `p2`, and luckily we have a handy function to help us with that!

Let's pull it all together; add this temporary code to `src/parser.ts` and run the file:
```ts
const parseN = pChar('n')
const parseO = pChar('o')
const parseNo = and(parseN, parseO, (l, r) => l + r)

console.log(parseNo('no'))
console.log(parseNo('na'))
console.log(parseNo('to'))
```
You can see that we create two tiny parsers, one for `'n'` and one for `'o'`. We then combine those two
parsers together using `and` to produce a parser that is capable of parsing not one, but _two_ characters, one after 
another!  Now that we're moving on up in the world, what do you expect the output to be? If you answered
```
[ '', 'no' ]
null
null
```
then you're in luck! If not, try and run through the examples in your head. This notion of passing functions
into other functions and returning a new function is only going to get more complex, so it's best to wrap
your head around it now. Work through a couple of examples, and when you feel that you understand what's
going on. If you feel like you can explain why `parseNo('notes')` returns `['tes', 'no']`, then you're good
to move on!

### The `str` Parser

Ok, so now we can create parsers for a single character, and we have a mechanism of chaining together smaller
parsers to make larger ones! We're ready for our next step, which is combining these two into a way to parse
entire strings of characters! It's time to create `pStr`:
```ts
export function pStr(str: string): Parser<string> {

}
```
I put this in `src/parser.ts`, below `pChar`. Much like `pChar`, this is a function which accepts accepts a
`string` and returns a `Parser<string>`, except here `string` is actually a string, rather than just a single
character.

So what does it mean to parse a string? Let's take our north star for example: `null`. Well, first we must parse
out an `n`, then a `u`, then an `l`, and finally another `l`. We _could_ write this as:
```ts
and(
  pChar('n'),
  and(
    pChar('u'),
    and(
      pChar('l'),
      pChar('l')
    )
  )
)
```
but of course it's ridiculous (and I left out the `combiner` parameter too!) to do this by hand. Computers _love_
doing repetitive tasks, so we should figure out a way to make it work for us. Really what we'd like to do is map
over all of the characters in a given string, create a `pChar` for each, and then chain them all together 
programmatically. Sounds like a great use-case for the `reduce` function! Let's take a look at the (perhaps 
surprisingly simple) `pStr` function:
```ts
export function pStr(str: string): Parser<string> {
  return str.split('')
    .map(pChar)
    .reduce((p1, p2) => and(p1, p2, (l, r) => l + r))
}
```
The very first thing you may notice is that we're not returning an arrow function here. In `pChar` and `and` we
did a `return input => { ... }`, but here we're just returning... _something_. Well, we're still returning a
`Parser<string>`, aren't we? Because otherwise Typescript wouldn't be happy. But of course we're returning a
`Parser<string>`, because we're using our `and` combinator! The `and` combinator _returns_ a function, and the
call to `reduce` is just folding together all of those `pChar` parsers in a nice convenient (and extensible) way.

Now we can rewrite our `parseNo` parser using our new `pStr` function, and run some temporary code once more:
```ts
const parseNo = pStr('no')

console.log(parseNo('no'))
console.log(parseNo('na'))
console.log(parseNo('to'))
```
We should still see the same output as before, which is a good thing! But now we can also parse a string of any
length:
```ts
const parseNote = pStr('note')

console.log(parseNote('note'))
console.log(parseNote('nate'))
console.log(parseNote('tote'))
```
This should of course produce the expected output:
```
[ '', 'note' ]
null
null
```
Alright, we are nearly there, I promise! It might not seem like we've accomplished a lot, but we've done a lot of
the legwork for what's to come, and considering we started from nothing I'd say we've gotten pretty far! We're
now capable of writing a parser that recognizes the input `null`. It will fail if it encounters any characters
that don't match, and it will succeed only if it sees `n`, `u`, `l`, `l`, in that order. But we do have a minor
problem, and it's a subtle one that might not make sense right away.

Right now we can write this:
```ts
const parseNull: Parser<null> = pStr('null')
```
but that doesn't actually work. Remember that the success value of `pStr` will be a tuple - the first element is
the remaining input left to parse, and the second input will be the string that we were able to parse. In this
case, for a valid input of say `"null123"`, `parseNull` would return `['123', 'null']`. This _doesn't quite_ match
up with what we want; a `Parser<T>` should return `T` as the second element in the tuple, and here we have a
`Parser<null>`. We're returning the _string_ `'null'`, but we really need to be returning the _value_ `null`.

*Do not* confuse this with the failure value for a parser, which is also `null`! If a `Parser<null>` succeeds, it
should return `['remaining input', null]`; if it fails it will return just `null`. The difference is subtle maybe,
but very important!

#### A Super-Quick Aside

So how can we fix this problem? We need some way of mapping the result returned by a parser from one value to another.
How would we do something like this in an array? We'd use the `Array.map` function, right?
```js
const numbers = [1, 2, 3]
const numbersAsStrings = numbers.map(n => `${n}`)  // ['1', '2', '3']
```
An array is a "container" of sorts, and the `.map` function allows us to modify the contents of that container. A
`Promise` works in kind of the same way! If we have a promise which contains a string, we can transform that string
_within_ the promise:
```js
const stringPromise = Promise.resolve('24')
const numberPromise = numberPromise.then(n => parseInt(n, 10))
```
At no point do we "open up" the promise (since we can't), we just use some function (here it's `.then`, but the
concept is the same) to modify the innards. What if we can do the same for our `Parser`? Turns out, that's exactly
what we want!

(Also if you're curious, the aforementioned "container" is actually called a _Functor_, which is a complex-sounding
word from the scary world of Functional Programming. Or _is_ it that scary?. After all, most of what we've been
doing so far could be considered "Functional Programming". If you're interested in learning more about _Functors_,
check out [this blog post](https://medium.com/@dtipson/functors-freedom-from-failure-f4abe6251b75) I found on Medium.)

### The `map` Function









