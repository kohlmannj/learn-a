# Lerna + Project References

This is a "bare minimum" repo that shows one way to configure TypeScript Project References with lerna. There are a lot of different ways you can set things up and this isn't intended to be authoratitive guidance or exclusionary of other ways that might work better in your project.

# Install lerna

```sh
yarn global add lerna
```

# Setting up this repo

```
> git clone https://github.com/kohlmannj/learn-a.git
> cd learn-a
> yarn bootstrap
> yarn build
```

Note that this project and its packages specify the [TypeScript 3.0 Release Candidate][1] (`^3.0.0-rc`) as a devDependency.

### General Structure

As with a normal lerna repo, there's a `packages` folder. Inside we have three creatively named packages `pkg1`, `pkg2`, and `pkg3`.

```
packages/
| tsconfig.settings.json
| tsconfig.json
| pkg1/
  | tsconfig.json
  | src/
  | | (typescript files)
  | lib/
  | | (javascript files)
  | | (.d.ts files)
| pkg2/
  | (same as pkg1)
| pkg3/
  | (same as pkg1)
```

Let's review each file in the repo and explain what's going on

#### `tsconfig.settings.json`

```js
{
    "compilerOptions": {
        // Always include these settings
        "composite": true,
        "declaration": true,
        "declarationMap": true,
        "sourceMap": true,

        // These settings are totally up to you
        "target": "es6",
        "module": "commonjs",
        "strict": true,
        "esModuleInterop": true,

        // Emit declarations only since we're using Babel to transpile
        "emitDeclarationOnly": true
    }
}
```

This file contains the "default" settings that all packages will use for compilation. You will definitely want the `composite`, `declaration`, `declarationMap`, and `sourceMap` settings enabled for all projects, so include those in this file. Other settings, like `target` and `strict`, can be specified here if you'd like to enable them by default. Also, since we'll be **using Babel 7 to transpile our TypeScript files**, we've added `emitDeclarationOnly`. This means that `tsc` emits only declarations and declaration maps, leaving the `.js` files to Babel. You'll also be able to override these settings on a per-package basis if needed.

#### `tsconfig.json`

```json
{
  "files": [],
  "references": [{ "path": "pkg1" }, { "path": "pkg2" }, { "path": "pkg3" }]
}
```

This file is pretty simple - simply list the packages that need to be built with `tsc` in the `references` array.
You should also include `"files": []` in this file - this will prevent an incorrect invocation of `tsc` without `-b` from trying to build the entire packages folder source files as one compilation (which will fail, but drop a bunch of .js files in random places as an annoying side effect).

#### `packages/pkg2/tsconfig.json`

We'll just cover one of the `pkg1` / `pkg2` / `pkg3` packages since they're basically identical for the purposes of this demo. Here's `pkg2`'s `tsconfig.json`:

```json
{
  "extends": "../tsconfig.settings.json",
  "compilerOptions": {
    "outDir": "lib",
    "rootDir": "src"
  },
  "references": [{ "path": "../pkg1" }]
}
```

The `extends` property pulls in the settings we wrote in `tsconfig.settings.json`, so we don't have to duplicate any settings described there.

In `compilerOptions`, we've set `outDir` to `lib` and `rootDir` to `src`, then placed all my `.ts` files in `src`. This means `src/index.ts` will build to `lib/index.js` and `lib/index.d.ts`. This is also the place where you could override settings like `strict` or `target` if you needed to change them on a per-project basis.

In the `references` array, we list the paths to the other projects' `tsconfig.json` files (or containing folders, as shown here). This will both ensure that we locate the `.d.ts` files correctly, and set up a proper build ordering.

#### `packages/pkg2/src/index.ts`

```ts
import * as p1 from "@ryancavanaugh/pkg1";

export function fn4() {
  p1.fn();
}
```

Nothing unusual going on here. We import and export with the usual syntax. Notably, if you open this repo in an editor, you can still "Go to Definition (F12)" on `p1.fn` here and land in `pkg1/foo.ts` - the original sourcecode - even though "under the covers" it's using the much faster `.d.ts` file for typechecking.

#### `packages/pkg2/package.json`

Here are the relevant excerpts from the `package.json`:

```json
{
  "main": "lib/index.js",
  "typings": "lib/index.d.ts",
  "scripts": {
    "prepublishOnly": "tsc -b ."
  },
  "devDependencies": {
    "typescript": "^3.0.0-dev.20180626"
  }
}
```

Because we build to `lib`, we need to set `main` to the `.js` file there _and_ `typings` to the `.d.ts` file.

In `scripts`, we use the local copy of `tsc` (listed here as a dev dependency) to run a _build mode_ compilation on the project. This will ensure that the `lib` folder is always built before `npm publish`, and blocks any publishes that try to push non-compiling code.

#### `packages/pkg2/.npmignore` / `packages/pkg2/.gitignore`

_.gitignore_

```
lib/
```

_.npmignore_

```
# Empty, but needs to exist
```

The `.gitignore` stops us from checking in build outputs, which is generally a good idea. By default, `npm` won't publish files that are ignored by `git`, so we need a separate `.npmignore` file so that the `lib` folder still gets published!

# Build Workflow

All your lerna commands and workflow will work as expected here.

To build the TypeScript projects, you can run individual builds with `tsc -b`:

```
 > lerna exec --scope=@ryancavanaugh/pkg3 -- yarn build
```

Or just build everything:

```
 > yarn build
```

**Note:** we use [concurrently][2] to run `babel` and `tsc` in parallel. (We can’t use `--kill-others`, however, since `tsc -b` appears to exit with code `null` when there’s no work to do.)

# Dev / Watch Mode Workflow

You can also run both Babel and the TypeScript compilers in watch mode during development.

Watch an individual package:

```
 > lerna exec --scope=@ryancavanaugh/pkg3 -- yarn dev
```

Since TypeScript knows that `pkg3` depends on both `pkg1` and `pkg2`'s type definitions, you'll notice that the above command results in the following output:

- `pkg1` and `pkg2`: type declarations (`.d.ts`) and declaration map files (`.d.ts.map`)
- `pkg3`: _transpiled JavaScript files_ (`.js`), type declarations (`.d.ts`), and declaration map files (`.d.ts.map`)

Watch all packages:

```
 > yarn dev
```

[1]:	https://blogs.msdn.microsoft.com/typescript/2018/07/12/announcing-typescript-3-0-rc/
[2]:	https://www.npmjs.com/package/concurrently