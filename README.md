# RegExp Named Capture Groups

Stage 4
Champions: Daniel Ehrenberg (Igalia) & Mathias Bynens (Google)

## Introduction

Numbered capture groups allow one to refer to certain portions of a string that a regular expression matches. Each capture group is assigned a unique number and can be referenced using that number, but this can make a regular expression hard to grasp and refactor.

For example, given `/(\d{4})-(\d{2})-(\d{2})/` that matches a date, one cannot be sure which group corresponds to the month and which one is the day without examining the surrounding code. Also, if one wants to swap the order of the month and the day, the group references should also be updated.

Named capture groups provide a nice solution for these issues.

## High Level API

A capture group can be given a name using the `(?<name>...)` syntax, for any identifier `name`. The regular expression for a date then can be written as `/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/u`. Each name should be unique and follow the grammar for ECMAScript IdentifierName.

Named groups can be accessed from properties of a `groups` property of the regular expression result. Numbered references to the groups are also created, just as for non-named groups. For example:

```js
let re = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/u;
let result = re.exec('2015-01-02');
// result.groups.year === '2015';
// result.groups.month === '01';
// result.groups.day === '02';

// result[0] === '2015-01-02';
// result[1] === '2015';
// result[2] === '01';
// result[3] === '02';
```

The interface interacts nicely with destructuring, as in the following example:

```js
let {groups: {one, two}} = /^(?<one>.*):(?<two>.*)$/u.exec('foo:bar');
console.log(`one: ${one}, two: ${two}`);  // prints one: foo, two: bar
```

### Backreferences

A named group can be accessed within a regular expression via the `\k<name>` construct. For example,

```js
let duplicate = /^(?<half>.*).\k<half>$/u;
duplicate.test('a*b'); // false
duplicate.test('a*a'); // true
```

Named references can also be used simultaneously with numbered references.

```js
let triplicate = /^(?<part>.*).\k<part>.\1$/u;
triplicate.test('a*a*a'); // true
triplicate.test('a*a*b'); // false
```

### Replacement targets

Named groups can be referenced from the replacement value passed to `String.prototype.replace` too. If the value is a string, named groups can be accessed using the `$<name>` syntax. For example:

 ```js
let re = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/u;
let result = '2015-01-02'.replace(re, '$<day>/$<month>/$<year>');
// result === '02/01/2015'
```

Note that an ordinary string literal, not a template literal, is passed into `replace`, as that method will resolve the values of `day` etc rather than having them as local variables. An alternative would be to use `${day}` syntax (while remaining not a template string); this proposal uses `$<day>` to draw a parallel to the definition of the group and a distinction from template literals.

If the second argument to `String.prototype.replace` is a function, then the named groups can be accessed via a new parameter called `groups`. The new signature would be `function (matched, capture1, ..., captureN, position, S, groups)`. Named captures would still participate in numbering, as usual. For example:

 ```js
let re = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/u;
let result = '2015-01-02'.replace(re, (...args) => {
  let {day, month, year} = args[args.length - 1];
  return `${day}/${month}/${year}`;
});
// result === '02/01/2015'
```

## Details

### Overlapping group names

RegExp result objects have some non-numerical properties already, which named capture groups may overlap with, namely `length`, `index` and `input`. In this proposal, to avoid ambiguity and edge cases around overlapping names, named group properties are placed on a separate `groups` object which is a property of the match object. This solution will permit additional properties to be placed on the result of `exec` in future ECMAScript versions without creating any web compatibility hazards.

The groups object is only created for RegExps with named groups. It does not include numbered group properties, only the named ones. Properties are created on the `groups` object for all groups which are mentioned in the RegExp; if they are not encountered in the match, the value is `undefined`.

### Backwards compatibility of new syntax

The syntax for creating a new named group, `/(?<name>)/`, is currently a syntax error in ECMAScript RegExps, so it can be added to all RegExps without ambiguity. However, the named backreference syntax, `/\k<foo>/`, is currently permitted in non-Unicode RegExps and matches the literal string `"k<foo>"`. In Unicode RegExps, such escapes are banned.

In this proposal, `\k<foo>` in non-Unicode RegExps will continue to match the literal string `"k<foo>"` *unless* the RegExp contains a named group, in which case it will match that group or be a syntax error, depending on whether or not the RegExp has a named group named `foo`. This does not affect existing code, since no currently valid RegExp can have a named group. It would be a refactoring hazard, although only for code which contained `\k` in a RegExp.

## Precedent in other programming languages

This proposal is analogous to what many other programming languages have done for named capture groups. It seems to be what the consensus syntax is moving towards, though the Python syntax is an interesting and compelling outlier which would address the non-Unicode backreference issue.

### Perl [ref](http://perldoc.perl.org/perlre.html#Regular-Expressions)

Perl uses the same syntax as this proposal for named capture groups `/(?<name>)/` and backreferences `/\k<name>/`.

### Python [ref](https://docs.python.org/2/library/re.html#regular-expression-syntax)

Named captures have the syntax `"(?P<name>)"` and have backrereferences with `(?P=name)`.

### Java [ref](https://blogs.oracle.com/xuemingshen/entry/named_capturing_group_in_jdk7)

JDK7+ supports named capture groups with syntax like Perl and this proposal.

### .NET [ref](https://msdn.microsoft.com/en-us/library/bs2twtah(v=vs.110).aspx#Anchor_1)

C# and VB.NET support named capture groups with the syntax `"(?<name>)"` as well as `"(?'name')"` and backreferences with `"\k<name>"`.

### PHP

According to a [Stack Overflow post](http://stackoverflow.com/questions/6971287/named-capture-in-php-using-regex) and a [comment on php.net docs](http://php.net/manual/en/function.preg-match.php#89418), PHP has long supported named groups with the syntax `"(?P<foo>)"`, which is available as a property in the resulting match object.

### Ruby [ref](https://ruby-doc.org/core-2.2.0/Regexp.html#class-Regexp-label-Capturing)

Ruby's syntax is identical to .NET, with named capture groups with the syntax `"(?<name>)"` as well as `"(?'name')"` and backreferences with `"\k<name>"`.

## Draft specification

[Draft spec](https://tc39.github.io/proposal-regexp-named-groups/)

## Implementations

* [V8](https://bugs.chromium.org/p/v8/issues/detail?id=5437), shipping in Chrome 64
* [XS](https://github.com/Moddable-OpenSource/moddable/blob/public/xs/sources/xsre.c), in [January 17, 2018 update](http://blog.moddable.tech/blog/january-17-2017-big-update-to-moddable-sdk/)
* [Transpiler (Babel plugin)](https://github.com/DmitrySoshnikov/babel-plugin-transform-modern-regexp#named-capturing-groups)
* [Safari](https://developer.apple.com/safari/technology-preview/release-notes/) beginning in Safari Technology Preview 40
