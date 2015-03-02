<a href="http://promisesaplus.com/">
    <img src="https://promises-aplus.github.io/promises-spec/assets/logo-small.png"
         align="right" valign="top" alt="Promises/A+ logo" />
</a>
# Jexl [![Build Status](https://travis-ci.org/TechnologyAdvice/Jexl.svg?branch=master)](https://travis-ci.org/TechnologyAdvice/Jexl)
Javascript Expression Language: Powerful context-based expression parser and evaluator

## Quick start
Use it with promises or callbacks:

```javascript
var context = {
    name: {first: 'Sterling', last: 'Archer'},
    assoc: [
        {first: 'Lana', last: 'Kane'},
        {first: 'Cyril', last: 'Figgis'},
        {first: 'Pam', last: 'Poovey'}
    ],
    age: 36
};

// Filter an array
jexl.eval('assoc[.first == "Lana"].last', context).then(function(res) {
    console.log(res); // Output: Kane
});

// Do math
jexl.eval('age * (3 - 1)', context, function(err, res) {
    console.log(res); // Output: 72
});

// Concatenate
jexl.eval('name.first + " " + name["la" + "st"]', context).then(function(res) {
    console.log(res); // Output: Sterling Archer
});

// Compound
jexl.eval('assoc[.last == "Figgis"].first == "Cyril" && assoc[.last == "Poovey"].first == "Pam"', context)
    .then(function(res) {
        console.log(res); // Output: true
    });

// Use array indexes
jexl.eval('assoc[1]', context, function(err, res) {
    console.log(res.first + ' ' + res.last); // Output: Cyril Figgis
});

// Transform
jexl.addTransform('upper', function(val) {
    return val.toUpperCase();
});
jexl.eval('"duchess"|upper + " " + name.last|upper', context).then(function(err, res) {
    console.log(res); // Output: DUCHESS ARCHER
});

// Transform asynchronously, with arguments
jexl.addTransform('lookup', function(val, args) {
    return dbSelectByLastName(val, args.stat); // Returns a promise
});
jexl.eval('name.last|lookup{stat: "weight"}', context, function(err, res) {
    if (err) console.log('Database Error', err.stack);
    else console.log(res); // Output: 184
});
```

