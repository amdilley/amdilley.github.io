---
layout:     post
title:      "Functional Programming in JavaScript"
date:       2025-02-08 17:00:00 +0200
categories: composition fp
---

# Functional Programming in JavaScript

## What is Functional Programming

Generally, Functional Programming (FP) is a coding style that centers 3 main paradigms:

* declarative flow
* immutability
* composition

## Declarative Programming

Declarative programming is a paradigm that relies on expressions that tell what is being computed as opposed to imperative programming that uses statements to tell how something needs to be computed with little clarity as to why. Consider the following example:

```js
// Imperative flow
function doubleNumbers(numbers) {
  const doubled = [];

  for (const n of numbers) {
    doubled.push(n * 2);
  }

  return doubled;
};

// Declarative flow
const doubleNumbers = numbers => numbers.map(n => n * 2);

// Declarative with ramda
const doubleNumbers = R.map(n => n * 2);
```

Here the imperative flow relies on temp variables, loops, actions, and a return all while obfuscating how this does what the function name suggests. Contrast with the declarative flows where we know we have an array of numbers and each number is mapped via the very self-explanatory lambda `n => n * 2`. We’ll continue to see declarative voice cuts down on code, makes code more readable, and the intent more easily known.

## Immutability

Immutability is the property of data structures not being able to be changed, or mutated, by the functions they’re passed to. Immutability ensures functions stay pure without any side effects, i.e. functions always return the same value given the same starting arguments without altering the global state around them. JavaScript does not enforce immutability at the language level, which means any steps should be taken to ensure this happens. Enter functional programming:

```js
// With mutation
function withoutKey(key, store) {
  delete store[key];

  return store;
};

// Immutability preserved
function withoutKey(key, store) {
  const {
    [key]: deleted,
    ...rest
  } = store;

  return rest;
};
```

The above example is particularly critical if the store in question is the DOM window object.

## Composition

One of the main benefits of JavaScript over other languages is that it features first-class functions. That is, functions can be assigned to variables or passed to other functions as arguments. In short functions act like any other assigned variable. Among other things this allows functions to be wrapped, or composed with one another. Consider the following example:

```js
const addOne = n => n + 1;
const double = n => n * 2;
const square = n => n * n;

// ((2 * n)^2) + 1 => 4n^2 + 1
const fourNSquaredPlusOne = n => addOne(square(double(n)));
```

Inline composition isn’t that readable and certainly becomes harder to parse as more functions are added to the composition. Libraries like [ramda][ramda-docs] address this with a function [compose][ramda-docs-compose] that can be used like this:

```js
const fourNSquaredPlusOne = R.compose(
  addOne,
  square,
  double,
);
```

However this can be a bit misleading since the evaluation happens inside out, or bottom up (right-ro-left if inline). The complementary [pipe][ramda-docs-pipe] function achieves the same outcome in a more intuitive order:

```js
const fourNSquaredPlusOne = R.pipe(
  double,
  square,
  addOne,
);
```

Under the hood pipe looks something like this:

```js
const pipe = (…fns) => x => fns.reduce((y, f) => f(y), x);
```

`y` here is the accumulator value, which is just the composition to this point. `f` is the next listed function which is simply applied to the resulting calculation of the accumulator. `x` is the starting value and represents the passed argument to the composed functions.

One of the many nice qualities of pipe is that it is self-flattening and self-composable. For example, these two representations are functionally equivalent:

```js
const fourNSquaredPlusOne = pipe(
  double,
  square,
  addOne,
);

const fourNSquaredPlusOne = pipe(
  double,
  pipe(
    square,
    addOne,
  ),
);
```

## Common recipes

### Getting

Objects in JavaScript are often deeply nested, but functions also can’t rely on nested paths existing. As such it’s very common to see something like this for getting:

```js
return foo?.bar?.biz;
```

This expression is actually quite compact compared to the legacy JavaScript equivalent:

```js
return foo && foo.bar && foo.bar.biz;
```

