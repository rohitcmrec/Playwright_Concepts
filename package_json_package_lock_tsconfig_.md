# package.json vs package-lock.json vs tsconfig.json

When working with a Node.js + TypeScript project, three files you will commonly see are:

- `package.json`
- `package-lock.json`
- `tsconfig.json`

They have different responsibilities.

---

## 1. What Is `package.json`?

`package.json` is the **main configuration file for a Node.js project**.

It describes your project and tells npm:

- What the project is called
- What version it is
- Which packages the project depends on
- Which packages are needed only for development
- Which commands/scripts can be executed

A simple example:

```json
{
  "name": "playwright-project",
  "version": "1.0.0",
  "scripts": {
    "test": "playwright test"
  },
  "dependencies": {
    "axios": "^1.7.0"
  },
  "devDependencies": {
    "@playwright/test": "^1.45.0",
    "typescript": "^5.5.0"
  }
}
```

### `dependencies`

These are packages required by the application when it runs.

Example:

```json
"dependencies": {
  "axios": "^1.7.0"
}
```

### `devDependencies`

These are packages required while **developing or testing** the project.

Example:

```json
"devDependencies": {
  "@playwright/test": "^1.45.0",
  "typescript": "^5.5.0"
}
```

For a Playwright automation project, Playwright and TypeScript are commonly placed in `devDependencies`.

---

## 2. What Happens When You Run `npm install`?

Suppose your `package.json` contains:

```json
"devDependencies": {
  "@playwright/test": "^1.45.0"
}
```

When you run:

```bash
npm install
```

npm reads `package.json` and installs the required packages.

It also creates/updates:

```text
node_modules/
package-lock.json
```

So the basic relationship is:

```text
package.json
      |
      | tells npm what packages are required
      ↓
npm install
      |
      ↓
node_modules/
      |
      ↓
package-lock.json
```

---

# 3. What Is `package-lock.json`?

`package-lock.json` records the **exact dependency versions that were installed**, including the dependency tree.

This is important because a package such as:

```text
@playwright/test
```

may itself depend on other packages.

So the dependency tree can look like:

```text
Your Project
    |
    └── @playwright/test
            |
            ├── package A
            |
            ├── package B
            |
            └── package C
```

`package-lock.json` records the resolved versions of these packages.

---

## 4. Why Do We Need `package-lock.json`?

Imagine your `package.json` contains:

```json
"typescript": "^5.5.0"
```

The `^` means npm can install a compatible newer version within the specified semver range.

For example, depending on the version range, a later `npm install` could resolve to a newer compatible release.

But `package-lock.json` records the exact version that was resolved.

For example:

```text
package.json
typescript: ^5.5.0

package-lock.json
typescript: 5.5.4
```

This helps ensure that different developers or CI systems install the same resolved dependency tree.

---

# 5. `package.json` vs `package-lock.json`

The easiest way to remember the difference:

| File | Purpose |
|---|---|
| `package.json` | Defines what your project needs |
| `package-lock.json` | Records exactly what npm resolved/installed |
| `node_modules` | Contains the actual installed packages |

Think of it like this:

```text
package.json
"What do I need?"

package-lock.json
"Exactly which versions did we resolve?"

node_modules
"Here are the actual packages."
```

---

# 6. Should `package-lock.json` Be Committed to Git?

For an application/project, generally **yes**.

You normally commit:

```text
package.json
package-lock.json
```

You normally do **not** commit:

```text
node_modules/
```

So a typical `.gitignore` contains:

```text
node_modules/
```

The reason is that `node_modules` can be recreated from the package files.

---

# 7. What Happens on Another Machine?

Suppose another developer clones your project.

They get:

```text
package.json
package-lock.json
```

but not:

```text
node_modules/
```

They run:

```bash
npm install
```

npm uses the dependency information and lockfile to recreate the dependency tree.

This helps keep the development environment consistent.

---

# 8. What Is `tsconfig.json`?

`tsconfig.json` is the **configuration file for TypeScript**.

It tells the TypeScript compiler:

> "How should I compile and type-check my TypeScript code?"

A simple example:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist"
  },
  "include": [
    "src/**/*.ts"
  ]
}
```

---

# 9. Why Do We Need `tsconfig.json`?

Suppose you have:

```text
src/
    login.ts
    test.ts
