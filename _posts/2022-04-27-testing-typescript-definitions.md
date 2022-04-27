---
title:  "Testing TypeScript Defintions"
date:   2022-04-27 19:00:00 +0200
excerpt: "TypeScript type definitions can be quite complex. They need to be tested to avoid mistakes during implementation and refactoring."
header:
  og_image: /assets/images/2022-04-27/testing-typescript-definitions.png
---

TypeScript type definitions can be quite complex. They need to be tested to avoid mistakes during implementation and refactoring.

## Introducing the Example

In my last post, I showed how to use TypeScript to check translations and their usage at compile time. I would like to briefly recall this example.

I defined a `type DeepKeysOf`, which returns all possible nested keys from one object.

```typescript
type DeepKeysOf<T, Key extends keyof T = keyof T> = Key extends string
  ? T[Key] extends string ? Key : `${Key}.${DeepKeysOf<T[Key]>}`
  : never;
```

If this type definition is used for the translate method now, the compiler can already point out faulty translations.

```typescript
const translation = {
  dialog: {
    title: 'Bestätigung',
    description: 'Möchten Sie fortfahren?'
  }
}

type TranslationKey = DeepKeysOf<typeof translation>; // 'dialog.title' | 'dialog.description'

const translate = (translationKey: TranslationKey): string => {
  // omit the actual implementation
}

translate('dialog.title'); // compiles fine
translate('dialog.header'); // compiler error: Argument of type '"dialog.header"' is not assignable to parameter of type 'TranslationKey'.
```

