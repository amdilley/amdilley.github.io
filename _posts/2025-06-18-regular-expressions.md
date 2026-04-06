---
layout:     post
title:      "Regular Expressions"
date:       2025-06-18 15:00:00 +0200
categories: regex
---

# Regular Expressions

## What Regex Is Good For

Regex is a way of identifying a class of strings according to a template. There are many times more than a strict equality check is needed. Regex can help. For instance, if we wanted to check to see if a string might be a phone number we could do:

```js
// is string in 555-555-5555 format
const isPhone = (str) => /^\d{3}-\d{3}-\d{4}$/.test(str);
```

Trying to achieve the above without regex is needlessly messy. Regex also makes modifying the template easier. Say we wanted our `isPhone` function to also recognize strings without the dashes:

```js
// is string in 555-555-5555/5555555555 format
const isPhone = (str) => /^\d{3}(-?)\d{3}\1\d{4}$/.test(str);
```

Say we wanted to relax the delimiter to be a period or a space. No problem:

```js
// is string in
// 555-555-5555/555.555.5555/555 555 5555/5555555555 format
const isPhone = (str) =>
  /^\d{3}([-.\s]?)\d{3}\1\d{4}$/.test(str);
```

Regex can also consolidate the number of passes over a given string. Here’s one example of a function that aims to extract the attribute name from a data attribute CSS pseudo-selector. The regex-free version:

```js
// [data-test_attribute] -> test_attribute
const getDataAttributeKey = (selector) =>
  selector
    .replace("[data-", "")
    .replace("]", "");
```

This works, but involves three passes over the input string. Now with regex:

```js
// [data-test_attribute] -> test_attribute
const getDataAttributeKey = (selector) =>
  selector.replace(/\[data-([^\]]+)\]/, "$1");
```

The regex isn’t easier to read, but we’ve cut down the number of passes from 2 to 1.

In general, the clearer the template the easier it is to write the regex. The following are good use cases for regex:
* substring existence check (no more `indexOf(str) !== -1`)
* short string pattern matching
* enum pattern matching (`/(red|blue|yellow)/`)
* multiple match parsing

## What Regex Is Not Good For

With great power… etc, etc, etc.

At first people tend to avoid regex because the syntax is difficult to read or write. However, the bigger problems often arise from those (like myself) who try to solve every problem with regex. And it’s easy to understand why that’s so tempting. Consider:

**Prime string length**
* `^(?!(..+)\1+$)`
* Matches: x, xx, xxx, xxxxx, xxxxxxx, ...
* Doesn’t match: xxxx, xxxxxx, xxxxxxxx, …

**Powers of 2**
* `^(?!((..)+.)\1*$)`
* Matches: x, xx, xxxx, xxxxxxxx, …
* Doesn’t match: xxx, xxxxx, xxxxxx, …

**Binary Divisibility by 3**
* `^(0|1(01*0)*1)*$`
* Matches: 0, 11, 110, 1001, ...
* Doesn’t match: 1, 10, 100, 101, 111, 1000, ...

But there are many instances where regex quickly falls apart. One of the more common cases is using regex to validate or match HTML. Under ideal circumstances this might not be so bad, but HTML is almost always malformed. The browser does a lot of heavy lifting to fill in the gaps when certain tags that should be closed aren’t. Or when tags contain certain unescaped characters.

Regex also struggles with larger texts spanning multiple lines. Consider the [catastrophic backtracking][catastrophic-backtracking] risk. Innocent assumptions can bring a regex engine to a crawl.

## Regex Readability and Testing

A common complaint levied against regex is how unreadable it is. There is no real counterargument. Regex can be parsed by those familiar with it, but there’s no guarantee you’ll be able to catch errors in the regex just by looking at it. For example, consider the RFC Standard regex for email validation:

```js
const EMAIL_REGEX = /(?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\])/;
```

No one should be expected to validate this on sight. Instead we should test these with known pass/fail cases:

```js
assert(
  EMAIL_REGEX.test("aaron@aarondilley.com"),
  true,
);

assert(
  EMAIL_REGEX.test("aaron@aarondilley@com"),
  false,
);
```

We should avoid writing a regex as complex as this from scratch where possible. But regardless of the origin, we should have coverage for all regex authored.

## Regex Basics

### Character ranges

* `[abc]` - matches any of `a`, `b`, or `c`
* `[^abc]` - matches any character not `a`, `b`, or `c`
* `[a-z]` - matches any lowercase character between `a` and `z`
* `[^a-z]` - matches any character not in the range from  `a` to `z`
* `[a-zA-Z]` - matchse any character between `a` and `z` or between `A` and `Z`

