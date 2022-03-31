---
title:  "Using TypeScript to validate translations at compile time"
date:   2022-03-30 21:36:32 +0200
excerpt: "Translations stored in JSON can be validated at compile time with a TypeScript type definition to avoid runtime errors."
header:
  og_image: /assets/images/2022-03-30/using-typescript-to-validate-translations-at-compile-time.png
---


In web applications as I know them, translations are mostly stored in JSONish format and accessed at runtime. Usually, the setup is more complex since the applications needs to support multiple translations. But for now let's keep it as simple as possible.

```typescript
const translation = {
  helloWorld: 'Hallo Welt!',
  dialog: {
    title: 'Bestätigung',
    description: 'Möchten Sie fortfahren?'
  },
  actions: {
    buttons: {
      submit: 'Bestätigen',
      cancel: 'Abbrechen'
    }
  }
}
```

To retrieve and display the translation at runtime there is a method called `translate` which returns the corresponding translation for a key from the translation object:

```typescript
type StringOnlyJson = string | { [property: string]: StringOnlyJson };

const translate = (translationKey: string): string => {
  const result = translationKey
    .split('.')
    .reduce(
      (json, propertyName) => typeof json === 'object' ? json[propertyName] : undefined,
      translation as StringOnlyJson | undefined
    );
  return typeof result === 'string' ? result : translationKey;
};
```

If the method gets called with `dialog.title`, it will return `Bestätigung`:

```typescript
translate('dialog.title'); // returns 'Bestätigung'
```

But what happens if an invalid or incorrect translation key is passed to the method? The method cannot find the corresponding translation and returns the passed key as a fallback. In the following example `dialog.header` is passed instead of `dialog.title`.

```typescript
translate('dialog.header'); // returns 'dialog.header'
```

In my experience this error pattern occurs quite often. A developer simply makes a typo in the translations or changes the naming of a translation key without adjusting them at each source code location. This results in the user seeing only the technical key instead of the expected translation. Such a fallback is helpful because the user can likely continue working with the application instead of getting a blank label or even worse an error message.

![Dialog with missing translation](/assets/images/2022-03-30/dialog.png)

As mentioned at the beginning, translations are usually loaded at runtime. Therefore, such errors do not occur earlier than at runtime. To detect and avoid such defects, an extensive test suite or a high manual testing effort is required. In worst case, such defects occur in production and are displayed to the end user.

For this reason, errors should be found as early as possible and that is usually at compile time.

Wouldn't it be great if the translation keys could be checked automatically? Let's jump into the power of TypeScript.

In TypeScript there are these String Literal Types. Isn't it possible to check the translation keys at compile time and inform the developer about his mistake? It would only need a list of all possible translation keys. That is worth a try:

```typescript
type TranslationKey =
  | 'helloWorld'
  | 'dialog.title'
  | 'dialog.description'
  | 'actions.buttons.submit'
  | 'actions.buttons.cancel';
```

Afterwards the signature of the `translate` method can be changed to use the type `TranslationKey` for the `translationKey` parameter instead of just `string`:

```typescript
const translate = (translationKey: TranslationKey): string => {
  // same code as above
};
```

This change leads to the fact that the method `translate` must only be called with one of the previously defined values. All other strings are treated as errors by the compiler.

Back to the example from above. What happens if the `translate` method gets called with the correct `dialog.title` and the incorrect `dialog.header`?

```typescript
translate('dialog.title'); // compiles fine
translate('dialog.header'); // compiler error: Argument of type '"dialog.header"' is not assignable to parameter of type 'TranslationKey'.
```

The compiler gives an error message on the second call. The program does not compile and the developer is forced to correct the mistake.

This solution works very well and is easy to implement. Problem solved. :-)

Well...

This solution requires the developer to maintain all translation keys twice: once in the actual translation and a second time in the definition of the `TranslationKey` type. These two definitions must always be kept in sync to avoid the above mentioned errors of missing translations. This process is tedious, error-prone and in the end does not lead to any improvement.

Is there no way to create the `TranslationKey` type automatically? The TypeScript compiler would only have to extract the translation keys from the JSON object and concatenate them with a dot.

Indeed, TypeScript can derive the `TranslationKey`!

```typescript
type DeepKeysOf<T, Key extends keyof T = keyof T> = Key extends string
  ? T[Key] extends string ? Key : `${Key}.${DeepKeysOf<T[Key]>}`
  : never;

type TranslationKey = DeepKeysOf<typeof translation>;
```

The type `TranslationKey` is identical to the manual definition from above. The behavior of the compiler is as well identical, a call to the `translate` method with an incorrect translation key will raise an error.

But what exactly does the `DeepKeysOf` type do? Let's look at the crucial part first:

```typescript
T[Key] extends string ? Key : `${Key}.${DeepKeysOf<T[Key]>}`
```