The full example with more detailed explanations can be found in my [previous post]({% post_url 2022-03-30-using-typescript-to-validate-translations-at-compile-time %}) or in this [TypeScript Playground](https://www.typescriptlang.org/play?#code/PTAEFUGcEsDsHNQBUCeAHAppAxgJ2mgC6iED2oAbgIYA20AJlYRiblbJDU9KR6E6GykAtmmg0WhaMIwAoEKAAWhQmkgAuEIUUiqkAHRpcGaJE4Zc+mcABGNUvGAAmAAxOnwFwGZgXl8ABXGAQAWkJ0LDwCQjDSEOo6RmYwtg4uKV5IEKYQoVFxDDDpOVkhDmJCVM5uXlAAXlAAb1lQJQwaewB1UlwaenVQAHIACVp7UE72wgBCQYAaFtB6aFoHAebW1qlCCQHBgCEsQgATqXgAhHnF1vpI-CIeWD2AWQA37GUMWFAAZWgWABmPUIAKoimMsAA-INFgBfBatKjYDIcdbXUA2AIqTJozabSABGzCaCEPaHSAnM5fK541rYdjYdp7ACCNhsxg+1PRsLhsh5snCmFAABEMBg0ABpDAoSAAeQBAB4kHNQFKUKAMAAPZiweiQUAAa2lpAByHqhuNpqQAD56ui1RrtV89aAKfgEOjIcgANpqgC6jp1LrdcEQXodAwABgASRpq2H6WOi8VquWKpC+6V+62wyPogawDAUCyyAURZBVdKPB0NZOS6VphWCjAm1jsaoo60AblLzd+lVDstgNBQAClILUGiGEKAAD5NUDeoykTC4cIDafwP0DH4DhBDkfj2qwnulTIVStMFgNAAUlXbVd4aoGSEvKLVAEoN3vEHVbRtNjKClQGMAkaGIBp7zSGpYDVdE8X0SA0DoQgbwAIn0NCP3gzZ9GMegAkZG8cNpUAbwAKwnWAVWXVdwgAOSoGQP3qW1m1bSjJzqBpBlIGxyIwZFBlAL1ONgJdcBXCwGKYjAAwGC5bgBOAMHoBFSNpKCO0efh9V3d14APMcqLnUBFIwZTC3oD8ezxYxCACXBvnY01QICcD6m4oZN2Er03I8gYtMfWDpR7E9ezfDAb0GZZVngfRtgkQYbIFSLoti+x4sUDAqFuXBkp7IA).


## Testing

Looking at the implementation, some issues stand out that need to be addressed:

- The definition is quite complex and not obvious at first sight.
- Check if the implementation works as expected, in the good case as well as in the bad case.
- Refactorings should not lead to new errors.
- Document the use of `DeepKeysOf`.

All these points apply in general to any code, not only to type definitions. The key is to write unit tests. And this is exactly what I want to do for TypeScript's type definitions.

The big difference to usual unit tests is that type definitions exist only at compile time. This implies that the tests also have to be executed by the compiler rather than at runtime like all other unit tests. This sounds complicated, but is actually fairly easy. In the following I want to briefly introduce the possibilities with `ts-expect-error` and `expect-type`.


### ts-expect-error

`ts-excpect-error` was introduces with [TypeScript 3.9](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html#-ts-expect-error-comments). This instructs TypeScript to expect a compiler error on the following line. If a line is preceded by a `// @ts-expect-error` comment, TypeScript suppresses the reporting of this error; but if there is no error, TypeScript reports that `// @ts-expect-error` was not necessary.

```typescript
translate('dialog.title');

// @ts-expect-error
translate('dialog.header');
```

The actual type definition of `DeepKeysOf` can be tested without using the `translate` method:

```typescript
// @ts-expect-error
const invalidKey: DeepKeysOf<typeof translation> = 'dialog.header';
```

A complete example can be found at [TypeScript Playground](https://www.typescriptlang.org/play?#code/PTAEBUFMGcBcEsB2BzCBPADpAygYwE7waygAikAZkvAgPaLQBQIoAFrLBtAFwiyu0AtgENoAOgz5I8aNAA2kfGMGRgAIzm1kwAEwAGHTuB6ALLoBswWDAQoAtLEwwCRWHYAmlanQaNGuejhQWHxhBjlhH1AAXlAAb0ZQNkg5TQB1Wnw5d25QAHIACWFU2lA0lNgAQjyAGkTQd3hirVyEpKSEWAVcvIAhGwAThGQAVxRa+qTPaBdieHoegFkAN9x2SERQbHhIUApM2AphVilEAH48+oBfOqThXB8eeMnQNRGOQNaXpOgRtUEaD1+nAhvBkBsJu12rgwrgUj0AIJqNRSNYQl5Xa6MTGMRxYMiQSAYADSkDQ0AA8hQADzgGqgUloUCQAAe1kQ7mgoAA1mTaBQIDEeXyBeAAHwxF6M5lsjac0BwQgoF5nCAAbUZAF0Zez5YqkKhVdLcgADAAkcUZVzEFvIRMZlJp4A1ZM1YquJpeuUQkAAboo-HjduBQuFIvNENLYnaSWTHdSg-zgqH5OH6GKANyBpxbEIGimIORoABS0HoQv1KFAAB94qA1ZJaFh8I5cpXkJrctg8ygC0XS+Wrln-IESCEwqnrEKABTjsM+Rm5EMTiILskASjbPdQ0QlbWho9AUl+chIsTnk4jjO+7TE0Awcho04ARGJn+ub0kxFJ3CM4dPPyhUBpwAKzLRB6UbZtHAAOWEFR1xiCVEwFMDy2iDD8loNQQMgB48lAVU0MQBt8CbRRYPgyBtVyMZPCoH13FuICWIvVcI1AURcyVZA+xLcCa1AOivEY9csyhKRYBGfBNhQo8YBGU8YkwvJ2wI1Vj0UkhcjYtNIzJLMhz8XEU1XSBpzyRpmmQMROgUPIxOYMAAAFYGgOxWSwB4PPwMj8BMldInMyymk0GzWEgYRPHwBzhwCBgSF9Yp4HcRcCXtOMqQTJwk10nwJViELrNsmh7KzFhXPczy8LcRQ-JHBLQCQJLH1SslchjB0srkvKIwK-IrLCsQIqixQ8izIA).


This is a very simple approach and can be used without additional libraries. However, the solution is not very verbose, in unit test usually the expected result is explicitly given.


### expect-type

[expect-type](https://www.npmjs.com/package/expect-type) is a library developed for exactly this purpose: Testing type definitions.

```typescript
const translation = {
  dialog: {
    title: 'Bestätigung',
    description: 'Möchten Sie fortfahren?'
  }
}

type TranslationKey = DeepKeysOf<typeof translation>;

expectTypeOf<'dialog.title'>().toMatchTypeOf<TranslationKey>();
expectTypeOf<'dialog.header'>().not.toMatchTypeOf<TranslationKey>();
```

In my opinion `expectTypeOf` is a very smart solution, but it adds another dependency to the project.