However if I want to *compose* this value with another operation I'd need an intermediate function like this:

```js
const getBiz = foo => foo?.bar?.biz;
```

Ramda simplifies this with its `path` function that takes the object and the desired path returning undefined by default if the path doesn’t exist:

```js
return R.path(['bar', 'biz'], foo);
```

This presentation is actually more complex than our `getBiz` function above. But ramda leverages *currying* to make `foo` an optional final parameter. When the data object is passed as the last argument the result is the value of `foo?.bar?.biz`. But without it the result is our `getBiz` function, flexible and composable.

```js
const foo = { bar: { biz: 4 } };

R.pipe(
  // foo
  //  ↳ path(['bar', 'biz'], foo)
  R.path(['bar', 'biz']),
  // foo
  //  ↳ path(['bar', 'biz'], foo)
  //    ↳ double(path(['bar', 'biz'], foo))
  double,
)(foo); //=> 8
```

### Setting

Updating values at a specified path is even more tedious as not only does that path need to be verified to exist, but for the sake of immutability nested spread operators are often used to create a mutation-free clone:

```js
if (foo?.bar?.biz) {
  return {
    ...foo,
    bar: {
      ...foo.bar,
      biz: 8,
    },
  };
}
```

Ramda comes to the rescue again with `set` (and `lensPath` which is similar to `path` but isn't concerned with *getting* at that path but rather *pointing* to that location):

```js
return R.set(R.lensPath(['bar', 'biz']), 8, foo);
```

Like `path`, `set` is configured for composition with the last argument optional. However the above doesn't read very well and it's annoying we need a separate utility function to handle this common operation. Ramda therefore provides us with `assocPath` to combine the getting and setting in a more compact function:

```js
const setBizTo8 = R.assocPath(['bar', 'biz'], 8);

setBizTo8(foo); //=> { bar: { biz: 8 } }
```

One common shortcoming of `assocPath` is that the new set value is independent of the existing value. Ramda provides a method `over` that handles just this.

```js
const squareCurrentBizValue = R.over(R.lensPath(['bar', 'biz']), square);

squareCurrentBizValue(foo); //=> { bar: { biz: 16 } }
```

We could simplify this further and create our own utility `overPath`:

```js
const overPath = R.curry(
  (path, fn, store) => R.over(R.lensPath(path), fn, store),
);

const squareCurrentBizValue = overPath(['bar', 'biz'], square);

squareCurrentBizValue(foo); // { bar: { biz: 16 } }
```

This is the second mention of currying, so we can talk a bit more about what this is now.

### Currying

Reusable, partially applied functions are a cornerstone of FP. Currying is a process that takes a function and allows its variables to be processed all at once or in chunks. Consider the following application:

```js
const sumTriple = R.curry(
  (a, b, c) => a + b + c,
);

// Following are equivalent
sumTriple(1, 2, 3); // 6
sumTriple(1, 2)(3); // 6
sumTriple(1)(2)(3); // 6
```

If we wanted to write these partials manually it would be a bit of a mess:

```js
function applyTwoArgs(a, b, fn) {
  return c => fn(a, b, c);
}

function applyOneArgAtATime(a, fn) {
  return b => c => fn(a, b, c);
}
```

`curry` takes a function `fn` and calculates `fn.arguments.length` to know when to return the calculated value as opposed to another partially applied function.

If creating custom FP functions, it’s best to wrap with `curry` for maximum flexibility.


### Placeholders

Sometimes when composing functions, the resolved and passed value isn’t the last accepted argument for the next function. Placeholders (denoted by the double underscore *__* in ramda) allow us to specify where the resolved value should be passed to the following function.

```js
const user = { name: 'Aaron', role: 'Software Engineer' };
const greet = R.replace('{{name}}', __, 'Hello {{name}}!');

const greetUser = R.pipe(
  R.prop('name'), //=> Aaron
  R.toUpper, //=> AARON
  greet, //=> Hello AARON! (AARON becomes second param where __ was)
);

greetUser(user); //=> Hello AARON!
```

## Examples

### Data adapters

Consider this adapter

```js
function adaptCareSymbols(careSymbols) {
  return (careSymbols ?? [])
    .filter(
      ({ id, name, image }) =>
        id !== undefined && name !== undefined && image !== undefined
    )
    .map(({ id, name, image }) => ({
      id,
      name,
      image,
    }));
}
```

The adapter filters a `careSymbols` array to only include items that have defined values for `id`, `image`, and `name` keys. Then only returns objects with those values.

We can rewrite this with ramda. We're going to try to return as much as we can inside our `pipe` function.

First we have

```js
(careSymbols ?? [])
```

In ramda, we have the function `or` which returns the first argument if it's truthy, otherwise, returns the second argument. So we want to do something like

```js
R.or(careSymbols, []),
```

But in the context of `pipe`, `careSymbols` would ordinarily be the last argument. We can use the placeholder to pull this into the right spot:

```js
R.pipe(
  // R.__ replaced with careSymbols once careSymbols is passed to pipe
  R.or(R.__, []),
)
```

Next we have

```js
.filter(
  ({ id, name, image }) =>
    id !== undefined && name !== undefined && image !== undefined
)
```

This is essentially three parts: `filter`, destructuring/selecting `id`, `image`, and `name` values, then checking none of these are `undefined`. The first part is straightforward. We can use ramda's `filter` function, and because this is functional programming, our filter callback will be wrapped in `pipe` (when in doubt, wrap with `pipe`). This looks like

```js
R.filter(R.pipe(
  ...
))
```

For destructuring/selecting `id`, `image`, and `name` values, ramda provides `R.props` which takes an array of key names and returns their values in an array in the same order. That is

```js
R.props(['id', 'image', 'name'], data) === [data.id, data.image, data.name]
```

Adding this to the filter callback above (dropping the `data` since it's passed automatically):

```js
R.filter(R.pipe(
  R.props(['id', 'image', 'name']),
  ...
))
```

Then we have to verify none of these resulting values are `undefined`. Ramda doesn't have any direct way to do this. If we wanted to check if *one* value was `undefined` we could use `R.equals(undefined)`. If we want to logically *negate* this assertion, that something is *not* `undefined` we can wrap with `R.complement`:

```js
R.complement(R.equals(undefined))
```

This isn't super readable inline with other code, so it's a good idea to set this as a standalone util function:

```js
const isDefined = R.complement(R.equals(undefined));
```

Since we need to ensure all the values return by `R.props` are defined, we can wrap with `R.all`:

```js
R.all(isDefined)
```

Now our filter component is complete:

```js
R.filter(R.pipe(
  R.props(['id', 'image', 'name']),
  R.all(isDefined),
))
```

Once our data is filtered, we want to map the filtered items to only return values for `id`, `image`, and `name`. We start with `R.map`. For the actual mapper, we can just use `R.pickAll(['id', 'image', 'name'])`. Together this is

```js
R.map(R.pickAll(['id', 'image', 'name']))
```

Altogether this looks like:

```js
const isDefined = R.complement(R.equals(undefined));

function adaptCareSymbols(careSymbols) {
  const definedKeys = ['id', 'image', 'name'];

  return R.pipe(
    // ensure we pass an empty array
    // if careSymbols is undefined
    R.or(R.__, []),
    R.filter(R.pipe(
      // grab values for each key in definedKeys
      R.props(definedKeys),
      // ensure all values are defined
      R.all(isDefined),
    )),
    // restrict data to only these key-values
    R.map(R.pickAll(definedKeys)),
  )(careSymbols);
}
```

But then we see we have this other function

```js
function adaptCareInstructions(careInstructions) {
  return (careInstructions ?? [])
    .filter(({ id, name }) => id !== undefined && name !== undefined)
    .map(({ id, name }) => ({
      id: id!,
      name: name!,
    }));
}
```

Instead of copy/pasting the inside of our modified `adaptCareSymbols` we can abstract to a shared factory:

```js
function pickAllDefined(definedKeys) {
  return R.pipe(
    R.or(R.__, []),
    R.filter(R.pipe(
      R.props(definedKeys),
      R.all(isDefined),
    )),
    R.map(R.pickAll(definedKeys)),
  );
}
```

Then we can write

```js
const adaptCareSymbols = pickAllDefined(['id', 'image', 'name']);
const adaptCareInstructions = pickAllDefined(['id', 'name']);
```

What if I have this function

```js
function adaptTechnicalInfos(technicalInfos) {
  return (technicalInfos ?? [])
    .filter(({ description }) => description !== undefined)
    .map(({ name, description }) => ({
      name,
      description,
    }));
}
```

Here you can see we only check for undefined against the `description` key but we want to map to include `description` and `name`. That's ok! We can add an optional second parameter to `pickAllDefined` for `outputKeys`. As a convenience we can have `definedKeys` as a default value:

```js
function pickAllDefined(definedKeys, outputKeys = definedKeys) {
  return R.pipe(
    R.or(R.__, []),
    R.filter(R.pipe(
      R.props(definedKeys),
      R.all(isDefined),
    )),
    R.map(R.pickAll(outputKeys)),
  );
}
```

Then our three functions look like

```js
const adaptCareSymbols = pickAllDefined(['id', 'image', 'name']);
const adaptCareInstructions = pickAllDefined(['id', 'name']);
const adaptTechnicalInfos = pickAllDefined(['description'], ['description', 'name']);
```

Of course we didn't have to use ramda or functional programming to write `pickAllDefined`, but by doing so we assess the function in terms of what it *does* and not what is does it *to*. The data target is removed entirely from the code.

### Analytics

The `pipe` function is great if we want to transform and then pass values on to subsequent functions. But what if we want to apply a function without interrupting the flow of the remaining functions? Consider the following utility function:

```js
const passThrough = curry(
  (fn, value) => {
    fn(value);
    return value;
  }
);
```

Here we want to apply a function, but we don't care about the returned value. Maybe we want to log the current value:

```js
const log = passThrough(console.log);
```

Now we can add this to `pipe` wherever we want a data snapshot:

```js
R.pipe(
  log, //=> 7
  double,
  log, //=> 14
  square,
  log, //=> 196
  addOne,
  log, //=> 197
)(7);
```

Maybe we want to track data as we go:

```js
const track = (adapter) => passThrough(
  R.pipe(adapter, trackWithConsent)
);

const dataWithTimestamp = (data) => ({
  data,
  time: Date.now(),
});

R.pipe(
  double, //=> 14
  square, //=> 196
  track(dataWithTimestamp), //=> trackWithConsent({ data: 196, time: 1739001045665 })
  addOne, //=> 197
)(7);
```

This is flexible and doesn't alter the end result.

## Final Thoughts

* Not everything needs the FP treatment
* Good FP is about maintainability above all else
* If FP conversion is too hard use smaller functions or rethink underlying data structure

## Additional References

* [The Hidden Treasures of Object Composition][the-hidden-treasures-of-object-composition] by Eric Elliot
* [Lenses][lenses] by Eric Elliott
* [Ramda documentation][ramda-docs]
* [Lodash FP guide][lodash-fp-docs]


[the-hidden-treasures-of-object-composition]: https://medium.com/javascript-scene/the-hidden-treasures-of-object-composition-60cd89480381
[lenses]:                                     https://medium.com/javascript-scene/lenses-b85976cb0534
[ramda-docs]:                                 https://ramdajs.com/
[ramda-docs-compose]:                         https://ramdajs.com/docs/#compose
[ramda-docs-pipe]:                            https://ramdajs.com/docs/#pipe
[lodash-fp-docs]:                             https://github.com/lodash/lodash/wiki/FP-Guide
