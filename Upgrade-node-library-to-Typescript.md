Once upon a time in grad school I got a
[real live Ph D](https://github.com/sandersn/dialect) in computational
linguistics, and programming languages were just my hobby. Back then I
bored my colleagues talking about lambdas all day. Now writing a
compiler is my job and linguistics is my hobby. So now it's the
Sapir-Whorf hypothesis that drives my colleagues to sneak away from the
lunch table while I keep talking.

Anyway, I forked the
[`natural`](https://github.com/NaturalNode/natural) package recently
to check out the state of natural language processing in the
Javascript world. I decided to upgrade it to Typescript so I would
know that I really understood the code. So far it's been more of an
exercise in upgrading a package to Typescript, so I thought you might
want to follow along to see what's involved. Here are the topics I
plan to cover:

1. Compile with Typescript
2. Switch to gulp
3. Actually upgrade to Typescript
4. Acquire types to make further development easier
5. Polish and make things stricter

You can follow
[the entire lurid history](https://github.com/sandersn/natural/commits/typescript)
on my fork of natural. That's probably too much detail, but I will
reference specific commits from time to time.

# Step 1: Compile with Typescript

This step is actually pretty easy. The first thing I did was add a
`tsconfig.json` to the root of the project:

```json
{
    "compilerOptions": {
        "allowJs": true,
        "outDir": "build"
    },
    "include": ["src/natural/**/*"],
    "exclude": ["lib", "node_modules"]
}
```

Then I made sure to copy everything from its original place in `lib`
to a new folder named `src`.

That's it! You can now run `tsc` at the package root and see the
Javascript files dropped into the `build` folder. Except for
formatting vagaries like 4-space vs 2-space tabs, the files should be identical.

OK, that was easy enough to get it [basically right in one commit](https://github.com/sandersn/natural/commit/24c32b9830a188a77d3f6911c3922ddcb49d244f), but
it's really incomplete. The files go to a new location, not `lib` like
they used to, which means that tests (and everything else) would have
to point to the new location. It doesn't convert tests. And it doesn't
copy data files, just .js files.

Also it's still Javascript! I just stuck `"allowJs": true` in the
tsconfig so that everything would Just Work without me having to
change anything.

Well, almost anything. Typescript complains about very few errors when
compiling Javascript, but it doesn't like dead code. So [I fixed](https://github.com/sandersn/natural/commit/c1698c4de4a56136288b1b8fbac270f90e1f69fe) a few
of those.

# Step 2: Switching to gulp

Honestly, this is the step that I know least about. I'm not 100% sure
I did the right things when setting up gulp, and I'm pretty sure I
have some gulp-malapropisms in my gulpfile. But I use a Windows
machine for recreational development (???), so I definitely couldn't
use the Makefile that came with `natural`.

Even if you already use gulp, you'll probably be interested in this
step because your gulpfile will look different after switching to
Typescript. And you can hunt for things to laugh at in mine.

Here it is:

```js
var gulp = require("gulp");
var ts = require("gulp-typescript");
var tsProject = ts.createProject("tsconfig.json");
var jasmine = require('gulp-jasmine-node');

gulp.task("copy-json", function() {
    return gulp.src("src/**/*json", { base: 'src' }).pipe(gulp.dest("lib"));
});
gulp.task("copy-txt", function() {
    return gulp.src("src/**/*txt", { base: 'src' }).pipe(gulp.dest("lib"));
});
gulp.task("copy-jg", function() {
    return gulp.src("src/**/*jg", { base: 'src' }).pipe(gulp.dest("lib"));
});
gulp.task("default", ["copy-json", "copy-txt", "copy-jg"], function () {
    return tsProject.src()
        .pipe(tsProject())
        .js.pipe(gulp.dest("lib/natural"));
});
gulp.task("test", ["default"], function () {
    return gulp.src(["spec/*js"]).pipe(jasmine({ verbose: false, timeout: 10000, color: true }))
});
```

Note that it's Javascript, not Typescript. I never put in the work to
learn how to set up a `gulpfile.ts` and this one is not big enough to
justify it.

By just giving you my final gulpfile, I'm glossing over a couple of
hours of flailing around that I needed to get files copied to the
right place, and then tests running on those files.
Let me explain what I ended up with.

There are 2 basic tasks: `default` and `test`. `default` relies on
some simple copy tasks, but its body is just some basic calls to the
`gulp-typescript` API which result in the output being copied to
`lib/natural`, where the Javascript source originally was. Note that
the `build` directory specified in the tsconfig is never actually
created!

`test` uses `gulp-jasmine-node` to kick off tests of the compiled
Javascript. This really isn't Typescript specific, but it still took me
a little while to get it working. (Apparently, you have to *return*
the gulp pipeline from the gulp task in order for globs to work correctly?)

Note that I had [to change a few lines of code](https://github.com/sandersn/natural/commit/87723cd5fc4185ffb10c99e54a36aacb344f35f5) to when I switched
from `jasmine-node` to `gulp-jasmine-node`. I guess fewer things get
dumped into the global namespace with `gulp-jasmine-node`.

Once I finally got gulp copying the compiled files and regularly
passing tests, I deleted the original source in `lib` from git and
added `lib` to `.gitignore`. `lib` is now just the build output directory.

# Step 3: Actually upgrade to Typescript

So. Finally, I was ready to start adding types to everything. Well,
almost. Actually, the first step is to find a nice small file to
rename. I chose `analyzers/sentence_analyzer.ts` because there's only
one file in the directory and the file itself isn't too big. Plus
there's decent test coverage, as far as I can tell.

## Fix imports/exports

I still wasn't to adding types yet, though. First I had to [fix the
imports and exports](https://github.com/sandersn/natural/commit/c1f14a0e0fd3d7f350d774852f58ad5ce2c8571c). This was pretty easy because Typescript has
specific support for Node modules. I didn't have to adapt things to
use ES6 modules, at least yet.

Here is the changed import:

```ts
var _ = require("underscore")._;
// becomes
import _ = require("underscore");
```

and here's the changed export:

```ts
module.exports = Sentence;
// becomes
export = Sentence;
```

Using ES6 modules would be more complicated because underscore would
need a default export (which it might) and Sentence would probably
become a default export too, meaning that its users would need to change.

## Upgrade to ESNext features

For the first change, I decided to turn the `Sentences` class into a real ES6
class. It changed from this:

```ts
var Sentences = function(pos, callback) {
    this.posObj = pos;
    this.senType = null;
    callback(this);
};

Sentences.prototype.part = function(callback) {
    var subject = [],
    // ....
```

To this:

```ts
class Sentences {
    posObj: any;
    senType: any;
    constructor(pos, callback) {
        this.posObj = pos;
        this.senType = null;
        callback(this);
    }

    part(callback) {
        var subject = [],
        // ...
```

There are lots of new features in ES6 and above, and I won't talk about
them too much since there are lots of better places to learn about them.

## Add types

I also added types to the properties and parameters. I left the bodies alone for now,
for three reasons.

1. The Typescript compiler is pretty good at inferring the types already.
2. I can easily see all the places the compiler can't infer when I turn on `noImplicityAny: true`.
3. The scope of an individual type annotation is small and doesn't add much value compared to annotations of interfaces.

I started off with `part`, the first method, and looked for all uses
of its parameter `callback`:

```ts
part(callback) {
    var subject = [],
    // ...
    callback(this);
}
```

With only one use, the type is easy to infer:

```ts
part(callback: (t: this) => void) {
    var subject = [],
    // ...
    callback(this);
}
```

I could have also used `(t: Sentences) => void` but I had to think
less to use `this`. I added a couple more types like this and noticed
that the properties of `Sentence`, `posObj` and `senType`, were pretty
easy to infer from usage too.

TODO: Explain this inference process.


# Acquire types

At this point I upgraded the next directory alphabetically. This was
the brill_pos_tagger. I started at what I thought was a leaf file (one
with no imports), but it turns out that it had a dependency on log4js.

I have to admit, I got stuck here. I made the same transformation as
for underscore and got an error that the log4js module was not found.
After messing with a couple of module resolution options, I decided to
work around the problem by doing something I was going to do
eventually: install typings for the libraries that `natural` uses.

Turns out this is really easy:

```sh
$ npm install --save @types/log4js
```

Once I did that, the compiler started looking in `node_modules/@types/log4js`
instead of `node_modules/log4js` to get types and found the type
definitions from DefinitelyTyped. This got around the error I ran into.

# Addendum: Full outline

1. Add tsconfig
  a. Fix bare-minimum compile errors
2. Switch to gulp
  a. Copy things over to src/ from lib/
  b. Add tasks to compile, then copy
  c. Add tasks to copy other stuff
  d. Add task to test
3. Actually upgrade to TS
  a. Fix imports and exports
  a. Add types
  b. Add a types file
  c. Upgrade to ESNext features
4. Acquire types
5. Polish
  a. Upgrade to noImplicitAny
  b. Upgrade to strictNullChecks
Publish your own types
