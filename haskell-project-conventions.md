# Haskell Project Conventions

Conventions for structuring Haskell projects with cabal, nix, and CI.
Based on [jappeace/haskell-template-project](https://github.com/jappeace/haskell-template-project).

## Key Principles

### 1. Library-centric layout

All real code goes in `src/` as library modules. The executable in `app/Main.hs`
is a trivial wrapper. Tests import the library.

```
my-project/
  app/Main.hs           -- trivial: import qualified MyLib; main = MyLib.main
  src/MyLib.hs           -- all real code
  src/MyLib/Internal.hs
  test/Test.hs           -- imports MyLib
  nix/
    pkgs.nix
    hpkgs.nix
  shell.nix
  default.nix
  my-project.cabal
```

### 2. Common stanza

Extensions, warnings, and base dependency are declared once and imported by all
components:

```cabal
common common-options
  default-language: GHC2021
  default-extensions:
    OverloadedStrings
    DeriveGeneric
    DerivingStrategies
  ghc-options:
    -Wall
    -Werror
    -Wunused-packages
    -Wunused-imports
  build-depends:
    base >= 4.14 && < 5

library
  import: common-options
  exposed-modules: MyLib
  hs-source-dirs: src

executable my-project
  import: common-options
  main-is: Main.hs
  hs-source-dirs: app
  build-depends: my-project
  ghc-options: -Wno-unused-packages

test-suite unit
  import: common-options
  type: exitcode-stdio-1.0
  main-is: Test.hs
  hs-source-dirs: test
  build-depends:
      my-project
    , tasty
    , tasty-hunit
  ghc-options: -Wno-unused-packages
```

### 3. Strict warnings

`-Wall -Werror -Wunused-packages` on everything. All warnings are errors.
Exe and test stanzas add `-Wno-unused-packages` to suppress false positives
from cabal dependency tracking.

### 4. Nix dependency chain

```
npins/ -> nix/pkgs.nix -> nix/hpkgs.nix -> shell.nix / default.nix
```

Uses `callCabal2nix` in `hpkgs.nix` overlay. No flakes.

```nix
-- nix/pkgs.nix
import (import ../npins).nixpkgs {}

-- nix/hpkgs.nix
{ pkgs }:
pkgs.haskellPackages.override {
  overrides = hnew: hold: {
    my-project = hnew.callCabal2nix "my-project" ../. {};
  };
}

-- shell.nix
let
  pkgs = import ./nix/pkgs.nix;
  hpkgs = import ./nix/hpkgs.nix { inherit pkgs; };
in
hpkgs.shellFor {
  packages = p: [ p.my-project ];
  buildInputs = [ pkgs.cabal-install ];
}

-- default.nix
let
  pkgs = import ./nix/pkgs.nix;
  hpkgs = import ./nix/hpkgs.nix { inherit pkgs; };
in
hpkgs.my-project
```

### 5. Dev speed

- Makefile overrides `-O2` with `-O0` for fast builds.
- `.ghci` uses `-fobject-code -O0` for fast reloads.
- `ghcid` runs tests on save.

```makefile
# makefile
OPTIMIZATION=-O0

build:
	cabal build --ghc-options="$(OPTIMIZATION)"

run:
	cabal run --ghc-options="$(OPTIMIZATION)"

test:
	cabal test --ghc-options="$(OPTIMIZATION)"
```

```
-- .ghci
:set -fobject-code -O0
:set -Wall
```

### 6. Test framework

Always `tasty` + `tasty-hunit`. Add `tasty-quickcheck` if properties make sense.
Test suite is always named `unit`.

### 7. Dual CI

Nix build on Ubuntu + cabal matrix across GHC versions and OSes. Both cancel
the workflow on first failure.

```yaml
# .github/workflows/ci.yaml (simplified)
jobs:
  nix-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cachix/install-nix-action@v24
      - run: nix-build

  cabal-matrix:
    strategy:
      matrix:
        ghc: ['9.6', '9.8', '9.10']
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: haskell-actions/setup@v2
        with:
          ghc-version: ${{ matrix.ghc }}
      - run: cabal test
```

## Setting Up npins

```bash
nix-shell -p npins --run "npins init --bare && npins add --frozen channel nixpkgs-unstable --name nixpkgs"
```

## Sub-libraries

When needed, add sub-libraries with `visibility: public` for external consumers:

```cabal
library my-sub-lib
  import: common-options
  exposed-modules: My.Sub.Module
  hs-source-dirs: src-sub
  visibility: public    -- required for external consumers
```

## Creating a New Project

1. Create directory structure: `src/`, `app/`, `test/`, `nix/`
2. Copy and adapt the template files, replacing `template` with the project name
3. Set up npins (see above)
4. Note: `hpkgs.nix` uses a nix name that may differ from the cabal package name
5. Verify: `nix-build` and `nix-shell --run "cabal test"`
6. Create a new repo — do NOT open a PR back to the template project