### Single tokens

* `.` - matches any character
* `\s` - matches any whitespace character (like a space)
* `\S` - matches any non-whitespace character
* `\d` - matches any digit, same as `[0-9]`
* `\D` - matches any non-digit, same as `[^0-9]`
* `\w` - matches any word character, same as `[a-zA-Z0-9_]`
* `\W` - matches any non-word character, same as `[^a-zA-Z0-9_]`
* `\n` - matches newline
* `\r` - matches carriage return
* `\t` - matches tab

### Group constructs

* `(…)` - matches anything in parens, reference-able later
* `(foo|bar)` - matches `foo` or `bar`

### Non-capturing group constructs

* `(?:...)` - matches anything in parens, not reference-able later
* `(?=...)` - positive lookahead
* `(?!...)` - negative lookahead
* `(?<=...)` - positive lookbehind
* `(?<!...)` - negative lookbehind

### Quantifiers

* `a?` - matches 0 or 1 `a`
* `a*` - matches 0 or more repeated `a`
  * greedy match
* `a*?` - matches 0 or more repeated `a` 
  * lazy match
* `a+` - matches 1 or more repeated `a`
* `a{3}` - matches `aaa`
* `a{3,}` - matches 3 or more repeated `a`
* `a{3,6}` - matches 3 to 6 repeated `a`

### Anchors

* `^` - starts with... (beginning of regex)
* `$` - ends with… (end of regex)

### Flags

* `/g` - global, find all instances
* `/i` - ignore case, no distinction between `[a-z]` and `[A-Z` ranges
* `/m` - multiline match, default is single line

## Regex Building

### Start with test cases

Always easiest to compile the cases the pattern *should* match along with the cases the pattern *should not* match. For example, if we have
```
# Matches
afoot
catfoot
dogfoot
fanfoot
foody
foolery
foolish
fooster
footage
foothot
footle
footpad
footway
hotfoot
jawfoot
mafoo
nonfood
padfoot
prefool
sfoot
unfool

# Doesn't match
Atlas
Aymoro
Iberic
Mahran
Ormazd
Silipan
altared
chandoo
crenel
crooked
fardo
folksy
forest
hebamic
idgah
manlike
marly
palazzi
sixfold
tarrock
unfold
```

We can probably surmise that the common thread in all the match cases is they have the substring `foo`. Therefore my regex is as easy as `/foo/`.

### Exercises

* Dates
  * `04/09/1987`
  * `4/9/1987`
  * `9 Apr 1987`
* HTML
  * span tags
    * `<span>Hello</span>`
    * `<span class="complex">Hello <em>there</em></span>`
  * anchor tag href
    * `<a class="plain-text" href="/about" data-event="click">About</a>`
  * secure anchor tags href
    * `<a class="plain-text" href="https://example.com" data-event="click">About</a>`
* Credit cards
  * Visa - starts with 4, 16 or 13 digits long
  * Mastercard - starts with 51 through 55 or 2221 through 2720, all 16 digits long
  * AmEx - starts with 34 or 37, all 15 digits long
  * Discover - starts with 6011 or 65, all 16 digits long

## FP Regex

Consider our earlier example

```js
// is string in 555-555-5555 format
const isPhone = (str) => /^\d{3}-\d{3}-\d{4}$/.test(str);
```

It would probably be pretty handy if we could eliminate the need to specify the `str` param. FP to the rescue:

```js
import { curry } from "lodash";
import { filter } from "lodash/fp";

const regexTest = curry((regex, str) => regex.test(str));

// is string in 555-555-5555 format
const isPhone = regexTest(/^\d{3}-\d{3}-\d{4}$/);

isPhone("555-555-5555"); // true

const onlyVowels = filter(regexTest(/^[aeiou]+$/i));

onlyVowels([
  "Hello",
  "EIEIO",
  "aaaaaa",
  "aaaaah",
]); // ["EIEIO", "aaaaaa"]
```

## Resources

* [Regex101][regex-101]
* [Multi-language word matching][multi-lang-word-matching]
* [Regex Golf][regex-golf]
* [Converting Deterministic Finite Automata to Regular Expressions][dfa-to-regex]

[catastrophic-backtracking]: https://vimeo.com/112065252
[regex-101]:                 https://regex101.com/
[multi-lang-word-matching]:  https://javascript.info/regexp-character-sets-and-ranges#example-multi-language-w
[regex-golf]:                https://alf.nu/RegexGolf
[dfa-to-regex]:              /assets/pdf/2005-03-16.DFA_to_RegEx.pdf