```

Your TypeScript files contain:

```typescript
const username: string = "Rohit";
```

TypeScript needs to know things such as:

- Which JavaScript version should be generated?
- Which module system should be used?
- Should strict type checking be enabled?
- Which files should be included?
- Where should compiled JavaScript files go?

These settings are defined in:

```text
tsconfig.json
```

---

# 10. Important `tsconfig.json` Options

## `target`

Defines the JavaScript version that TypeScript should compile to.

Example:

```json
{
  "compilerOptions": {
    "target": "ES2020"
  }
}
```

Conceptually:

```text
TypeScript
    ↓
JavaScript ES2020
```

---

## `module`

Defines the module system used by the generated JavaScript.

Example:

```json
{
  "compilerOptions": {
    "module": "commonjs"
  }
}
```

or:

```json
{
  "compilerOptions": {
    "module": "ESNext"
  }
}
```

---

## `strict`

Enables strict TypeScript type checking.

Example:

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

This is very useful because TypeScript will catch more potential errors during development.

For example:

```typescript
let username: string = "Rohit";

username = 123;
```

With strict/type checking enabled, TypeScript can identify that `123` is not a string.

---

## `esModuleInterop`

Helps with compatibility between CommonJS and ES module imports.

Example:

```json
{
  "compilerOptions": {
    "esModuleInterop": true
  }
}
```

---

## `outDir`

Specifies where compiled JavaScript files should be placed.

Example:

```json
{
  "compilerOptions": {
    "outDir": "dist"
  }
}
```

Conceptually:

```text
src/
    test.ts

        ↓ TypeScript compiler

dist/
    test.js
```

---

## `include`

Specifies which files TypeScript should process.

Example:

```json
{
  "include": [
    "src/**/*.ts"
  ]
}
```

This means TypeScript should include TypeScript files under the `src` directory.

---

# 11. `tsconfig.json` in a Playwright Project

In a TypeScript Playwright project, you may have something like:

```text
playwright-project/
│
├── tests/
│   ├── login.spec.ts
│   └── checkout.spec.ts
│
├── playwright.config.ts
├── package.json
├── package-lock.json
├── tsconfig.json
└── node_modules/
```

Here:

```text
package.json
```

defines the project and its npm dependencies/scripts.

```text
package-lock.json
```

locks the resolved dependency tree.

```text
tsconfig.json
```

controls how TypeScript behaves.

```text
playwright.config.ts
```

contains Playwright-specific configuration.

---

# 12. How These Three Files Work Together

Think about the complete flow:

```text
                    package.json
                         |
                         | Dependencies
                         ↓
                    npm install
                         |
                         ↓
                 package-lock.json
                         |
                         | Exact resolved
                         | dependency tree
                         ↓
                    node_modules
```

Separately:

```text
                    tsconfig.json
                         |
                         | TypeScript rules
                         ↓
                 TypeScript Compiler
                         |
                         ↓
                    JavaScript
```

So:

```text
package.json
    ↓
"What packages does my project need?"

package-lock.json
    ↓
"Which exact dependency versions were resolved?"

tsconfig.json
    ↓
"How should TypeScript understand/check/compile my code?"
```

---

# 13. Very Important: They Are Not the Same Thing

A common beginner confusion is:

> "If `package.json` already tells npm about the packages, why do we need `package-lock.json`?"

Because they serve different purposes.

### `package.json`

Defines the project's declared dependency requirements.

```text
I need TypeScript.
I need Playwright.
I need this version range.
```

### `package-lock.json`

Records the exact dependency resolution.

```text
This is the exact version of TypeScript
and these are the exact versions of its
dependencies that were resolved.
```

### `tsconfig.json`

Does not manage npm packages.

Instead, it controls TypeScript.

```text
Check my TypeScript this way.
Compile it this way.
Use these files.
Generate output here.
```

---

# 14. Easy Way to Remember

Think of a Playwright project as a car.

### `package.json`

The **shopping list**:

> What tools and packages does my project need?

### `package-lock.json`

The **exact receipt**:

> Which exact versions were actually resolved?

### `tsconfig.json`

The **instruction manual for TypeScript**:

> How should TypeScript check and compile my code?

And:

### `node_modules`

The **actual toolbox**:

> Here are the installed packages.

---

# Quick Revision

```text
package.json
→ Project metadata + dependencies + scripts

package-lock.json
→ Exact resolved dependency versions/tree

tsconfig.json
→ TypeScript compiler/type-checking configuration

node_modules
→ Actual installed npm packages
```

The most important distinction:

```text
package.json       → What I want
package-lock.json  → What was resolved
tsconfig.json      → How TypeScript should work
node_modules       → What is actually installed
```
