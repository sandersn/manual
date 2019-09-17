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

#### Property overriding accessor, with Set semantics:

```ts
class CleverBase {
  _p: unknown
  get p() {
    return _p
  }
  set p(value) {
    // cache or transform or register or ...
    _p = value
  }
}
class SimpleUser extends CleverBase {
  // base setter runs, caching happens
  p = "just fill in some property values"
}
class DeviousUser extends CleverBase {
  constructor() {
    Object.defineProperty(this, "p", { value: "skips base setter, no caching" })
  }
}
```

Property overriding accessor, with Define semantics:

```ts
class CleverBase {
  _p: unknown
  get p() {
    return _p
  }
  set p(value) {
    // cache or transform or register or ...
    _p = value
  }
}
class SimpleUser extends CleverBase {
  constructor() {
    // base setter runs, caching happens
    this.p = "just fill in some property values"
  }
}
class DeviousUser extends CleverBase {
  p = "skips base setter, no caching"
}
```

Notably, opinions differ on whether SimpleUser or DeviousUser is the
common case. I observed SimpleUser in the wild in Angular 2. I haven't
seen DeviousUser.

To avoid the new error, SimpleUser has to switch to a set in the
constructor. DeviousUser has to use a `defineProperty` in the constructor.

#### Accessor overriding property, with Set semantics:

```ts
class LegacyBase {
  p = 1
}
class SmartDerived extends LegacyBase {
  get() {
    // clever work on get
  }
  set(value) {
    // additional work to skip initial set from the base
    // clever work on set
  }
}
class SmarterDerived extends LegacyBase {
  constructor() {
    Object.defineProperty(this, "p", {
      get() {
        // clever work on get
      },
      set(value) {
        // clever work on set
      }
    })
  }
}
```

Neither solution works with class field with Define semantics.
Well, SmarterDerived still works the same:

```ts
class LegacyBase {
  p = 1
}
class SmarterDerived extends LegacyBase {
  constructor() {
    Object.defineProperty(this, "p", {
      get() {
        // clever work on get
      },
      set(value) {
        // clever work on set
      }
    })
  }
}
```

The only way to avoid the error (without changing LegacyBase) is to
use defineProperty in the constructor. This avoids some confusing code
in SmartDerived (or subtly wrong, since the common case will be to
forget the additional work to skip initial set.)

I have no idea how to communicate to users which fix to choose. Or
even that set in the constructor is a common fix, or that
defineProperty is possible in the constructor.

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

1. Technically incorrect class hierarchies like Azure SDK, VS Code.
   They redeclare supertype properties with a derived type and then
   rely on bivariance.
1. Technically similar, but the base property type is any or unknown.
2. Redundant redeclarations, eg Babylon, F12, Firestore, Skype.
   This could be
   1. cut-and-paste
   2. a love of locality.
   3. not being sure whether the base defines the property.

   Either way the authors will be surprised by Define semantics and need to switch.

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
