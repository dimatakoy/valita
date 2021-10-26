# @badrap/valita [![tests](https://github.com/badrap/valita/workflows/tests/badge.svg)](https://github.com/badrap/valita/actions?query=workflow%3Atests) [![npm](https://img.shields.io/npm/v/@badrap/valita.svg)](https://www.npmjs.com/package/@badrap/valita)

A TypeScript library for validating & parsing structured objects. The API is _heavily_ influenced by [Zod's](https://github.com/colinhacks/zod/tree/v3) excellent API, while the implementation side aims for the impressive performance of [simple-runtypes](https://github.com/hoeck/simple-runtypes).

We also pay special attention for providing descriptive validation error messages:

```ts
const vehicle = v.union(
  v.object({ type: v.literal("plane"), airline: v.string() }),
  v.object({ type: v.literal("train") }),
  v.object({ type: v.literal("automobile"), make: v.string() })
);
vehicle.parse({ type: "bike" });
// ValitaError: invalid_literal at .type (expected "plane", "train" or "automobile")
```

## Installation

```bash
npm i @badrap/valita
```

## Docs aren't my forté

A motivating example in lack of any better documentation:

```ts
import * as v from "@badrap/valita";

const Pet = v.object({
  type: v.union(v.literal("dog"), v.literal("cat")),
  name: v.string(),
});

const Person = v.object({
  name: v.string(),
  age: v.number(),
  pets: v.array(Pet).optional(),
});
```

Now `Person.parse(value)` returns `value` if it matches the Person schema - or throws an error otherwise.

```ts
const grizzlor = Person.parse({
  name: "Grizzlor",
  age: 101,
  pets: [
    { type: "cat", name: "Mittens" },
    { type: "cat", name: "Parsley" },
    { type: "cat", name: "Lulu" },
    { type: "cat", name: "Thomas Percival Meowther III" },
  ],
});
```

The real magic here comes from TypeScript's type inference. The inferred type for `grizzlor` is:

```ts
const grizzlor: {
  name: string;
  age: number;
  pets?: { type: "dog" | "cat"; name: string }[] | undefined;
};
```

You can use `Infer<T>` to get your mitts on the inferred type in your code:

```ts
type PersonType = v.Infer<typeof Person>;
```

## API Reference

This section contains an overview of all validation methods.

### Primitive Types

Let's start with the basics! Like every validation library we support all primitive types like `strings`, `numbers`, `booleans`, `null`, `undefined` and more.

```ts
import * as v from "@badrap/valita";

const developer = v.object({
  name: v.string(),
  age: v.number(),
  usesValita: v.boolean(),
  projects: v.array(v.string()),
  null: v.null(),
  undefined: v.undefined(),
  bigNumber: v.bigint(),
});
```

On top of that we support additional methods to coerce specific values to a certain TypeScript type:

```ts
import * as v from "@badrap/valita";

const something = v.object({
  shouldNotHappen: v.never(),
  noClueWhatThisIs: v.unknown(),
});
```

### Optional Object Properties

One common occurrence when working with APIs is that some fields in an object are optional and therefore can be missing completely. Valita can skip validating those via the `.optional()` method:

```ts
import * as v from "@badrap/valita";

const person = v.object({
  name: v.string(),
  // Not everyone filled in their favorite song
  song: v.string().optional(),
});

// Passes
person.parse({ name: "Jane Doe", song: "Never gonna give you up" });
// Passes
person.parse({ name: "Jane Doe" });
```

### Record Type

Whereas `.object()` can be used to validate objects, it has the limitation that all property keys have to be known upfront. It doesn't support an arbitrary number of properties, which is where Record types come to the rescue.

```ts
import * as v from "@badrap/valita";

// Every property key must be known upfront
const obj = v.object({
  foo: v.number(),
  bar: v.number(),
});

// ...with records
const obj2 = v.record(v.number());
```

With the record type you're essentially specifying that there are (theoretically) an infinite number of property keys which are all of type `string`. The argument you pass to `.record()` is the type of the property values. In the example above all property values are numbers.

### Tuple Types

Despite JavaScript not having tuple values (yet?), many APIs leverage plain arrays as an escape hatch. For example: If we encode a range between two numbers we'd likely choose `type Range = [number, number]` as the data type. From JavaScript's point of view it's just an array but TypeScript knows about the value of each position and that the array **must** have two entries.

We can express the same with valita:

```ts
import * as v from "@badrap/valita";

const range = v.tuple([v.number(), v.number()]);

// Passes
range.parse([1, 2]);
range.parse([200, 2]);

// Fails
range.parse([1]);
range.parse([1, 2, 3]);
range.parse([1, "2"]);
```

### Union Types

A union type is a value which can have several different representations. Let's imagine we have a value of type `Shape` that can be either a circle, a square or a rectangle.

```ts
import * as v from "@badrap/valita";

const rectangle = v.object({ type: "rectangle" });
const square = v.object({ type: "square" });
const circle = v.object({ type: "circle" });

// Validates either as "rectangle", "square" or "circle"
const shape = v.union(rectangle, square, circle);
```

Note that although in this example all representations have a shared property `type`, it's not necessary at all. Each representation can be completely different:

```ts
import * as v from "@badrap/valita";

const primitive = v.union(v.number(), v.string(), v.boolean());
```

### Literal Types

Sometimes knowing if a value is of a certain type is not enough. We can use the `.literal()` method to check for actual values, like checking if a string is either `"red"`, `"green"` or `"blue"` and not just any string.

```js
import * as v from "@badrap/valita";

const rgb = v.union(v.literal("red"), v.literal("green"), v.literal("blue"));
```

We can also use this to check for concrete numbers, bigint literals or boolean values too:

```ts
import * as v from "@badrap/valita";

const banana2YearsOld = v.object({
  kind: v.literal("banana"),
  age: v.literal(2),
  isFresh: v.literal(true),
});
```

For more complex values you can use the `.assert()`-method. Check out the [Custom validation functions](#user-content-custom-validation-functions) to learn more about it.

### Custom validation functions

The `.assert()`-method can be used for custom validation logic like verifying that a number is inside a certain range for example.

```js
import * as v from "@badrap/valita";

const schema = v
  .number()
  .assert((v) => v >= 0 && v <= 255, "Must be in between 0 or 255");
```

### Recursive Types

One strong suit of valita is its ability to parse types that references itself, like `type T = string | T[]`. We can express such a shape with valita using the `.lazy()` method.

```ts
import * as v from "@badrap/valita";

type T = string | T[];
const myType: v.Type<T> = v.lazy(() => v.union(v.string(), v.array(myType)));
```

_Note that TypeScript needs an explicit type cast as it cannot interfere return types of recursive functions. That's why the variable is typed as `v.Type<T>`._

## License

This library is licensed under the MIT license. See [LICENSE](./LICENSE).
