# Attack On TypeScript

## Introduction

### What is TypeScript and what is TypeScript not?

TypeScript is among other things, a tool to document your code as you write it.  
The technique used is called "typing", and is a method that assign a layer of
semantics to your code by annotating it with types.

```typescript
const shout = (input: string): string => input + '!';
```

If the shout function is provided an argument that is anything other than a string,
TypeScript will give a compile time error.

```typescript
shout(5); // Argument of type 'number' is not assignable to parameter of type 'string'.
```

Even, taking a step back, it helps the developer of the `shout` function assert
that the function always returns a string. If the implementation had differed slightly,
TypeScript would complain about this.

```typescript
// Function lacks ending return statement and return type does not include 'undefined'.
const shout = (input: string, happyMood?: boolean): string => {
  if (happyMood) {
    return input + '!';
  }
};
```

Note: TypeScript is not designed to catch errors in logic, only in data types.

### What types are there?

We're already used to the `string`-type, for example: `boolean` and `number`.
And also an infinite amount of composite types, for example:

- Arrays: `Array<string>`
- Tuples: `[string, number]`
- Objects: `{name: string}`

There are, however, some more complicated types that are used to encode
semantics that are on the "edge" of the type system. For example:

`never`, `any`, `unknown`.

Did you know that `never` is the type that is returned by a function that always throws?

Try it:

```typescript
const alwaysThrows = () => {
  throw new Error('haha');
};
```

`unknown` is a really good type to use when we want to warn people that they might need to do
some investigating to figure out what type the value is. We'll get to that in a minute.

`any` is a type that you want to use when you want to bail out of the type system - basically
telling the compiler to give up and not report any errors. This has other unexpected consequences
and those consequences are basically what this is about.

Something to note about `any`, any value can be assigned to an `any`. But even more devlish: `any` can be assigned to any value.

So this is 100% correct TypeScript:

```typescript
const myValue: any = 'hello';
const myNumber: number = myValue;
```

Remember this, because this is incredibly useful when finding bugs.
Also remember: `any` isn't categorically evil in 100% of cases, it's just that it CAN sometimes come and bite you.

## IO Boundaries

When using TypeScript, you're not only using your own types, you're using the already typed types of the standard library.
Sometimes, the already existing types of the standard library are wrong - and can lead you down a buggy path which
propagates bugs throughout your whole code. These kinds of errors can sometimes be pretty tough to track down,
especially if you're trusting TypeScript to do its job properly.

There are a few examples, and the hardest one has to tackle has to do with typing the response of an network call (websocket, fetch, etc).
We will get to that, and solve that elegantly.

But I want to start with an example that's easier to demonstrate locally, can you find the error here?

```typescript
const shout = (input: string): string => input + '!';

type Person = { person: { name: string } };

const parse = (unparsed: string): Person => JSON.parse(unparsed);

const parseAndShout = (unparsed: string): string => {
  const parsed = parse(unparsed);
  const shoutedName = shout(parsed.person.name);
  return shoutedName;
};
```

If we want to be picky (and yes, here we do want to be picky), this code shouldn't be accepted by TypeScript.
Because this code has an error, the nature of the error is Data Types, and TypeScript is supposed to catch
datatype errors. The runtime error will occur in this line: `parsed.person.name`, however the compile time error
should have occured much earlier, actually in the `parse` function.

If we unroll the `parse` function, make it a bit more explicit, we can see what's going on:

```typescript
const parse = (unparsed: string): { person: { name: string } } => {
  const returnValue = JSON.parse(unparsed); // This is an `any`
  return returnValue; // An any returned here satisfies the return type `{ person: {name: string}}`
};
```

The comments say it all. How do we fix this?

Note, before we fix this, we'll just quickly show that `fetch` has the same problem by this code:

```typescript
const getPerson = (): Promise<{ person: { name: string } }> => {
  return fetch('https://localhost:5000/person').then(x => x.json());
};
```

It's not obvious, but this code won't work most of the time because there's nothing in runtime
checking that `https://localhost:5000/person` actually returns a valid response, with JSON, in
the shape that the `getPerson`-function promises.

The problem here is that `x.json` returns a `Promise<any>`, which is a valid type substitution for
`{person: {name: string}}` since `any` substitutes any type.

### Fixing the standard libraries types, or getting in even more trouble.

Fixing the return type of `body.json()`

```typescript
declare global {
  interface Body {
    json(): Promise<unknown>;
  }
}
```

Fixing the return type of JSON.parse:

```typescript
declare global {
  interface JSON {
    parse(text: string): unknown;
  }
}
```

### Checking unknowns

To verify the shape of objects in runtime, we can use something called type predicates.

```typescript
const myFunction = (input: unknown) => {
  if (typeof input === 'string') {
    // Here `input` is a `string`.
    // The type is checked in runtime AND compile time.
  }
  // Here `input` is still `unknown`
};
```

We can break out type-checking into functions:

```typescript
const isString = (input: unknown): input is string => typeof input === 'string';
const myFunction = (input: unknown) => {
  if (isString(input)) {
    // Here `input` is a `string`.
    // The type is checked in runtime AND compile time.
  }
  // Here `input` is still `unknown`
};
```

But we can also use something called codecs:

```typescript
import * as io from 'io-ts';

const myFunction = (input: unknown) => {
  if (io.string.is(input)) {
    // Here `input` is a `string`.
    // The type is checked in runtime AND compile time.
  }
  // Here `input` is still `unknown`
};
```

The benefits of codecs is that they can encode really complicated types,
even composite shapes, such as: objects, arrays, tuples.

```typescript
import * as io from 'io-ts';

const PersonCodec = io.type({
  name: io.string,
  age: io.number,
});

const Person = typeof PersonCodec._A;

const myFunction = (input: unknown) => {
  if (PersonCodec.is(input)) {
    // Here `input` is a `Person`.
    // The type is checked in runtime AND compile time.
  }
  // Here `input` is still `unknown`
};
```

### Finishing up, fixing the original code.

```typescript
import * as io from 'io-ts

const shout = (input: string): string => input + '!';

const PersonCodec = io.type({
  person: io.type({
    name: io.string
  })
})

type Person = typeof PersonCodec._A

const parse = (unparsed: string): Person | null => {
  const parsed = JSON.parse(unparsed);
  if (PersonCodec.is(parsed))
    return parsed
  return null
};

const parseAndShout = (unparsed: string): string => {
  const parsed = parse(unparsed);
  // We'll get an error here, asking us to check for null
  const shoutedName = shout(parsed.person.name);
  return shoutedName;
};
```

