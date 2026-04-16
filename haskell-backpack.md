# GHC Backpack: Module-Level Dependency Injection

Backpack is GHC's module-level dependency injection system. A library declares a
**signature** (`.hsig`) for a module it needs but doesn't provide. Consumers supply
a concrete implementation to **instantiate** the library.

## Core Concepts

### Signature files (.hsig)

A signature declares the interface a module must satisfy:

```haskell
-- src-sig/MyFramework/App.hsig
signature MyFramework.App where

import MyFramework.Lifecycle (AppContext)
import MyFramework.Widget (Widget)

appContext :: AppContext
appView :: IO Widget
```

### Indefinite vs Definite libraries

- **Indefinite**: has `signatures:` in its cabal stanza — cannot be compiled standalone
- **Definite**: all modules are concrete — compiles normally

```cabal
-- Indefinite library (has a signature hole)
library
  signatures: MyFramework.App
  exposed-modules: MyFramework
  hs-source-dirs: src src-sig
  build-depends: my-framework:my-framework-lifecycle

-- Definite sub-library (no signatures)
library my-framework-lifecycle
  exposed-modules: MyFramework.Lifecycle
  hs-source-dirs: src-lifecycle
  visibility: public
```

### Instantiation

When a package depends on both an indefinite library and a package providing
the required module, cabal automatically instantiates:

```cabal
-- Consumer provides the implementation module
library myapp-impl
  exposed-modules: MyFramework.App  -- satisfies the signature
  build-depends: my-framework:my-framework-lifecycle

-- Instantiation library: depends on both indefinite lib + impl
library myapp
  reexported-modules: MyFramework, MyFramework.App
  build-depends:
      my-framework,          -- indefinite (has signature hole)
      myapp:myapp-impl       -- provides MyFramework.App
```

Cabal's solver sees that `myapp-impl` provides `MyFramework.App`, which fills
the signature hole in `my-framework`, producing a fully instantiated `MyFramework`.

## Sub-library Architecture

Backpack libraries commonly split into sub-libraries to separate definite from
indefinite code:

```
my-framework/
  src-lifecycle/     -> library my-framework-lifecycle  (definite, visibility: public)
  src-ui/            -> library my-framework-ui         (definite, visibility: public)
  src-sig/           -> signature MyFramework.App       (part of main indefinite lib)
  src/               -> library my-framework            (indefinite, imports signature)
  default-app/       -> library my-framework-default-app (definite, default impl)
```

**Key rule**: `visibility: public` on sub-libraries is required for external packages
to depend on them via `build-depends: my-framework:my-framework-lifecycle`.

## Avoiding Backpack When You Don't Need It

Often, the indefinite main library is only needed for:
1. FFI bridge functions that re-export through the instantiated module
2. A convenience re-export aggregating sub-library + app modules

If your consumer only needs functions from definite sub-libraries plus its own
implementation module, you can **skip the indefinite dependency entirely**:

```cabal
-- BEFORE: depends on indefinite lib (requires Backpack instantiation)
library myapp
  reexported-modules: MyFramework, MyFramework.App, ...
  build-depends:
      my-framework,                           -- indefinite!
      myapp:myapp-impl,
      my-framework:my-framework-lifecycle

-- AFTER: only definite deps (no Backpack needed)
library myapp
  reexported-modules: MyFramework.App, MyFramework.Lifecycle, ...
  build-depends:
      myapp:myapp-impl,                         -- provides MyFramework.App
      my-framework:my-framework-lifecycle,       -- definite sub-lib
      my-framework:my-framework-ui               -- definite sub-lib
```

Then update imports in consuming code:
```haskell
-- BEFORE: importing from the instantiated indefinite module
import MyFramework (appContext, platformLog)

-- AFTER: importing from definite sources directly
import MyFramework.App (appContext)           -- own impl module
import MyFramework.Lifecycle (platformLog)    -- definite sub-lib
```

This is especially valuable when **only the desktop build** uses cabal and the
**mobile/embedded build** compiles sources directly with raw GHC (bypassing cabal
entirely).

## Nix Integration

### The callCabal2nix Problem

`callCabal2nix` (and nixpkgs Haskell infrastructure generally) uses `Setup.hs`
which **cannot perform Backpack instantiation**. Only `cabal v2-build` can do that.
This is nixpkgs issue [#40128](https://github.com/NixOS/nixpkgs/issues/40128).

**Symptoms:**
- "Could not resolve dependencies" errors mentioning indefinite libraries
- Sub-library visibility errors: "does not contain library 'foo'"

### Workarounds

#### 1. Remove the indefinite dependency (preferred)

If your package doesn't actually need the instantiated module, restructure to
depend only on definite sub-libraries (see section above). Then `callCabal2nix`
works normally:

```nix
# nix/hpkgs.nix
pkgs.haskellPackages.override {
  overrides = hnew: hold: {
    my-framework = hnew.callCabal2nix "my-framework" sources.my-framework {};
    myapp = hnew.callCabal2nix "myapp" ../. {};
  };
}
```

#### 2. Source injection via cabal.project

Include both packages in a single cabal project so `cabal v2-build` handles
instantiation:

```
# cabal.project
packages:
  .
  ./my-framework-src/
```

```nix
# shell.nix — symlink sources in
shellHook = ''
  ln -sfn ${sources.my-framework} my-framework-src
'';
```

Downside: `nix-build` can't use this pattern (only dev shells).

#### 3. Raw GHC compilation (cross builds)

For cross-compilation targets, bypass cabal entirely and compile with raw
GHC flags, providing source directories explicitly:

```bash
ghc -i${src}/src -i${src}/src-lifecycle -i${src}/default-app \
    -this-unit-id my-framework-1.0.0 \
    ...
```

This sidesteps the entire Backpack resolver — GHC compiles everything as
concrete modules since all sources are provided together.

## Cabal Internals

### ModuleShape

Backpack's type-level information flows through `ModuleShape`:
- `modShapeProvides`: modules and signatures the component provides
- `modShapeRequires`: signature holes that need filling
- `modShapeRequiresDecls`: source locations of signature declarations (for error messages)
- `modShapeDefinedNames`: module names defined in source (not from deps)

### ConfiguredComponent

After dependency resolution, components carry Backpack metadata:
- `cc_includes`: what modules are brought in from dependencies
- `cc_hsig_decls`: declaration info for `.hsig` files in this component
- `cc_defined_names`: which modules this component actually defines in source

### LinkedComponent

The linker matches signature requirements against providers:
- Checks that every required signature is provided by exactly one dependency
- Produces fully instantiated unit IDs with concrete module mappings

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "Unfilled requirement" | No dependency provides the signature module | Add a dependency that exposes the required module name |
| "does not contain library 'X'" | Sub-library not registered in package DB | Use `visibility: public` in the library's cabal file; or use source injection |
| "Could not resolve dependencies" with indefinite lib | callCabal2nix can't do Backpack | Remove indefinite dep or use source injection |
| "Duplicate module" | Two deps provide same module name | Check reexported-modules for conflicts; ensure only one instantiation path |

## Testing Backpack Libraries

- Unit tests for **definite** sub-libraries work normally.
- Tests for the **instantiation** library need a concrete implementation in scope.
- Put tests in a `test-suite` that depends on the instantiation library, not the
  indefinite one.
- The test suite itself is always definite (no signatures in test code).
