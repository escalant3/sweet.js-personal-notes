# sweet.js notes

#### Disclaimer

These are my personal sweet.js notes. They are far from being complete (or
  correct). You've been warned!

They include contents from the next original sites:

- [sweet.js site](http://sweetjs.org/)
- [James Long tutorial](http://jlongster.com/Writing-Your-First-Sweet.js-Macro)
- [Mozilla wiki examples](https://github.com/mozilla/sweet.js/wiki/Example-macros)


## Introduction

From [sweetjs.org](http://sweetjs.org)

> Sweet.js brings the hygienic macros of languages like Scheme and Rust to JavaScript. Macros allow you to sweeten the syntax of JavaScript and craft the language you’ve always wanted.


### What is a macro?

From Wikipedia [1](http://en.wikipedia.org/wiki/Macro_(computer_science),[2](http://en.wikipedia.org/wiki/Hygienic_macro)

> A macro (short for "macroinstruction", from Greek μακρο- 'long') in computer science is a rule or pattern that specifies how a certain input sequence (often a sequence of characters) should be mapped to a replacement output sequence (also often a sequence of characters) according to a defined procedure.

#### Syntactic macros

> Macro systems—such as the C preprocessor described earlier—that work at the level of lexical tokens cannot preserve the lexical structure reliably. Syntactic macro systems work instead at the level of abstract syntax trees, and preserve the lexical structure of the original program. The most widely used implementations of syntactic macro systems are found in Lisp-like languages such as Common Lisp, Clojure, Scheme, ISLISP and Racket.

#### Hygienic macros

> Hygienic macros are macros whose expansion is guaranteed not to cause the accidental capture of identifiers.

### Why macros in JS?

It seems people like to customize JS: https://github.com/jashkenas/coffee-script/wiki/List-of-languages-that-compile-to-JS

Macros allow to:
- Customize JS
- Avoid full-commitment
- Specific problem optimization


## Usage

### Live code editor

[Online editor](http://sweetjs.org/browser/editor.html)

### Installation

```
npm install -g sweet.js
```

## Dive in

### Pattern-based macros

(syntax pattern) ---(template)---> (new syntax)

Macros are defined using the `macro` keyword:

```javascript
macro <name> {
 rule { <pattern> } => { <template> }
}
```

**Macro names** can contain any valid JavaScript indentifier or keyword. They
can also include (or be exclusively) one or more symbols with the exception of
`[ ] ( ) { }` that are considered delimiters.

**Patterns** are matched literally with the exception of some special tokens
that will be shown later in this document. The first of them is the `$` symbol
as shown in the following identity macro example:

```javascript
macro id {
  rule { ($x) } => { $x }
}

// Output
id (23)       // 23
id ([1,2,3])  // [1,2,3]
id ('Hello')  // 'hello'
```

The **template** is a set of statements that uses the matched patterns to
generate valid JavaScript code.

Macros can be composed by a set of rules. The first one that matches the input
is the only one getting triggered.

Being able to use JavaScript keywords as macro names means that we can extend
the default behaviour to, for instance, add ES6 capabilities.

#### Parse classes

Letting `$x` match anything would not allow to perform a proper control over
the pattern. That's why sweet.js provides parse classes that match only certain
inputs using the format
`$name:class`. The classes available are:

1. `expr`: matches an expresion
2. `ident`: matches an identifier
3. `lit`: matches a literal

```javascript
macro pclass {
  rule { $x:lit } => { $x + 'literal'  }
  rule { $x:ident } => { $x + 'identifier' }
  rule { $x:expr } => { $x + 'expression' }
}

pclass 8;
pclass "hello world";
pclass foo;
pclass (2 + 5 * 3);
pclass 2 + 5 * 3;
pclass baz();
pclass true;


// Generates
// 8 + 'literal';
// 'hello world' + 'literal';
// foo + 'identifier';
// 2 + 5 * 3 + 'expression';
// 2 + 'literal' + 5 * 3; (Rule order matters!)
// baz + 'identifier'();
//true + 'literal';
```

The sweet.js parser is not greedy, so the first result matches. The `expr`
parse class can match `lit` and `ident` if its precedence is higher. Try
deleting some of the previous rules and checking the result.

#### Repetition

Another powerful tool is the ability to match repeated patterns using `...`. It
is also possible to specify the separator for the repeated patterns. The
default separator is any blank space character.

```javascript
macro print {
  rule {
    ($x ...)
  } => {
    console.log($x (,) ...)
  }
}

print (1 2 3 4)
// console.log(1, 2, 3, 4);


macro sum {
  rule {
    ($x ...)
  } => {
    console.log($x (+) ...)
  }
}

sum (1 2 3 4)
//console.log(1 + 2 + 3 + 4);
```

#### Grouping

```javascript
macro m {
  rule { ( $($id:ident = $val:expr) (,) ...) } => {
    $(var $id = $val;) ...
  }
}
m (x = 10, y = 2+10)

// var x = 10;
// var y = 2 + 10;
```

#### Recursion
The result of applying a macro is also parsed to look for more macros:

```javascript
macro foo {
  rule { $x } => {
    'foo' + bar $x
  }
}

macro bar {
  rule { $x } => {
    'bar' + $x
  }
}

foo 5

// 'foo' + 'bar' + 5;
```

While this is a powerful feature, it comes with a drawback. If we want to extend
a built-in keyword like `var` to support ES6 de-structuring and try something
like:

```javascript
// Don't try this in the online editor
macro var {
  rule {
    $ident:ident = $expr:expr
  } => {
    var $ident = $expr
  }
}

var a = 5
```

The code above in the online editor will freeze Chrome. Using Firefox we'll
get a more useful message:

> InternalError: too much recursion

The parser is converting `var` into the contents of the *<template>* section.
This contents contain `var` as well, that is again parsed and so on.

Sweet.js provides the `let` keyword to avoid this. With let, all the built-in
keywords in the *<template>* section will be treated as the original one and
not as the macro.

```javascript
// Recursion safe
let var = macro {
  rule {
    $ident:ident = $expr:expr
  } => {
    var $ident = $expr
  }
}

var a = 5
```

With the let keyword we are able to have de-structuring assignment easily:

```javascript
// https://gist.github.com/aaditmshah/7065183

let var = macro {
    rule { $name:ident = $value:expr } => {
        var $name = $value
    }

    rule { {$name:ident (,) ...} = $value:expr } => {
        var object = $value
        $(, $name = object.$name) ...
    }

    rule { [$name:ident (,) ...] = $value:expr } => {
        var array = $value, index = 0
        $(, $name = array[index++]) ...
    }
}

var o = 0;

var [a, b, c] = [1, 2, 3];

var {x, y, z} = {x: 1, y: 2, z: 3};
```


### Lookback

```javascript
macro is {
  rule infix { $left:expr | $right:expr } => {
    ($left === $right)
  }
}

macro unless {
  rule infix { $value:expr | $guard:expr } => {
    if (!($guard)) {
      $value
    }
  }
}
var supermanWins = 'no';
var rock = 'kryptonite';

supermanWins = 'yes' unless rock is 'kryptonite'

console.log('Does he win?', supermanWins);

//var supermanWins = 'no';
//var rock = 'kryptonite';
//if (!(rock === 'kryptonite')) {
//    supermanWins = 'yes';
//}
//console.log('Does he win?', supermanWins);
```

```javascript
macro in {
  rule infix { $value:expr | $array:expr } => {
    ($array.indexOf($value) !== -1)
  }
}

2 in [1,2,3,4]
//([ 1, 2, 3, 4 ].indexOf(2) !== -1)
8 in [1,2,3,4]
//([ 1, 2, 3, 4 ].indexOf(8) !== -1)
'e' in 'Hello'
//('Hello'.indexOf('e') !== -1)
'a' in 'Hello'
//('Hello'.indexOf('a') !== -1)
if 'o' in 'Hello' { console.log('Found!') }
//if ('Hello'.indexOf('o') !== -1) {
//    console.log('Found!');
//}

```

### Procedural macros

> Sweet.js also provides a more powerful way to define macros: case macros. Case macros allow you to manipulate syntax using the full power of JavaScript.

```javascript
macro <name> {
  case { <pattern> } => { <template> }
}
```

They also match the macro name:

```javascript
macro m {
  case { $name $x } => { ... }
}
m 42
```

Although you can ignore it with a wildcard:

```javascript
macro m {
  case { _ $x } => { ... }
}
```

> The other difference from rule macros is that the body of a macro contains a mixture of templates and normal JavaScript that can create and manipulate syntax. Templates are now created with the #{...} form.

```javascript
macro id {
  case {_ $x } => {
    return #{ $x }
  }
}
```

#### Syntax objects

**letstx** macro that binds syntax objects to pattern variables

```javascript
macro m {
  case {_ $x } => {
    var y = makeValue(42, #{$x});
    letstx $y = [y], $z = [makeValue(2, #{$x})];
    return #{$x + $y - $z}
  }
}
m 1
// --> expands to
1 + 42 - 2
```

- **makeValue(val, stx)** – *val* can be a boolean, number, string, or
null/undefined
- **makeRegex(pattern, flags, stx)** – *pattern* is the string representation
of the regex pattern and *flags* is the string representation of the regex flags
- **makeIdent(val, stx)** – *val* is a string representing an identifier
- **makePunc(val, stx)** – *val* is a string representing a punctuation
(e.g. =, ,, >, etc.)
- **makeDelim(val, inner, stx)** – *val* represents which delimiter to make
and can be either "()", "[]", or "{}" and inner is an array of syntax objects
for all of the tokens inside the delimiter.
