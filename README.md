# Status

Champions: None

authors: Scotty Jamison

Stage: -1 (Not currently at a stage)

# Motivation

According to the "State of JS survey", static typing is the most wanted featured by JavaScript developers (as seen in their [2020](https://2020.stateofjs.com/en-US/opinions/missing_from_js/) and [2021](https://2021.stateofjs.com/en-US/opinions/#currently_missing_from_js_wins) results). Recently, Microsoft pushed for a [type annotations](https://github.com/tc39/proposal-type-annotations) proposal to help address some of these needs, however, their proposal comes at the cost of having multiple different type-systems operating in the same syntax space, instead of a single, unified type system for JavaScript.

This proposal will take things in the opposite direction and explore something completely different. Their README argues that "TC39 has a tradition of programming language design which favors local, sound checks", and then goes on to state that TypeScript's success is based on not following these principles, which is why a native type-system for JavaScript shouldn't be micro-managed by TC39. However, TypeScript also had severe limitations in what it could accomplish. If we marry the type-checker and the runtime together, we might be able to create a sound type system, one that doesn't leave room for surprises. A type system where, if you say something needs to be a number, it's going to be a number, and that's that (a.k.a. this is a strongly-typed type system, not a weakly-typed one).

The goal of this proposal is to explore the space of implementing an official strong type system in JavaScript, to see what it would take, and to see the pros and cons of using such an approach. In order to acomplish this proposal, we're going to need to create a new standard document that describes an official JavaScript type-checker - a tool that runs against your JavaScript code before your code is run in a production environment.

**NOTE:** In the future, when I have time, I might add finer details to this repository that provides a more complete description of what this proposal will provide, along with prior art, a comparison with different languages, etc, but for now, I just want to address some of the most important points regarding this topic, and provide a general idea of how this kind of type-checker can be created.

# High-Level Objectives

* This type system will be strongly typed. A single, standard weak type system for JavaScript that uses a types-as-comments approach is certainly a valid approach as well that ought to be explored, but this isn't what this proposal is choosing to cover at this moment.
* Low runtime cost. This will likely be the primary concern about having a strong type system, so one major focus of this proposal will be to provide tools to lower the runtime cost assosiated with strongly typing a dynamic language.
* Avoid any sort of `"use types"` directive. There's a lot of pushback associated with adding new directives, so it would be nice if we can find a solution that didn't require one.
* Backwards compatibility. This also means it should be possible to add types to the web-APIs without introducing a breaking change.
* Unlike Microsoft's proposal, we're not trying to create any parity with existing type systems. We can certainly be inspired by what they've done and refer to them as prior art, but there's no incitive to preserve any kind of backwards compatibility with TypeScript, or any other existing type-system. What's being introduced is too fundamentally different for this sort of thing to matter.

One more point: This type system isn't attempting to bless every type of obscure development pattern with type safety, instead, the goal is to provide type safety for a reasonable subset of the language, that's broad enough to let people work comfortably (especially in newer code), without being so broad as to incur a lot of complexity. We do need to keep in mind the backwards compatibility objective, and make sure the type system is broad enough to allow developers to translate their existing APIs into something type-safe without introducing breaking changes. We should also make sure we provide appropriate escape hatches for developers to use if, for whatever reason, the provided type system does not suit their needs (e.g. an `any` type).

# Description

## The fundamentals

Lets start with the basics, declaring a type. We'll just use `::` syntax for this, this is a strawman, any syntax could be used, we just need to pick something. The `::` syntax is being chosen mostly because the popular choice, `:`, already has so many meanings in the language, and causes issues when you want to declaring types while destructuring.

As an added nice-to-have feature, we'll also permit you to put a type declaration on the line before the variable/function declaration.

```javascript
:: number
let myNumb = 2

let myNumb2 :: number = 2

:: (number, number) => number
function add(x, y) {
  return x + y
}

function add2(x :: number, y :: number) :: number {
  return x + y
}
```

We good so far? The first two chunks of that code sample are just showing the two ways to declare a variable of type `number`, and the last two chunks are showing how to add type declarations to a function.

These type declarations are also assertions. You can not reassign the variable `myNumb` to a string value, nor can you call `add()` with string arguments, or with too few arguments.

Mutable declarations without type information will automatically have an `any` type associated with them. These pairs of declarations are all the same:

```javascript
// These two lines are the same
let x = 2
let x :: any = 2

// These two lines are the same
function fn(x) { return x + 1 }
function fn(x :: any) :: any { return x + 1 }
```

The `any` type behaves a bit differently from TypeScript's `any` type. Type-checking is effectively turned off for the `any` type, however, runtime checks will still happen against it if it's assigned to a value with a declared type. For example:

```javascript
let hasAnyType = 'x'
let hasStringType :: string = 'x'

const add1 = (x :: number) :: number => x + 1

add1(hasAnyType) // Runtime error
add1(hasStringType) // Type-check-time error and runtime error (remember that all users aren't using the official type-checker)
```

Note that if a lot of the code has types, it should be possible for engines to optimize out many of these assertions, because they can see that the function will only be called with valid types. In fact, they might be able to run faster than normal because of this foreknowledge of what kinds of types they can expect the function to receive.

`const` declarations will automatically attempt to infer their type from the RHS expression. It will tend towards a more conservative type, and will be quick to fallback to an `any` type if it's unable to figure it out.

```javascript
// These two lines are the same
const x = 2
const x :: number = 2

// These two lines are the same
const y = x * x
const y :: number = x * x
```

You can ask a mutable binding to automatically infer it's type by using the `infer` keyword. We could perhaps discuss a shorter way to write this, due to how often it's likely going to be used, but for now this should do.

```javascript
// These two lines are the same
let x :: infer = 2
let x :: number = 2

// These two lines are the same
function fn(y :: infer = x * x) { ... }
function fn(y :: number = x * x) { ... }
```

Note that this means the following are errors:

```javascript
let x :: infer = 2
x = "Now I'm a string!" // Type-check-time and runtime error

let y :: any = 'string'
x = y // Runtime error
```

The purpose of the `any` type is to facilitate in upgrading existing JavaScript code to typed code, and to provide an escape-hatch if you need to implement some logic outside of the type system (either because the type system can't handle your current use case very well, or because you need to work around some performance issues).

One thing to note is that you can't just take existing JavaScript code, run it through the type checker, and expect it to always come out without any errors. For example, the following is valid JavaScript (and doesn't throw an error, even at runtime), however, the typechecker will create an error about how you can't subtract strings.

```javascript
> 'x' - 'y'
NaN
```

For this reason, it may be useful to provide different strictness options to the type-checker, similar to how TypeScript is configurable. One path we could take would be to create the spec for the type-checker using the most strict rules (e.g. rules like no-implicit-any are "enabled" by default), and then, maybe, provide a handful of the most important configuration options in the spec (or we don't provide any configuration options in the spec), then permit each implementation to provide other, additional configuration options to further configure how strict the type-checker will be, on a more granular level. It's most important to provide a spec that describes how the strictest version of the type-checker behaves, it's less important to describe the intermediate strictness levels that mainly exist to help you upgrade your code.

## Importing modules

We don't have the help of directives to help us toggle types on and off, so instead, the type system is made to act a bit like a disease - once you start using types in a particular module, their effect will cascade as far as reasonably possible, however, it will never implicitly cascade through a module boundary. This allows you to upgrade to types, one module at a time.

This shows the cascading effect in action:

```javascript
// We provide a single type declaration
const fn = () :: number => 2

const x = fn()
const getY = () => fn()

const z = x + getY()
console.log(z.length) // type-check-time error! (length is not define on a number)
```

Through the means of type-inference, all of this code became type-safe from the single type declaration. It won't always happen this way, especially if you use `let` bindings a lot (which by default get inferred as an `any` type).

Now, lets split this example up across multiple modules.

```javascript
// example.js
const fn = () :: number => 2

export const x = fn()
export const getY = () => fn()

// main.js
import { x, getY } from './example.js'
const z = x + getY()
console.log(z.length) // No errors, not even at runtime (it'll simply print undefined, since numbers don't define the length property)
```

By default, when you import a value, it's imported with an `any` type, in order to ensure type-safety doesn't leak across module boundaries. If you want to import the values with their associated types, you'll have to do this:

```javascript
import inferred { x, getY } from './example.js'
// Or this works too, if you need more granular control
import { inferred x, getY } from './example.js'
```

If the `import inferred` syntax were used in the above example, we would have gotten a type-check-time error on the `console.log(z.length)` line.

Note that even if you're importing a function as an `any` type, this doesn't mean you can pass whatever you want into it. The following code still results in a runtime error, because the assertions defined with `addOne()` still get triggered.

```javascript
// example.js
const addOne = (x :: number) => x + 1

// main.js
import { addOne } from './example.js' // addOne has the type `any` here.
addOne('hi there!') // Runtime error!
```

It is certainly possible to transition to types in a more granular approach than one module at a time, if this is needed. You can just use an `as` assertion to change the type of a value within a particular scope to `any`, to keep the type-checker off your back within that scope.

## Object Types

In TypeScript, you might be used to writing type declarations for objects like this:

```typescript
const getX = (point: { x: number, y: number }): number => point.x
getX({ x: 2, y: 3 })
```

The type declaration of the `point` parameter shown above simply describes the shape of an object, and any value that conforms to this shape automatically has that type, whether or not you explicitly gave it that type.

While this works great for TypeScript, it can be problematic if we want a fast type system with strong guarantees. If we allowed a function to be declared in this sort of way in JavaScript, the runtime would have to go through each property of a passed-in object to make sure it conforms to the type, before executing the function's body.

To help improve performance, this proposal will provide a tagging system, which works like this:

```javascript
interface Point {
  x :: number
  y :: number
}

:: (Point!) => number
const getX = point => point.x

getX({ x: 2, y: 3 })
```

The getX function's type declaration states that it'll accept any object that's tagged with the `Point` interface. The `!` before `Point` indicates that if the object currently doesn't have a `Point` tag, the runtime should attempt to auto-tag it as follows: If the passed-in value conforms to the interface, it will receive a `Point` tag in a hidden field, otherwise, an error would be thrown. The reason the `!` symbol was chosen was to try and draw attention to the fact that this can be a potential performance bottleneck if you're dealing with especially large interfaces.

If you don't place a `!` in front, than an error will be thrown if the function ever receives a value that is untagged, it won't fall back to auto-applying a tag. Note that the user can still apply the tag on their end before passing in a value into a function:

```javascript
interface Point {
  x :: number
  y :: number
}

:: (Point) => number // Not using a `!` this time
const getX = point => point.x

getX({ x: 2, y: 3 }) // Type-check-time and runtime error
const myPoint :: Point! = { x: 2, y: 3 }
getX(myPoint) // ok
getX({ x: 2, y: 3 } as Point!) // Also ok. I'm sure the type-checker will have some sort of `as` operator, like TypeScript has.
```

A value can loose its tag if it gets modified in an unsupported way. Such modifications would be type-checker errors, but a user isn't required to use a type-checker. The example below is littered with type-checker errors, but it will all run just fine, until you reach the last line.

```javascript
interface Point {
  x :: number
  y :: number
}

const myPoint :: Point! = { x: 2, y: 3 }
myPoint.z = 4 // ok
myPoint.x = 4 // ok
myPoint.x = 'hi there!' // Causes myPoint to loose its tag
delete myPoint.y // Also causes myPoint to loose its tag

// This will make `myPoint` conform to the `Point` interface again, however, it won't automatically regain its tag.
myPoint.x = 2
myPoint.y = 3

// This would throw a runtime error, because myPoint is still untagged. (note there's no `!` to auto-apply the missing tag)
const anotherPoint :: Point = myPoint
```

To help ensure you have good performance, try to reduce how often you're using a `!`. In general, you shouldn't have to worry as much about internal functions, as the only ones calling the internal functions are under your control, and are being validated with a type-checking tool. Because of this, it should be safe to omit the `!` from the type signatures of internal functions. For public-facing functions, you want to make sure you only check what needs to be checked, for example, in our `getX()` function we really only care about the `x` property on the passed-in object. It really doesn't matter to us if the `y` property is present or not - if we want to assert that it's there, we can (which is what we're currently doing), but perhaps it's ok to drop this particular _runtime_ assertion (if our end-users are using a type-checker, we still want to force them to pass in a valid Point object).

```javascript
interface Point {
  x :: number
  y :: number
}

:: (request Point) => number
const getX = point => {
  if (x === null || typeof x !== 'object') throw new TypeError('Bad Input!')
  if (!('x' in point)) throw new TypeError('Bad Input!')
  return point.x
}
```

We've seen the `!` type modifier, in this code snippet we run into another type modifier, the `request` modifier. This `request` modifier is used to indicate that this particular type is only enforced during the type-check phase, and will not be enforced at runtime, at all. In the above example, this means anyone trying to call getX who's using a type-checker will be required to pass in an instance of `Point`, while anyone who's calling it without using a type-checker will simply receive a runtime error if the `x` property was missing from their argument (because we're explicitly throwing an error if this is the case).

It's called `request` because you're simply, politely requesting that the end-user provides a value of this particular type, nothing's being done at runtime to enforce this type. For this reason, within the function definition the `point` parameter is treated as having an `unknown` type, because theoretically it could be receiving any value. Those `if (...) throw ...` assertions are important, not just to make sure the value passed in was valid, but to also make the `return point.x` line pass the type-checker - just like in TypeScript, assertions can be made to teach the type-checker more information about a particular type (i.e. after the first `if` the type-checker knows that the type is an object, and after the second `if` the type checker knows the object has an `x` property, so by the time we get to `return`, it's legal to do `point.x`)

Now that we understand the basic theory, we can introduce another simpler way to achieve the same effect. It requires splitting up the interface into smaller pieces:

```javascript
interface HasXProp {
  x :: number
}

// Interface inheritance works as expected
interface Point extends HasXProp {
  y :: number
}

// The `&` combinator lets you apply multiple types at once.
:: (HasXProp! & request Point) => number
const getX = point => point.x
```

This example does a very similar thing to the previous one. We're defining a function, `getX()`, which requests that a `Point` type be passed in, however, the only thing it enforces at runtime is the `HasXProp!` type, i.e. all you have to do is pass in an object that conforms to `HasXProp` at runtime, and the `!` modifier will cause it to be auto-tagged. Within the function definition, we know that the `point` parameter conforms to `HasXProp! & request Point`, which is the same as `HasXProp! & unknown`, which is the same as `HasXProp`, thus we're free to access the `x` property of `point`.

Some other miscellaneous items to be aware of:

A single object can be tagged with multiple interface tags at once. This can happen due to interface inheritance (if an object is tagged with `Point`, it'll also be tagged with `HasXProp`), but it can also be done simply by the user using `AnotherTag!` a lot on the same object.

Also note that tags are applied to _values_, not _variable names_. This means its a side-effect. As an example:

```javascript
// This is illegal
const myPoint1 :: unknown = { x: 2, y: 3 }
const myPoint2 :: Point = myPoint1 // Error - I'm not using `!`, so a tag won't be auto-applied.

// But if I add a line in between the two statements, the last line doesn't throw anymore.
const myPoint1 :: unknown = { x: 2, y: 3 }
const whatever :: Point! = myPoint1 // I added this line. The `!` assertion will cause
                                    // the whatever/myPoint1 object to be tagged with this interface
const myPoint2 :: Point = myPoint1 // Not an error, because myPoint1 is now tagged.
```

The line `myPoint1 as Point!` could have been inserted as well to get the same effect.

The side-effect nature of tagging is another reason to avoid overusing the `!` auto-tag modifier. Use it where it's needed, but when it's not, don't use it. Newer APIs that aren't designing for backwards compatibility may choose to never auto-tag an existing object with `!`, instead, they can let the end-user manually and explicitly tag their own objects (and if wanted, the end-user can choose to never apply tags to existing objects). For example:

```javascript
// yourApi.js
export interface Point { ... }

:: (Point) => number
export const getX = point => point.x

// example.js
import inferred { Point, getX } from './yourApi.js'

getX({ x: 2, y: 3 }) // Type-check-phase and runtime error - the value is not tagged.
getX({ x: 2, y: 3 } as Point!) // ok
const myPoint = { x: 2, y: 3 }
getX({ ...myPoint } as Point!) // Example of applying a tag to a duplicate object, to avoid mutating the original object.
```

In the end, I don't feel like the fact that tags "mutate" existing objects is a major concern - it may cause some unexpected issues, but the biggest issues  would happen during the type-check phase, making them early errors.

A couple other notes on interfaces:
* You're able to create an interface that describes a nested object structure.
* You're also able to compose a nested interface using another interface. e.g. `interface MyInterface { x: AnotherInterface }`. If you auto-tag an object with `!` against `MyInterface` from this example, its nested `x` property will also be auto-tagged with `AnotherInterface`.

## Classes

Each class definition has an interface bundled with it that describes the shape of its instances. A class full of public and private properties, each with a type declaration would have an associated interface that contains those private and public properties with their corresponding types. Any missing type definitions will be treated as the `any` type.

For example:

```javascript
class MyClass {
  x :: number | undefined // `|` combinators are allowed, just like in TypeScript,
                          // and let you say the property is one of these types.
  y = 0 // has the type `unknown`
  #z :: string | undefined
}

// obj has the type `MyClass`, and is actually tagged with the MyClass interface.
const obj = new MyClass()
const obj2 :: MyClass = obj // Works

obj2.a = 3 // ok
obj2.x = 'Hi there!' // This mutation will invalidate the tagged interface (and cause a type-check-phase error)
const obj3 :: MyClass = obj2 // A runtime error would occur here.

// If wanted, the object could be repaired so it matches the MyClass interface again,
// and then a `MyClass!` assertion can be used to restore the tag.
// Note that because the interface contains private members, only instances of MyClass can conform to this interface.
```

A class's method automatically expects to be called with the class's instance as a receiver (the `this param`), and it won't try to auto-tag it.

```javascript
interface HasXProp { x :: number }

class Point {
  x :: number
  y :: number

  // This...
  doThing(x :: number) { ... }

  // Is the same as this...
  // (This is the syntax to annotate the type of the "this" parameter)
  doThing(this :: Point, x :: number) { ... }

  // If we want finer control over how the `this` parameter is typed, we can do so, for example, like this:
  doThing(this :: Point!, x :: number) { ... }

  // Or like this:
  doThing(this :: HasXProp! & request Point, x :: number) { ... }
}
```

Concepts such as Java's `final`, `@Override`, etc would also be interesting topics to explore, but won't be explored at this time.

## Misc features

The current rendition of this proposal is mostly trying to focus on features that were important to the overall premise of doing strong, native type-checking in JavaScript. For this reason, many type-related features were only briefly discussed, like union types (`x | y`) or interface inheritance. Certainly, as this proposal matures, we'll need to discuss other type-safe features we'd like to include or not include such as generics, tuple types, class types, mapped object types `{ [key: string]: number }`, abstract classes, etc).

It is good to point out that we ought to support a way to add types to something that currently doesn't have types, e.g. something akin to TypeScript's declaration syntax and declaration files. For example:

```javascript
// thirdPartyLib.d.js
export interface Point { x :: number, y :: number }

export declare const mutatePoint = (point :: request Point) => Point!

// example.js
import * as Point from './thirdPartyLib.js' declaredAt './thirdPartyLib.d.js'
```

You'll notice that the type signature for `mutatePoint` is `(point :: request Point) => Point!` - this type signature will be layered on top of the imported function. The parameters are simple requests that you'll be required to comply with since you're using a type-checker, while the return value is an auto-tagging assertion, forcing the API to comply with the return value you expected from it. When the arguments are passed into a third-party function, they'll be put into a special state that forbids them from beind modified in any way that would cause them to loose interface tags that their caller expects them to have. (e.g. if the `mutatePoint` did `point.x = 'hi there!'`, a runtime error would be thrown, instead of the value silently loosing its tag).

Also note that, if we do go forwards with a proposal like this, we can choose to split it up into many smaller proposals. A base proposal to provide the main features, and side proposals for features such as union types. The whole thing can move through the stages together, but can be discussed independently. Hopefully this can help alleviate some of the concerns about the size of this sort of proposal.

## globalThis mutation protection

We're going to bake one more useful features into interfaces - it's the ability to require a property to contain an exact value, like this:

```javascript
const specificObj = { x: 2 }

interface MyInterface {
  myObj: is specificObj
}

{ myObj: specificObj } as MyInterface! // Works
{ myObj: { x: 2 } } as MyInterface! // Fails, the identity of the myObj property didn't match the expected object identity.
```

With a built-in type system like this, and the ability to create interfaces that require exact values by their identities, it becomes fairly easy to protect yourself from globalThis mutations (e.g. someone replacing `Math.max` with some other bad function). Here's how it works.

You will need to declare what type `globalThis` is. This is important so that the type-checker can know if this is, for example, running in a browser environment, or a node environment, etc. EcmaScript will provide an interface that specifies what should be found on any untampered globalThis object, e.g. if you want to run in any compatible, untampered EcmaScript environment, you can do `declare globalThis as EcmaScriptGlobalThis` where `EcmaScriptGlobalThis` is the globally provided interface for EcmaScript's API. While it wouldn't be included in the spec, Node, and web APIs can choose to globally provide their own interfaces, and someone building a type-checker can choose to support them. Someone expecting to run in a custom environment where globaalThis has been modified can import a declaration file supplied by that environment (or that they write it themselves), and apply that to the `globalThis` value.

The default EcmaScriptGlobalThis interface provided is full of these identity checks (e.g. it was declared using a bunch of `is someValue`). This means, if anyone tampers with any value on globalThis, globalThis will loose its label, and any of your runtime code doing `globaThis.whatever` will fail, because globalThis isn't what you said it was supposed to be. The only way to restore the label is to restore `globalThis` back to its original form. You can still add new stuff to globalThis without making it loose its label, and if you're first-running code you can even replace the globally provided interface with a new one (important if you're a polyfill).

## Other advantages to spec-ing a static-code-analysis tool.

Doing type-checking isn't the only thing a standard code-analysis tool can be used for. Here's a handful of other ideas we could peruse in the future if we go this route:
* We could include basic, built-in linting. For example, we can state that "using semicolons" is the way to go in JavaScript, and automatically require them if you use the standard code-analysis tool.
* Along a similar vein, it becomes easier to phase out older, broken features. For example, the `typeof` operator is full of footguns (null is an object? A function's type is `function` but every other object's type is `object`?). It would be hard to add a newer, better `typeof` function to detect primitive types, and then phase out the old `typeof` operator by dis-allowing it when you use the official code analysis tool. Of course, the `typeof` operator will always be available at runtime, it just wouldn't be available for use with the code-analysis tooling (or a warning would be generated if you do use it). Removing support for certain syntax features at "build"-time can help the language loose some weight, and become less foot-gunny.
* Older, deprecated APIs can also be restricted by the code-analysis tooling. For example, the tooling can forbid you to pass a string into `setTimeout` to have it be auto-evaluated. (Yes, `setTimeout` isn't technically part of the EcmaScript spec, but it still would be nice if browsers could do this sort of soft deprecation using the official code-analysis tooling as well. Perhaps the web-APIs would make an "extension" spec describing how the EcmaScript code analyzer should be extended to properly support browser-based APIs).
* We can make new "keywords" like making `async`/`await` become actual keywords at "build"-time. e.g. you're not allowed to use `await` as a variable name if you want to make the code analyzer happy.
* Or, we don't do any of these things. These are simply meant to showcase some other potential directions we could take if we had an official code analyzer.
