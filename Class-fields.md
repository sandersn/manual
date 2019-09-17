# Plan (summary)

## 3.6

- #32787: allow, but do not generate, accessors in declaration files
- No changes to .js emit yet
- No changes to .d.ts emit yet

# 3.7

- #33470: generate accessors in declaration files
- #33401: accessors may not override properties and vice versa.
- #33423: uninitialised properties may not override properties.
- New flag, `legacyClassFields`, to switch between set-vs-define emit.

## Disallow accessor/property overrides

- Properties can only override properties
- Accessors can only override accessors

- Accessor overriding property is confusing with Set.
  (and most people who use it will think they are being clever)
- And it doesn't work with Define.
- Property overriding accessor doesn't work with Define either.
- Assuming that 'work' means that the base gets to do something
  clever.

<!--
- "Some" TC39 "members" think this is a "feature", citing locality.
- Have these same people ever used a class? The whole reason classes
  exist is to break locality.
- If I cared about locality I would use closures like a reasonable
  person.
-->

### Examples

### Details

### Caveats

### Breaks

## Disallow uninitialised override properties
Introduce new syntax, class C extends B { declare x: number }
Introduce a codefix that inserts declare where needed.
This declaration is erased, like the original declaration in old versions of TS
  Introduce a new flag, legacyClassFields.

### Examples

### Details

### Caveats

### Breaks

## Always emit accessors in d.ts

```ts
class C {
  get p1() { return 1 }
  get p2() { return 1 }
  set p2(value) { }
  set p3(value) { }
}
```

Now produces

```ts
declare class C {
  get p1(): number;
  get p2(): number;
  set p2(value: number);
  set p3(value: any);
}
```

Previously it produced

```ts
declare class C {
  readonly p1: number;
  p2: number;
  p3: any;
}
```

## Flag for emitting Object.defineProperty

- When `"legacyClassFields": true`:
  * Initialized fields in class bodies are emitted as constructor assignments (even when targeting ESNext).
  * Uninitialized fields in class bodies are erased.
  * The preceding errors are silenced.
- When `"legacyClassFields": false`:
  * Fields in class bodies are emitted as-is for ESNext.
  * Fields in class bodies are emitted as Object.defineProperty assignments in the constructor otherwise.
- In 3.7, "legacyClassFields" defaults to true`
- Declaration emit is the same regardless of the flag's value.

### Examples

### Details

When flag is false