`T` is the type definition of the translation object, `Key` is a property of the translation object, `T[Key]` is therefore the value of this property. `T[Key]` can be either a `string`, like `'Hello World!'` or another object, like the value of `dialog`. In the first case, the execution can be stopped and the result is simply `helloWorld`. In the second case a recursion is called, which adds `.` to the `Key` and again uses the type definition `DeepKeysOf` for the object of `T[Key]`. With the help of this recursion it is possible to use arbitrarily deeply nested translation objects.

A practical example for `DeepKeysOf<typeof translation>`: `T` is the entire translation object, `Key` is a property of this, i.e. `helloWorld`, `dialog` or `action`. `T[Key]` is the value of this property, for the `Key` `helloWorld` it is `'Hello World!'` , for `dialog` it is the object `{ title: 'Bestätigung', description: 'Möchten Sie fortfahren?' }`. Thus, if `Key` is `helloWorld` then the expression `T[Key] extends string` holds true and thus the result of the expression will be `helloWorld`. On the other hand, if `Key` is `dialog`, then `T[Key]` is an object, the expression holds false, and the result is a concatenation of `dialog.` (including the dot) with the result of `DeepKeysOf<T['dialog']>`.

However, it still has to be clarified how to iterate through the different properties within an object. For this purpose `keyof` and a type alias named `Key` is used: `Key extends keyof T = keyof T`. `keyof T` is an alias for all properties of the object `T` and allows in that way an iteration through all properties. `Key` then contains the current property selected by the iteration through `keyof T`. The actual iteration is performed by TypeScript itself.

As a last point there is the wrapper `Key extends string ? ... : never` around the actual expression (abbreviated by ...). In the translation object `Key` is always a `string`, so this expression is actually not relevant. But TypeScript does not actually know this, because this has not been defined. But for the later concatenation with `.` TypeScript expects a `string` (or several other types). By the way, the else branch with the result `never` is not called with the translation object. But if the object would contain some keys which are not `string` (i.e. `number`, or similar), then using `never` the corresponding invalid branches in the input object would be ignored.

This entire implementation is also available for simple follow up on the [TypeScript Playground](https://www.typescriptlang.org/play?#code/PTAEFUGcEsDsHNQBUCeAHAppAxgJ2mgC6iED2oAbgIYA20AJlYRiblbJDU9KR6E6GykAtmmg0WhaMIwAoEKAAWhQmkgAuEIUUiqkAHRpcGaJE4Zc+mcABGNUvGAAmAAxOnwFwGZgXl8ABXGAQAWkJ0LDwCQjDSEOo6RmYwtg4uKV5IEKYQoVFxDDDpOVkhDmJCVM5uXlAAXlAAb1lQJQwaewB1UlwaenVQAHIACVp7UE72wgBCQYAaFtB6aFoHAebW1qlCCQHBgCEsQgATqXgAhHnF1vpI-CIeWD2AWQA37GUMWFAAZWgWABmPUIAKoimMsAA-INFgBfBatKjYDIcdbXUA2AIqTJozabSABGzCaCEPaHSAnM5fK541rYdjYdp7ACCNhsxg+1PRsLhsh5snCmFAABEMBg0ABpDAoSAAeQBAB4kHNQFKUKAMAAPZiweiQUAAa2lpAByHqhuNpqQAD56ui1RrtV89aAKfgEOjIcgANpqgC6jp1LrdcEQXodAwABgASRpq2H6WOi8VquWKpC+6V+62wyPogawDAUCyyAURZBVdKPB0NZOS6VphWCjAm1jsaoo60AblLzd+lVDstgNBQAClILUGiGEKAAD5NUDeoykTC4cIDafwP0DH4DhBDkfj2qwnulTIVStMFgNAAUlXbVd4aoGSEvKLVAEoN3vEHVbRtNjKClQGMAkaGIBp7zSGpYDVdE8X0SA0DoQgbwAIn0NCP3gzZ9GMegAkZG8cNpUAbwAKwnWAVWXVdwgAOSoGQP3qW1m1bSjJzqBpBlIGxyIwZFBlAL1ONgJdcBXCwGKYjAAwGC5bgBOAMHoBFSNpKCO0efh9V3d14APMcqLnUBFIwZTC3oD8ezxYxCACXBvnY01QICcD6m4oZN2Er03I8gYtMfWDpR7E9ezfDAb0GZZVngfRtgkQYbIFSLoti+x4sUDAqFuXBkp7IA).

In the end, this solution is very powerful, thanks to TypeScript's extensive type system. Translations are no longer as error-prone as I know from my past. Overall, a single type definition increases the quality of the software and this quality can also be checked automatically at compile-time.

Comments are welcome on [Twitter](https://twitter.com/TheThomasPr/status/1509516711434895364) or [LinkedIn](https://www.linkedin.com/posts/thomas-preissler_using-typescript-to-validate-translations-activity-6915282669471698944-PGmU).
