---
layout: post
title: TypeScript types cheat sheet
tags: TypeScript
---

This post is a collection of the available TypeScript types, examples of their usual use, and JavaScript outputs.

# Boolean

Nothing special, just `true` and `false`:

**TypeScript**:
```
let isDone: boolean = true;
```
**JavaScript**:
```
let isDone = true;
```

# Number

4 ways to declare a number:

**TypeScript**:
```
let decimal: number = 6;
let hex: number = 0xf00d;
let binary: number = 0b0101;
let octal: number = 0o744;
```
**JavaScript**:
```
let decimal = 6;
let hex = 0xf00d;
let binary = 0b0101;
let octal = 0o744;
```

> Note: All numbers in TypeScript are floating point values

# String

Same to `boolean`, pretty much as-is:

**TypeScript**:
```
let color: string = "blue";
let template = `Sky is ${color}`;
```
**JavaScript**:
```
let color = "blue";
let template = `Sky is ${color}`;
```

# Array

2 ways to declare an array:

**TypeScript**:
```
let list: number[] = [1, 2, 3];
let array: Array<number> = [1, 2, 3];
```
**JavaScript**:
```
let list = [1, 2, 3];
let array = [1, 2, 3];
```

> Note: Both ways produce the same JavaScript output

# Tuple

Tuple is a type that allows you to express an array with a fixed number of elements whose types are known:

**TypeScript**:
```
let x: [string, number];
x = ["hello", 10];
let str = x[0];
let num = x[1];
```

**JavaScript**:
```
let x;
x = ["hello", 10];
let str = x[0];
let num = x[1];
```

# Enum

## Numeric

Numeric enum allows you to use and specify `numeric` values:

**TypeScript**:
```
enum Color { Red = 1, Green = 2 }
let c: Color = Color.Green;
```
**JavaScript**:
```
var Color;
(function (Color) {
    Color[Color["Red"] = 1] = "Red";
    Color[Color["Green"] = 2] = "Green";
})(Color || (Color = {}));
let c = Color.Green;
```

Look at this elegant approach: 

```
Color[Color["Red"] = 1] = "Red";
```

Since the `Color` is the object, this line is responsible for producing the following output:

```
{
  1: "Red"
  2: "Green"
  Red: 1
  Green: 2
}
```

Eventually, it allows us to access the enum both ways: either `Color.Red` or `Color[1]`.

### Get name

Two ways to get the enum's key:

**TypeScript**:
```
let cs: string = Color[0] || Color[Color.Green] // Green
```

**JavaScript**:
```
let cs = Color[0] || Color[Color.Green]; // Green
```

> Note: Works with numeric enums only

### Duplicates

I don't know why but you can use a single value for multiple enum keys:

**TypeScript**:
```
enum Vehicle { Car = 1, Plane = 1 }
let vs: Vehicle = Vehicle.Car;       // 1
let vss: Vehicle = Vehicle.Plane;    // 1
```

**JavaScript**:
```
var Vehicle;
(function (Vehicle) {
    Vehicle[Vehicle["Car"] = 1] = "Car";
    Vehicle[Vehicle["Plane"] = 1] = "Plane";
})(Vehicle || (Vehicle = {}));
let vs = Vehicle.Car; // 1
let vss = Vehicle.Plane; // 1
```

In this case you'll end up with the following output:

```
{
  1: "Plane"
  Car: 1
  Plane: 1
}
```

> Note: You can get the same value by multiple keys but not vice versa (who is the last and is the father)

## String

TBD

## Heterogeneous

TBD

# Any

TBD

# Null & undefined

TBD

# Never

TBD

Reference:

1. [TypeScript playground](http://www.typescriptlang.org/play/?ssl=1&ssc=1&pln=68&pc=53#code/PTAECEHtIGwUwIYDsBQ8AuoCWBnAIpEnAFygBG08yoAvKOgE4CucA3CiiKAHJMC2ZOAzRxMAEzgBjLHwQxSSfoIa1QANnYZQACzgAPBUqGqADHoBmJk2M2jyWJAgYBPQwON0TZEwEZftzEhJdDk3ZVNIAHYAFmj2TjAAZUYHAHMRTElYSAZSHBSkVNUAIjIYFmKA+jg+AAcYBHQ4VQADRIBrZ2wcUAASAG8smByAXxb4rgBBBgYEZwzQGFx0MKEAbQBdVTWfABpQACZ9gGYNqqdZ11Bpy4AeRXcGAD5tvcOTs44uABUmergFgZQGt8gw0vsHspPnptsVdDBhsV9n5PlpQao9GsTKi7A8MTtPglQABRPEAChwMn+EP4Qiwkn2qVxCD4cAAlCg4HiAMLZFT9UAAJTgYlUbwA4gw4FzVAdQCMFpJSLzhio6CqcgA6SXSpBVSQ4PIFIrqvlYrYAHwtoA1DDWtu1Uq5Wy4Oq5XzApP4oApVPg+1BaQ5XO9iX0oAFAFk5M06MVI5MADLEpGgABiNRjJTTxITyeK8oWhtAYZhdFLmuj8FAXCrALRxdLqlLa3jSZTlutLYrdZdnpmOQ9JPJOBZzQAbnIWMG8QA1ODaenVgXcpxi-YABQaRDFha04+L88XkmrdCPS7gmtXDFYoDv95rYB8CwPh4XF9U55Pl63yDYD64Z8iS9PgfV0JoGEgJkiEgJgcBnUMQnQOCIxLEIGCaUU6AAckSb5JkFb5iTwbDN0g1IpRwHo6DeAgdwVIlkHmLQkEgdBEiYKVSCY1Q4hQVj2M42NQGw2RnEEUAEFAQNCmw9gBI4qVVHMOQcDYIcbjmUBIHMSSkGY3E2MUuBE2Wbj9M2V59mw8wnVI+hmDgDYh0UBF9iYJAJHMBwRQWJhSA8ryfKw0BArgbyiBsBYkEMBFVFcmAJjAIhxyEFBzA84IsEIUAhAYMk+BwVIjTBQo2QUOBUv5FAH3QbRIIAd1AIgmuJAd8sK1I2XYBUMqQLKcpUrAYDJcrmsq4x+hq+8pWQhgkFymYyWwtMEGGkVsO6lBesy9BsoWhxzETaBalGiqqojaa7waxdqzJRhp1QhUGK4dBnFqZoECooQ9sIItIFZWdzK6OM6twbpJOk41Kn+wHEy5VI6tWNUfVuGTUieHAAbgWc2U1eBCjqqosbhhG6oOZHVApbHZ0knp0bxgnEe0VggA);
2. [Basic types](https://www.typescriptlang.org/docs/handbook/basic-types.html);
3. [Enums in TypeScript](https://www.tutorialsteacher.com/typescript/typescript-enum);