## Installation
Jexl requires an environment that supports the
[Promise/A+](https://promisesaplus.com/) specification as standardized in ES6.
Node.js version 0.12.0 and up is great right out of the box (no --harmony flag
necessary), as well as the latest versions of many browsers. To support older
browsers, just include a Promise library such as
[Bluebird](https://github.com/petkaantonov/bluebird).

For Node.js, type this in your project folder:

    npm install jexl --save

For the frontend, drop `dist/jexl.min.js` into your project and include it on
your page with:

    <script src="path/to/jexl.min.js"></script>

Access Jexl the same way, backend or front:

    var jexl = require('Jexl');

## All the details
### Binary Operators

| Operation        |      Symbol      |
|------------------|:----------------:|
| Add, Concat      |         +        |
| Subtract         |         -        |
| Multiply         |         *        |
| Divide           |         /        |
| Divide and floor |        //        |
| Modulus          |         %        |
| Power of         |         ^        |
| Logical AND      |        &&        |
| Logical OR       |   &#124;&#124;   |

### Unary Operators

| Operation | Symbol |
|-----------|:------:|
| Negate    |    !   |

### Comparisons

| Comparison            | Symbol |
|-----------------------|:------:|
| Equal                 |   ==   |
| Not equal             |   !=   |
| Greater than          |    >   |
| Greater than or equal |   >=   |
| Less than             |    <   |
| Less than or equal    |   <=   |

### Native Types

| Type     |            Examples            |
|----------|:------------------------------:|
| Booleans |         `true`, `false`        |
| Strings  | "Hello \"user\"", 'Hey there!' |
| Numerics |      6, -7.2, 5, -3.14159      |

### Groups

Parentheses work just how you'd expect them to:

| Expression                          | Result |
|-------------------------------------|:-------|
| (83 + 1) / 2                        | 42     |
| 1 < 3 && (4 > 2 &#124;&#124; 2 > 4) | true   |

### Identifiers

Access variables in the context object by just typing their name. Objects can
be traversed with dot notation, or by using brackets to traverse to a dynamic
property name.

Example context:

```javascript
{
    name: {
        first: "Malory",
        last: "Archer"
    },
    exes: [
        "Nikolai Jakov",
        "Len Trexler",
        "Burt Reynolds"
    ],
    lastEx: 1
}
```

| Expression        | Result        |
|-------------------|---------------|
| name.first        | Malory        |
| name['la' + 'st'] | Archer        |
| exes[2]           | Burt Reynolds |
| exes[lastEx - 1]  | Len Trexler   |

### Collections

Collections, or arrays of objects, can be filtered by including a filter
expression in brackets. Properties of each collection can be referenced by
prefixing them with a leading dot. The result will be an array of the objects
for which the filter expression resulted in a truthy value.

Example context:
```javascript
{
    employees: [
        {first: 'Sterling', last: 'Archer', age: 36},
        {first: 'Malory', last: 'Archer', age: 75},
        {first: 'Lana', last: 'Kane', age: 33},
        {first: 'Cyril', last: 'Figgis', age: 45},
        {first: 'Cheryl', last: 'Tunt', age: 28}
    ],
    retireAge: 62
}
```

| Expression                                    | Result                                                                                |
|-----------------------------------------------|---------------------------------------------------------------------------------------|
| employees[.first == 'Sterling']               | [{first: 'Sterling', last: 'Archer', age: 36}]                                        |
| employees[.last == 'Tu' + 'nt'].first         | Cheryl                                                                                |
| employees[.age >= 30 && .age < 40]            | [{first: 'Sterling', last: 'Archer', age: 36},{first: 'Lana', last: 'Kane', age: 33}] |
| employees[.age >= 30 && .age < 40][.age < 35] | [{first: 'Lana', last: 'Kane', age: 33}]                                              |
| employees[.age >= retireAge].first            | Malory                                                                                |

### Transforms

The power of Jexl is in transforming data, synchronously or asynchronously.
Transform functions take two arguments: The value to be transformed, and
a map of arguments. They must return either the transformed value, or a Promise
that resolves with the transformed value. Add them with
`jexl.addTransform(name, function)`.

```javascript
jexl.addTransform('split', function(val, args) {
    return val.split(args.char);
});
jexl.addTransform('lower', function(val) {
    return val.toLowerCase();
});
```

| Expression                                       | Result                |
|--------------------------------------------------|-----------------------|
| "Pam Poovey"&#124;lower&#124;split{char: ' '}[1] | poovey                |
| "password==guest"&#124;split{char: '=' + '='}    | ['password', 'guest'] |

#### Advanced Transforms
Using Transforms, Jexl can support additional string formats like embedded
JSON, YAML, XML, and more.  The following, with the help of the
[xml2json](https://github.com/buglabs/node-xml2json) module, allows XML to be
traversed just as easily as plain javascript objects:

```javascript
var xml2json = require('xml2json');

jexl.addTransform('xml', function(val) {
    return xml2json.toJson(val, {object: true});
});

var context = {
    xmlDoc:
        "<Employees>" +
            "<Employee>" +
                "<FirstName>Cheryl</FirstName>" +
                "<LastName>Tunt</LastName>" +
            "</Employee>" +
            "<Employee>" +
                "<FirstName>Cyril</FirstName>" +
                "<LastName>Figgis</LastName>" +
            "</Employee>" +
        "</Employees>"
};

var expr = 'xmlDoc|xml.Employees.Employee[.LastName == "Figgis"].FirstName';

jexl.eval(expr, context).then(function(res) {
    console.log(res); // Output: Cyril
});
```

### Context

Variable contexts are straightforward Javascript objects that can be accessed
in the expression, but they have a hidden feature: they can include a Promise
object, and when that property is used, Jexl will wait for the Promise to
resolve and use that value!

### API

#### jexl.Jexl
A reference to the Jexl constructor. To maintain separate instances of Jexl
with each maintaining its own set of transforms, simply re-instantiate with
`new jexl.Jexl()`.

#### jexl.addTransform(_{string} name_, _{function} transform_)
Adds a transform function to this Jexl instance.  See the **Transforms**
section above for information on the structure of a transform function.

#### jexl.addTransforms(_{{}} map_)
Adds multiple transforms from a supplied map of transform name to transform
function.

#### jexl.getTransform(_{string} name_)
**Returns `{function|undefined}`.** Gets a previously set transform function,
or `undefined` if no function of that name exists.

#### jexl.eval(_{string} expression_, _{{}} [context]_, _{function} [callback]_)
**Returns `{Promise<*>}`.** Evaluates an expression.  The context map and
callback function are optional. If a callback is specified, it will be called
with the standard signature of `{Error}` first argument, and the expression's
result in the second argument.  Note that if a callback function is supplied,
the returned Promise will already have a `.catch()` attached to it.

## License
Jexl is licensed under the MIT license. Please see `LICENSE.txt` for full
details.

## Credits
Jexl was designed and created at
[TechnologyAdvice](http://technologyadvice.com).
