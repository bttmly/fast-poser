# fast-poser

## About This Fork

_(Doesn't look like a fork in GitHub so I could re-fork fast.js for other purposes)_

This fork allows you to use fast.js with [poser](). Instead of exporting the Fast constructor, fast-poser exports a function into which you pass `Ctor`, an Array-like constructor. Wherever fast.js top-level methods use `new Array()` or `[]`, fast-poser uses `new Ctor()`. 

```javascript
var poser = require('poser');
var Collection = poser('Array');
var Fast = require('fast-poser')(Collection);
```

If you call the exported function without an argument it'll use `Array`.

## fast.js

[![Build Status](https://travis-ci.org/nickb1080/fast-poser.svg?branch=master)](https://travis-ci.org/nickb1080/fast-poser)

Faster user-land reimplementations for several common builtin native JavaScript functions.

> Note: fast.js is very young and in active development. The current version is optimised for V8 (chrome / node.js) and may not perform well in other JavaScript engines, so you may not want to use it in the browser at this point. Please read the [caveats section](#caveats) before using fast.js.

## What?

Fast.js is a collection of micro-optimisations aimed at making writing very fast JavaScript programs easier. It includes fast replacements for several built-in native methods such as `.forEach`, `.map`, `.reduce` etc, as well as common utility methods such as `.clone`.

## Installation

Via [npm](https://npmjs.org/package/fast-poser):
```
npm install --save fast-poser
```

## Usage

```js
var fast = require('fast.js')();
console.log(fast.map([1,2,3], function (a) { return a * a; }));
```

## How?

Thanks to advances in JavaScript engines such as V8 there is essentially no performance difference between native functions and their JavaScript equivalents, providing the developer is willing to go the extra mile to write very fast code. In fact, native functions often have to cover complicated edge cases from the ECMAScript specification, which put them at a performance disadvantage.

An example of such an edge case is sparse arrays and the `.map`, `.reduce` and `.forEach` functions:

```js
var arr = new Array(100); // a sparse array with 100 slots

arr[20] = 'Hello World';

function logIt (item) {
  console.log(item);
}

arr.forEach(logIt);

```

In the above example, the `logIt` function will be called only once, despite there being 100 slots in the array. This is because 99 of those slots are empty. To implement this behavior according to spec, the native `forEach` function must check whether each slot in the array has ever been assigned or not (a simple `null` or `undefined` check is not sufficient), and if so, the `logIt` function will be called.

However, almost no one actually uses this pattern - sparse arrays are very rare in the real world. But the native function must still perform this check, just in case. If we ignore the concept of sparse arrays completely, and pretend that they don't exist, we can write a JavaScript function which comfortably beats the native version:

```js
var fast = require('fast.js')();

var arr = [1,2,3,4,5];

fast.forEach(arr, logIt); // faster than arr.forEach(logIt)
```


By optimising for the 99% use case, fast.js methods can be up to 5x faster than their native equivalents.

## Caveats

As mentioned above, fast.js does not conform 100% to the ECMAScript specification and is therefore not a drop in replacement 100% of the time. There are at least three scenarios where the behavior differs from the spec:

- Sparse arrays are not supported. A sparse array will be treated just like a normal array, with unpopulated slots containing `undefined` values. This means that iteration functions such as `.map()` and `.forEach()` will visit these empty slots, receiving `undefined` as an argument. This is in contrast to the native implementations where these unfilled slots will be skipped entirely by the iterators. In the real world, sparse arrays are very rare. This is evidenced by the very popular [underscore.js](http://underscorejs.org/)'s lack of support.

- Functions created using `fast.bind()` and `fast.partial()` are not identical to functions created by the native `Function.prototype.bind()`, specifically:

    - The partial implementation creates functions that do not have immutable "poison pill" caller and arguments properties that throw a TypeError upon get, set, or deletion.

    - The partial implementation creates functions that have a prototype property. (Proper bound functions have none.)

    - The partial implementation creates bound functions whose length property does not agree with that mandated by ECMA-262: it creates functions with length 0, while a full implementation, depending on the length of the target function and the number of pre-specified arguments, may return a non-zero length.

    > See the documentation for `Function.prototype.bind()` on [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind#Compatibility) for more details.


- The behavior of `fast.reduce()` differs from the native `Array.prototype.reduce()` in some important ways.

    - *Specifying* an `undefined` `initialValue` is the same as specifying no initial value at all. This differs from the spec which looks at the number of arguments specified. We just do a simple check for `undefined` which may lead to unexpected results in some circumstances - if you're relying on the normal behavior of reduce when an initial value is specified, make sure that that value is not `undefined`. You can usually use `null` as an alternative and `null` will not trigger this edge case.

    - A 4th argument is supported - `thisContext`, the context to bind the reducer function to. This is not present in the spec but is provided for convenience.


In practice, it's extremely unlikely that any of these caveats will have an impact on real world code. These constructs are extremely uncommon.

## Benchmarks

To run the benchmarks in node.js:

```
npm run bench
```

To run the benchmarks in SpiderMonkey, you must first download [js-shell](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey/Introduction_to_the_JavaScript_shell).
If you're on linux, this can be done by running:

```
npm run install-sm
```

This will download the [latest nightly build](http://ftp.mozilla.org/pub/mozilla.org/firefox/nightly/latest-trunk/) of the js-shell binary and extract it to [ci/environments/sm](./ci/environments/sm). If you're on mac or windows, you should download the appropriate build for your platform and place the extracted files in that directory.

After js-shell has been downloaded, you can run the SpiderMonkey benchmarks by running:


```
npm run bench-sm
```

### Most Recent Benchmark Output

```
> node ./bench/index.js

  Running 41 benchmarks, please wait...

  Native try {} catch (e) {} vs fast.try()
    ✓  try...catch x 65,972 ops/sec ±2.58% (75 runs sampled)
    ✓  fast.try() x 1,612,220 ops/sec ±3.63% (83 runs sampled)

    Result: fast.js is 2343.78% faster than try...catch.

  Native try {} catch (e) {} vs fast.try() (single function call)
    ✓  try...catch x 67,081 ops/sec ±2.73% (73 runs sampled)
    ✓  fast.try() x 1,948,507 ops/sec ±1.32% (93 runs sampled)

    Result: fast.js is 2804.71% faster than try...catch.

  Native .apply() vs fast.apply() (3 items, no context)
    ✓  Function::apply() x 11,039,599 ops/sec ±2.31% (90 runs sampled)
    ✓  fast.apply() x 11,934,275 ops/sec ±7.16% (82 runs sampled)

    Result: fast.js is 8.10% faster than Function::apply().

  Native .apply() vs fast.apply() (3 items, with context)
    ✓  Function::apply() x 7,444,772 ops/sec ±4.03% (77 runs sampled)
    ✓  fast.apply() x 10,526,781 ops/sec ±4.38% (77 runs sampled)

    Result: fast.js is 41.40% faster than Function::apply().

  Native .apply() vs fast.apply() (6 items, no context)
    ✓  Function::apply() x 9,502,453 ops/sec ±2.27% (85 runs sampled)
    ✓  fast.apply() x 13,005,394 ops/sec ±2.11% (84 runs sampled)

    Result: fast.js is 36.86% faster than Function::apply().

  Native .apply() vs fast.apply() (6 items, with context)
    ✓  Function::apply() x 7,743,741 ops/sec ±2.60% (84 runs sampled)
    ✓  fast.apply() x 11,601,222 ops/sec ±1.76% (87 runs sampled)

    Result: fast.js is 49.81% faster than Function::apply().

  Native .apply() vs fast.apply() (10 items, no context)
    ✓  Function::apply() x 6,756,873 ops/sec ±3.48% (82 runs sampled)
    ✓  fast.apply() x 6,676,832 ops/sec ±4.01% (85 runs sampled)

    Result: fast.js is 1.18% slower than Function::apply().

  Native .apply() vs fast.apply() (10 items, with context)
    ✓  Function::apply() x 7,154,866 ops/sec ±1.85% (85 runs sampled)
    ✓  fast.apply() x 6,176,639 ops/sec ±2.42% (87 runs sampled)

    Result: fast.js is 13.67% slower than Function::apply().

  fast.clone() vs underscore.clone() vs lodash.clone()
    ✓  fast.clone() x 527,022 ops/sec ±1.93% (88 runs sampled)
    ✓  underscore.clone() x 345,471 ops/sec ±1.30% (89 runs sampled)
    ✓  lodash.clone() x 239,981 ops/sec ±1.82% (86 runs sampled)

    Result: fast.js is 119.61% faster than lodash.clone().

  Native .indexOf() vs fast.indexOf() (3 items)
    ✓  Array::indexOf() x 10,777,714 ops/sec ±1.80% (87 runs sampled)
    ✓  fast.indexOf() x 14,304,867 ops/sec ±1.55% (87 runs sampled)
    ✓  fast.indexOf() v0.0.2 x 17,726,905 ops/sec ±1.92% (89 runs sampled)
    ✓  underscore.indexOf() x 7,768,668 ops/sec ±2.21% (85 runs sampled)
    ✓  lodash.indexOf() x 11,537,650 ops/sec ±2.11% (87 runs sampled)

    Result: fast.js is 32.73% faster than Array::indexOf().

  Native .indexOf() vs fast.indexOf() (10 items)
    ✓  Array::indexOf() x 7,633,019 ops/sec ±2.08% (89 runs sampled)
    ✓  fast.indexOf() x 10,311,573 ops/sec ±1.32% (93 runs sampled)
    ✓  fast.indexOf() v0.0.2 x 11,599,637 ops/sec ±1.83% (92 runs sampled)
    ✓  underscore.indexOf() x 6,659,299 ops/sec ±1.41% (94 runs sampled)
    ✓  lodash.indexOf() x 8,469,026 ops/sec ±1.16% (92 runs sampled)

    Result: fast.js is 35.09% faster than Array::indexOf().

  Native .indexOf() vs fast.indexOf() (1000 items)
    ✓  Array::indexOf() x 231,753 ops/sec ±1.30% (88 runs sampled)
    ✓  fast.indexOf() x 225,856 ops/sec ±1.15% (90 runs sampled)
    ✓  fast.indexOf() v0.0.2 x 235,863 ops/sec ±1.31% (88 runs sampled)
    ✓  underscore.indexOf() x 228,972 ops/sec ±1.33% (90 runs sampled)
    ✓  lodash.indexOf() x 206,821 ops/sec ±1.41% (90 runs sampled)

    Result: fast.js is 2.54% slower than Array::indexOf().

  Native .lastIndexOf() vs fast.lastIndexOf() (3 items)
    ✓  Array::lastIndexOf() x 21,028,189 ops/sec ±1.31% (92 runs sampled)
    ✓  fast.lastIndexOf() x 26,750,756 ops/sec ±1.49% (89 runs sampled)
    ✓  fast.lastIndexOf() v0.0.2 x 32,928,926 ops/sec ±1.52% (85 runs sampled)
    ✓  underscore.lastIndexOf() x 15,310,544 ops/sec ±1.48% (92 runs sampled)
    ✓  lodash.lastIndexOf() x 25,633,291 ops/sec ±1.16% (92 runs sampled)

    Result: fast.js is 27.21% faster than Array::lastIndexOf().

  Native .lastIndexOf() vs fast.lastIndexOf() (10 items)
    ✓  Array::lastIndexOf() x 10,267,638 ops/sec ±1.57% (93 runs sampled)
    ✓  fast.lastIndexOf() x 15,180,947 ops/sec ±2.95% (91 runs sampled)
    ✓  fast.lastIndexOf() v0.0.2 x 17,918,562 ops/sec ±2.83% (83 runs sampled)
    ✓  underscore.lastIndexOf() x 7,781,747 ops/sec ±1.46% (89 runs sampled)
    ✓  lodash.lastIndexOf() x 14,446,876 ops/sec ±3.07% (87 runs sampled)

    Result: fast.js is 47.85% faster than Array::lastIndexOf().

  Native .lastIndexOf() vs fast.lastIndexOf() (1000 items)
    ✓  Array::lastIndexOf() x 501,391 ops/sec ±2.38% (89 runs sampled)
    ✓  fast.lastIndexOf() x 602,888 ops/sec ±1.20% (93 runs sampled)
    ✓  fast.lastIndexOf() v0.0.2 x 608,585 ops/sec ±1.42% (93 runs sampled)
    ✓  underscore.lastIndexOf() x 504,521 ops/sec ±1.33% (90 runs sampled)
    ✓  lodash.lastIndexOf() x 515,004 ops/sec ±2.20% (90 runs sampled)

    Result: fast.js is 20.24% faster than Array::lastIndexOf().

  Native .bind() vs fast.bind()
    ✓  Function::bind() x 522,291 ops/sec ±2.71% (84 runs sampled)
    ✓  fast.bind() x 4,604,123 ops/sec ±1.41% (88 runs sampled)
    ✓  fast.bind() v0.0.2 x 4,013,722 ops/sec ±1.06% (91 runs sampled)
    ✓  underscore.bind() x 305,811 ops/sec ±1.94% (86 runs sampled)
    ✓  lodash.bind() x 223,603 ops/sec ±1.79% (76 runs sampled)

    Result: fast.js is 781.52% faster than Function::bind().

  Native .bind() vs fast.bind() with prebound functions
    ✓  Function::bind() x 2,808,338 ops/sec ±1.13% (87 runs sampled)
    ✓  fast.bind() x 12,305,166 ops/sec ±1.66% (91 runs sampled)
    ✓  fast.bind() v0.0.2 x 9,216,655 ops/sec ±2.14% (91 runs sampled)
    ✓  underscore.bind() x 2,819,067 ops/sec ±1.25% (93 runs sampled)
    ✓  lodash.bind() x 3,119,670 ops/sec ±1.33% (92 runs sampled)

    Result: fast.js is 338.17% faster than Function::bind().

  Native .bind() vs fast.partial()
    ✓  Function::bind() x 476,066 ops/sec ±3.21% (71 runs sampled)
    ✓  fast.partial() x 4,570,669 ops/sec ±2.90% (88 runs sampled)
    ✓  fast.partial() v0.0.2 x 4,191,368 ops/sec ±3.72% (91 runs sampled)
    ✓  fast.partial() v0.0.0 x 4,397,700 ops/sec ±1.19% (91 runs sampled)
    ✓  underscore.partial() x 858,560 ops/sec ±1.56% (94 runs sampled)
    ✓  lodash.partial() x 215,827 ops/sec ±1.89% (74 runs sampled)

    Result: fast.js is 860.09% faster than Function::bind().

  Native .bind() vs fast.partial() with prebound functions
    ✓  Function::bind() x 2,668,208 ops/sec ±1.64% (93 runs sampled)
    ✓  fast.partial() x 9,783,559 ops/sec ±10.26% (72 runs sampled)
    ✓  fast.partial() v0.0.2 x 7,317,568 ops/sec ±4.70% (78 runs sampled)
    ✓  fast.partial() v0.0.0 x 8,269,752 ops/sec ±6.43% (83 runs sampled)
    ✓  underscore.partial() x 4,175,763 ops/sec ±2.73% (92 runs sampled)
    ✓  lodash.partial() x 3,102,533 ops/sec ±1.11% (96 runs sampled)

    Result: fast.js is 266.67% faster than Function::bind().

  Native .map() vs fast.map() (3 items)
    ✓  Array::map() x 1,281,667 ops/sec ±1.89% (86 runs sampled)
    ✓  fast.map() x 7,015,926 ops/sec ±1.52% (91 runs sampled)
    ✓  fast.map() v0.0.2a x 6,677,604 ops/sec ±1.19% (93 runs sampled)
    ✓  fast.map() v0.0.1 x 6,605,295 ops/sec ±1.51% (92 runs sampled)
    ✓  fast.map() v0.0.0 x 5,865,050 ops/sec ±2.69% (85 runs sampled)
    ✓  underscore.map() x 1,062,254 ops/sec ±3.81% (88 runs sampled)
    ✓  lodash.map() x 4,426,603 ops/sec ±4.46% (90 runs sampled)

    Result: fast.js is 447.41% faster than Array::map().

  Native .map() vs fast.map() (10 items)
    ✓  Array::map() x 617,891 ops/sec ±6.92% (72 runs sampled)
    ✓  fast.map() x 2,619,135 ops/sec ±4.73% (81 runs sampled)
    ✓  fast.map() v0.0.2a x 2,636,916 ops/sec ±3.52% (84 runs sampled)
    ✓  fast.map() v0.0.1 x 2,674,765 ops/sec ±3.85% (86 runs sampled)
    ✓  fast.map() v0.0.0 x 2,250,683 ops/sec ±4.24% (83 runs sampled)
    ✓  underscore.map() x 685,888 ops/sec ±4.10% (71 runs sampled)
    ✓  lodash.map() x 2,410,105 ops/sec ±1.63% (92 runs sampled)

    Result: fast.js is 323.88% faster than Array::map().

  Native .map() vs fast.map() (1000 items)
    ✓  Array::map() x 17,096 ops/sec ±1.70% (93 runs sampled)
    ✓  fast.map() x 41,387 ops/sec ±1.24% (93 runs sampled)
    ✓  fast.map() v0.0.2a x 40,289 ops/sec ±1.46% (93 runs sampled)
    ✓  fast.map() v0.0.1 x 39,521 ops/sec ±1.45% (91 runs sampled)
    ✓  fast.map() v0.0.0 x 32,164 ops/sec ±1.10% (90 runs sampled)
    ✓  underscore.map() x 17,425 ops/sec ±1.09% (91 runs sampled)
    ✓  lodash.map() x 37,155 ops/sec ±0.92% (90 runs sampled)

    Result: fast.js is 142.09% faster than Array::map().

  Native .filter() vs fast.filter() (3 items)
    ✓  Array::filter() x 1,137,679 ops/sec ±2.00% (81 runs sampled)
    ✓  fast.filter() x 4,083,210 ops/sec ±1.70% (91 runs sampled)
    ✓  underscore.filter() x 1,031,243 ops/sec ±1.38% (87 runs sampled)
    ✓  lodash.filter() x 3,310,774 ops/sec ±2.14% (90 runs sampled)

    Result: fast.js is 258.91% faster than Array::filter().

  Native .filter() vs fast.filter() (10 items)
    ✓  Array::filter() x 585,374 ops/sec ±2.79% (79 runs sampled)
    ✓  fast.filter() x 1,845,855 ops/sec ±1.52% (90 runs sampled)
    ✓  underscore.filter() x 564,617 ops/sec ±2.28% (83 runs sampled)
    ✓  lodash.filter() x 1,502,759 ops/sec ±5.42% (82 runs sampled)

    Result: fast.js is 215.33% faster than Array::filter().

  Native .filter() vs fast.filter() (1000 items)
    ✓  Array::filter() x 11,478 ops/sec ±4.13% (84 runs sampled)
    ✓  fast.filter() x 23,063 ops/sec ±1.24% (92 runs sampled)
    ✓  underscore.filter() x 12,533 ops/sec ±1.72% (92 runs sampled)
    ✓  lodash.filter() x 22,778 ops/sec ±1.25% (89 runs sampled)

    Result: fast.js is 100.94% faster than Array::filter().

  Native .reduce() vs fast.reduce() (3 items)
    ✓  Array::reduce() x 2,776,441 ops/sec ±1.54% (89 runs sampled)
    ✓  fast.reduce() x 7,549,862 ops/sec ±2.52% (90 runs sampled)
    ✓  fast.reduce() v0.0.2c x 3,637,143 ops/sec ±2.39% (90 runs sampled)
    ✓  fast.reduce() v0.0.2b x 8,382,726 ops/sec ±1.74% (90 runs sampled)
    ✓  fast.reduce() v0.0.2a x 7,565,424 ops/sec ±1.95% (91 runs sampled)
    ✓  fast.reduce() v0.0.1 x 7,540,612 ops/sec ±1.55% (91 runs sampled)
    ✓  fast.reduce() v0.0.0 x 7,263,109 ops/sec ±1.57% (94 runs sampled)
    ✓  underscore.reduce() x 2,196,257 ops/sec ±1.32% (93 runs sampled)
    ✓  lodash.reduce() x 3,883,037 ops/sec ±1.40% (89 runs sampled)

    Result: fast.js is 171.93% faster than Array::reduce().

  Native .reduce() vs fast.reduce() (10 items)
    ✓  Array::reduce() x 1,330,405 ops/sec ±1.86% (95 runs sampled)
    ✓  fast.reduce() x 3,419,496 ops/sec ±1.21% (95 runs sampled)
    ✓  fast.reduce() v0.0.2c x 1,762,393 ops/sec ±1.14% (92 runs sampled)
    ✓  fast.reduce() v0.0.2b x 3,557,156 ops/sec ±0.89% (93 runs sampled)
    ✓  fast.reduce() v0.0.2a x 3,349,559 ops/sec ±1.22% (92 runs sampled)
    ✓  fast.reduce() v0.0.1 x 3,242,995 ops/sec ±1.44% (92 runs sampled)
    ✓  fast.reduce() v0.0.0 x 2,807,336 ops/sec ±0.95% (94 runs sampled)
    ✓  underscore.reduce() x 1,166,136 ops/sec ±1.23% (91 runs sampled)
    ✓  lodash.reduce() x 2,105,040 ops/sec ±1.29% (91 runs sampled)

    Result: fast.js is 157.03% faster than Array::reduce().

  Native .reduce() vs fast.reduce() (1000 items)
    ✓  Array::reduce() x 18,009 ops/sec ±1.41% (94 runs sampled)
    ✓  fast.reduce() x 44,939 ops/sec ±1.42% (92 runs sampled)
    ✓  fast.reduce() v0.0.2c x 27,075 ops/sec ±1.22% (89 runs sampled)
    ✓  fast.reduce() v0.0.2b x 45,958 ops/sec ±1.23% (94 runs sampled)
    ✓  fast.reduce() v0.0.2a x 45,496 ops/sec ±1.78% (89 runs sampled)
    ✓  fast.reduce() v0.0.1 x 44,654 ops/sec ±1.60% (96 runs sampled)
    ✓  fast.reduce() v0.0.0 x 31,335 ops/sec ±3.20% (89 runs sampled)
    ✓  underscore.reduce() x 18,163 ops/sec ±1.02% (93 runs sampled)
    ✓  lodash.reduce() x 34,446 ops/sec ±1.76% (93 runs sampled)

    Result: fast.js is 149.53% faster than Array::reduce().

  Native .reduceRight() vs fast.reduceRight() (3 items)
    ✓  Array::reduceRight() x 2,858,156 ops/sec ±1.03% (94 runs sampled)
    ✓  fast.reduceRight() x 8,059,775 ops/sec ±1.34% (90 runs sampled)
    ✓  underscore.reduceRight() x 2,201,049 ops/sec ±1.14% (91 runs sampled)
    ✓  lodash.reduceRight() x 2,537,010 ops/sec ±1.08% (92 runs sampled)

    Result: fast.js is 181.99% faster than Array::reduceRight().

  Native .reduceRight() vs fast.reduceRight() (10 items)
    ✓  Array::reduceRight() x 1,288,729 ops/sec ±1.37% (94 runs sampled)
    ✓  fast.reduceRight() x 3,474,325 ops/sec ±1.33% (96 runs sampled)
    ✓  underscore.reduceRight() x 1,132,806 ops/sec ±1.45% (93 runs sampled)
    ✓  lodash.reduceRight() x 1,559,888 ops/sec ±1.38% (93 runs sampled)

    Result: fast.js is 169.59% faster than Array::reduceRight().

  Native .reduceRight() vs fast.reduceRight() (1000 items)
    ✓  Array::reduceRight() x 17,790 ops/sec ±1.09% (96 runs sampled)
    ✓  fast.reduceRight() x 45,672 ops/sec ±1.33% (90 runs sampled)
    ✓  underscore.reduceRight() x 17,838 ops/sec ±1.06% (93 runs sampled)
    ✓  lodash.reduceRight() x 30,986 ops/sec ±1.19% (91 runs sampled)

    Result: fast.js is 156.73% faster than Array::reduceRight().

  Native .forEach() vs fast.forEach() (3 items)
    ✓  Array::forEach() x 1,189,685 ops/sec ±1.57% (90 runs sampled)
    ✓  fast.forEach() x 1,694,065 ops/sec ±1.19% (92 runs sampled)
    ✓  fast.forEach() v0.0.2a x 1,679,212 ops/sec ±1.45% (90 runs sampled)
    ✓  fast.forEach() v0.0.1 x 1,685,784 ops/sec ±1.45% (89 runs sampled)
    ✓  fast.forEach() v0.0.0 x 1,644,848 ops/sec ±1.20% (95 runs sampled)
    ✓  underscore.forEach() x 1,106,432 ops/sec ±4.36% (87 runs sampled)
    ✓  lodash.forEach() x 1,720,851 ops/sec ±1.15% (91 runs sampled)

    Result: fast.js is 42.40% faster than Array::forEach().

  Native .forEach() vs fast.forEach() (10 items)
    ✓  Array::forEach() x 493,856 ops/sec ±1.07% (92 runs sampled)
    ✓  fast.forEach() x 682,928 ops/sec ±1.21% (97 runs sampled)
    ✓  fast.forEach() v0.0.2a x 668,328 ops/sec ±1.72% (92 runs sampled)
    ✓  fast.forEach() v0.0.1 x 676,832 ops/sec ±1.28% (91 runs sampled)
    ✓  fast.forEach() v0.0.0 x 625,505 ops/sec ±1.28% (92 runs sampled)
    ✓  underscore.forEach() x 488,658 ops/sec ±1.47% (92 runs sampled)
    ✓  lodash.forEach() x 666,097 ops/sec ±1.21% (90 runs sampled)

    Result: fast.js is 38.29% faster than Array::forEach().

  Native .forEach() vs fast.forEach() (1000 items)
    ✓  Array::forEach() x 5,897 ops/sec ±1.26% (90 runs sampled)
    ✓  fast.forEach() x 7,907 ops/sec ±1.38% (86 runs sampled)
    ✓  fast.forEach() v0.0.2a x 8,018 ops/sec ±0.93% (92 runs sampled)
    ✓  fast.forEach() v0.0.1 x 8,027 ops/sec ±1.28% (91 runs sampled)
    ✓  fast.forEach() v0.0.0 x 7,163 ops/sec ±2.15% (92 runs sampled)
    ✓  underscore.forEach() x 5,982 ops/sec ±1.19% (90 runs sampled)
    ✓  lodash.forEach() x 7,802 ops/sec ±1.32% (91 runs sampled)

    Result: fast.js is 34.07% faster than Array::forEach().

  Native .some() vs fast.some() (3 items)
    ✓  Array::some() x 3,174,981 ops/sec ±1.07% (94 runs sampled)
    ✓  fast.some() x 9,612,633 ops/sec ±0.88% (96 runs sampled)
    ✓  underscore.some() x 2,863,160 ops/sec ±1.09% (94 runs sampled)
    ✓  lodash.some() x 6,745,454 ops/sec ±1.54% (89 runs sampled)

    Result: fast.js is 202.76% faster than Array::some().

  Native .some() vs fast.some() (10 items)
    ✓  Array::some() x 1,373,806 ops/sec ±5.57% (84 runs sampled)
    ✓  fast.some() x 4,754,689 ops/sec ±1.03% (95 runs sampled)
    ✓  underscore.some() x 1,430,441 ops/sec ±1.26% (88 runs sampled)
    ✓  lodash.some() x 3,792,287 ops/sec ±1.14% (94 runs sampled)

    Result: fast.js is 246.10% faster than Array::some().

  Native .some() vs fast.some() (1000 items)
    ✓  Array::some() x 21,843 ops/sec ±1.77% (94 runs sampled)
    ✓  fast.some() x 71,994 ops/sec ±1.10% (97 runs sampled)
    ✓  underscore.some() x 21,905 ops/sec ±1.52% (93 runs sampled)
    ✓  lodash.some() x 69,019 ops/sec ±1.72% (92 runs sampled)

    Result: fast.js is 229.60% faster than Array::some().

  Native .concat() vs fast.concat() (3 items)
    ✓  Array::concat() x 815,665 ops/sec ±1.06% (92 runs sampled)
    ✓  fast.concat() x 4,351,586 ops/sec ±1.77% (92 runs sampled)

    Result: fast.js is 433.50% faster than Array::concat().

  Native .concat() vs fast.concat() (10 items)
    ✓  Array::concat() x 663,085 ops/sec ±1.92% (91 runs sampled)
    ✓  fast.concat() x 3,195,359 ops/sec ±1.38% (91 runs sampled)

    Result: fast.js is 381.89% faster than Array::concat().

  Native .concat() vs fast.concat() (1000 items)
    ✓  Array::concat() x 790,957 ops/sec ±0.60% (96 runs sampled)
    ✓  fast.concat() x 120,855 ops/sec ±1.15% (94 runs sampled)

    Result: fast.js is 84.72% slower than Array::concat().

  Native .concat() vs fast.concat() (1000 items, using apply)
    ✓  Array::concat() x 30,089 ops/sec ±1.48% (92 runs sampled)
    ✓  fast.concat() x 53,097 ops/sec ±1.23% (91 runs sampled)

    Result: fast.js is 76.47% faster than Array::concat().

  Finished in 1117 seconds
```

## Credits

Fast.js is written by [codemix](http://codemix.com/), a small team of expert software developers who specialise in writing very fast web applications. This particular library is heavily inspired by the excellent work done by [Petka Antonov](https://github.com/petkaantonov), especially the superb [bluebird Promise library](https://github.com/petkaantonov/bluebird/).

The very minor modifications to fast.js that comprise fast-poser were written by [Nick Bottomley](https://github.com/nickb1080), and were intended specifically to add support for [poser](https://github.com/bevacqua/poser). This fork is used in [super-collection](https://github.com/nickb1080/super-collection).

## License

MIT, see [LICENSE.md](./LICENSE.md).
