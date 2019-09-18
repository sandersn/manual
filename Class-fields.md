# Where We Are

## In 3.6

1. Allow accessors in d.ts files.

## For 3.7

1. Emit accessors in d.ts.
2. Two new errors: one for Define semantics, one for uninitialised properties.
3. Add a flag to emit Define when on, silence the new errors when off.

# Plan 

## 3.6

- #32787: allow, but do not generate, accessors in declaration files
- No changes to .js emit yet
- No changes to .d.ts emit yet

# 3.7

- #33470: always generate accessors in declaration files
- #33401: accessors may not override properties and vice versa.
- #33423: uninitialised properties may not override properties.
- New flag, `legacyClassFields`, to switch between set-vs-define .js emit.

## Disallow accessor/property overrides

- Properties can only override properties
- Accessors can only override accessors
- Except when the base is abstract or an interface
  - abstract: nothing there to interfere
  - interface: we pretend there's not
- Open question: whether to provide codefixes for either error.

### Property overrides accessor

- Property overriding accessor doesn't work with Define.
- Assuming that 'work' means that the base gets to do something clever.

<!--
- "Some" TC39 "members" think this is a "feature", citing locality.
- Have these same people ever used a class? The whole reason classes
  exist is to break locality.
- If I cared about locality I would use closures like a reasonable
  person.
-->


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
```

<!--

class DeviousUser extends CleverBase {
  constructor() {
    Object.defineProperty(this, "p", { value: "skips base setter, no caching" })
  }
}

-->

- SimpleUser is broken when switching to Define semantics.
- So we'll issue an error.
- Migrating is simple but not immediately obvious.
- Switch to an assignment in the constructor.

```ts
class SimpleUser extends CleverBase {
  constructor() {
    // base setter runs, caching happens
    this.p = "just fill in some property values"
  }
}
```

### Accessor overrides property

- Accessor overriding property with Set *almost* lets you avoid the
  base class property.
- But the base class still calls the derived accessor when setting the
  base property.
- Most people who do this will think they are being clever.
- And ... it doesn't work at all with Define.

```ts
class LegacyBase {
  p = 1
}
class SmartDerived extends LegacyBase {
  get() p {
    // clever work on get
  }
  set(value) p {
    // additional work to skip initial set from the base
    // clever work on set
  }
}
```

- SmartDerived is broken when switching to Define semantics.
- So we'll issue an error.
- Migrating is simple, but not obvious and does not have exactly the
  same semantics
- Specifically, "additional work to skip initial set" needs to be
  removed.
- If it was ever there in the first place...

```ts
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



<!--

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

-->

Notably, opinions differ on whether SimpleUser or DeviousUser is the
common case. I observed SimpleUser in the wild in Angular 2. I haven't
seen DeviousUser.

To avoid the new error, SimpleUser has to switch to a set in the
constructor. DeviousUser has to use a `defineProperty` in the constructor.

#### Accessor overriding property, with Set semantics:

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

### Breaks

Scattered all over large codebases. There are not a large number of
breaks, but a significant chunk (30%?) of RWC projects had failures.
Only a couple in user tests, which consist mostly of JS and therefore
don't get this error.

## Disallow uninitialised override properties

- Introduce new syntax, class C extends B { declare x: number }
- Introduce a codefix that inserts declare where needed.
- This declaration is erased, like the original declaration in old versions of TS
- Open question: syntax bikeshed

### Examples

### Details

### Caveats

### Breaks

(TODO 80%) of the uses I saw were *trying* to declare a the existence property or
refine its type. They want exactly this syntax, although they would be
unlikely to know it until given an error.
The remaining uses were declaring the property and initialising it in
the constructor. With strictNullChecks they would not have received
the error.

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

<!--

## Even More Examples

class CleverCachingBase {
    _value = 'cleverCachingBase'
    _clever = new String()
    get p() {
        console.log('... getting CleverCachingBase')
        // do something with _clever 
        return this._value
    }
    set p(value) {
        this._value = value
    }
}
class SimpleDerived extends CleverCachingBase {
    p = 'SimpleDerived'
}
var sd = new SimpleDerived()
console.log(sd.p)

////////
class SimpleWrongBase {
    p = 'SimpleWrongBase'
}
class CleverFixupDerived extends SimpleWrongBase {
    _value = 'CleverFixupDerived'
    get p() {
        console.log('... getting CleverFixupDerived')
        return this._value
    }
    set p(value) {
        this._value = value
    }
}
var cfd = new CleverFixupDerived()
console.log(cfd.p)

/////////
class LazyInitBase {
    // tag name
    get p() {
        console.log('... getting LazyDefaultBase')
        return 'LazyDefaultBase'
    }
    set p(value) {
        // throw after the first set
        console.log('... setting LazyDefaultBase to', value)
    }
}
class SimpleOverrideDerived extends LazyInitBase {
    p = 'SimpleOverrideDerived'
}
var sod = new SimpleOverrideDerived()
console.log(sod.p)

//////////
class CompositeBase {
    p = 'compositeBase'

}
class EmptyDerived extends CompositeBase {
    get p() {
        console.log('... getting EmptyDerived')
        return '0'
    }
    set p(value) {
        console.log('... setting EmptyDerived to', value)
    }
}
var ed = new EmptyDerived()
console.log(ed.p)

////////
class SupertypeBase {
    p: unknown
}
class SubtypeDerived extends SupertypeBase {
    p: number
}

-->

<!--

Define arguments

= implies Set -VS- Could come up with other syntax.
Babel and Typescript use Set with (almost) no problems.

JS programmers don't distinguish between imperative and declarative. -VS- git gud

Base classes may be incorrectly initialised if their setters are skipped.
People will be surprised, they will miss deprecation warnings, whole
frameworks will explode in flames.

-VS-

Object literal property declarations don't do this!
 -- LOL! This is just a more extreme git gud argument, in which you
 have to use moon logic to connect two points that are further away
 than the nearest logical ones.

Define leaves space for const properties, or types.
It's also the way that private fields (have to?) work.

The last point is the strongest. The other two are just garbage,
and there's no real answer for the arguments of familiarity and
existing, successful practise.

-->