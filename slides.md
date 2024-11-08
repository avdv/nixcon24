---
title: Buck2 + Nix
theme: dracula
highlight-theme: solarized
css: assets/style.css
revealOptions:
  transition: 'fade'
---
<!-- .slide: data-background-iframe="./slide1_bg.html" -->

## Integrating Buck2 and Nix for fun and profit

by Claudio Bley

<p style="font-size: smaller">NixCon 2024 (EU) &mdash; 26th October 2024</p>

Note: Me, scalable builds group, tweag. Polyglot build system.

---

## Motivation

* Mercury https://mercury.com/
* monorepo
* long deployment times
* increase developer happiness (fast turn-around, git branch switching)

---

## Collaboration

* Mercury, Ian-Woo Kim
* Tweag by Modus Create
   - T. Schmits, C. Shao, S. Visscher (GHC)
   - A. Herrmann, C. Bley (build)
* Well-Typed, M. Pickering, Z. Duggal

---

## Codebase

* 1.1M lines Haskell
* 10k modules
* uses ghc from nix (heavily patched) and haskell libraries from nix (not haskell.nix)
* uses flakes ❄
* Typescript and code generation

---

<img alt="" src="assets/buck2_logo.svg" width="20%" style="float:right; margin: 0 5% 2% 5%"/>

## Buck2

https://buck2.build/

<q>fast, reliable, extensible</q>

* polyglot build system from Meta
* supports distributed builds and caching (like Bazel)

Notes: still being worked on, no official release yet, just use latest snapshots, author of Build Systems a la cart paper

---

## Introducing buck2.nix

```
load("@nix//:nix.bzl", "nix")

nix.rules.package(
    name = "ghc",
    binaries = [
        "ghc-pkg",
    ],
    binary = "ghc",
    flake = "nix", # path to flake directory
)
```

* calls `nix build`
* output is a symlink to the nix store
* also returns `RunInfo` for the binaries

---

## Relatedly...

For Bazel: https://github.com/tweag/rules_nixpkgs

For buck2: https://github.com/thoughtpolice/buck2-nix

Note: this project provides an alternative prelude which is incompatible.
(does not provide Haskell rules) Also, does not support flakes. (but took
inspiration for our rules)

---

## Buck2 project organization

* root - the project
* prelude   - the standard ruleset provided by buck2
* toolchains - cell which configures the required toolchains:

    - `@toolchains//:cxx` => C/C++ toolchain
    - `@toolchains//:haskell` => Haskell Toolchain
    - ...

---

## Toolchains with Nix

* add a flake at `toolchains/nix`
* wire up ghc package from toolchains nix flake and expose it as `@toolchains//:haskell` target
* repeat for other toolchains...

---

## Haskell Integration

`ghcWithPackages` + nix shell

* provides a complete ghc environment, with ghc package DB
* "monolithic"
* out of control for buck2

---

## Haskell Integration

instead:

* let Buck2 control building nix packages individually
* fine-grained management of haskell dependencies per target

---

## Introducing dynamic actions

* build dependency tree is static
* *but* the action graph can contain dynamic actions

---

<!-- ## Example project -->

<!-- flake.nix => buck2 -->

<!-- external cell buck2.nix -->

<!-- toolchains/nix flake ❄ -->

## Haskell Integration (continued)

Ensure every used Haskell package has a valid ghc package DB

<div style="position: relative">
  <img src="assets/overlay_stamp_white.webp" style="width: 15%; rotate: 30deg; position: absolute; right: 1em; z-index: 100; top: -1em "/>

```nix
# applied to all isHaskellLibrary packages
postInstall = (drv.postInstall or "") + ''
    ghc-pkg recache --package-db $packageConfDir
'';
```
</div>

---

## Haskell Integration (continued)

```nix
# flake.nix
let
  haskellLibraries =
    let
      packages = builtins.map (n: hsPkgs."${n}")
        toolchainLibraries;
      isHaskellLibrary = p: p ? isHaskellLibrary;
    in
      builtins.filter isHaskellLibrary
        (lib.closePropagation packages);
in {
packages = {
   haskellPackages = builtins.listToAttrs
     (builtins.map (p: { "name" = p.pname; "value" = p; })
       haskellLibraries);
   ...
}
```

---

## Haskell Integration (continued)

```console
$ nix eval --json \
  --apply "hs: builtins.map (h: h.drvPath) (builtins.attrValues hs)" \
  --no-update-lock-file \
  --no-use-registries \
  path:toolchains/nix#haskellPackages
```

Notes: get a list of all haskell library derivations

---

## Haskell Integration (continued)

```console
$ nix derivation show --stdin > drv.json <<EOF
/nix/store/cjvjkl61g1gi9jbqhp2q3fas8gid1cn4-HUnit-1.6.2.0.drv^*
/nix/store/px5cl37cj80q0x3gxnw295pcmdp1yyqn-OneTuple-0.4.2.drv^*
/nix/store/m9iqpsk111x8ajm0kwjmr9cm5v97dimp-QuickCheck-2.14.3.drv^*
...
EOF
```

outputs `drv.json`, which is read in a dynamic action.

---

## Demo

Notes: cells, toolchains, targets, dependency graph

---

## Next

* publish buck2.nix ruleset
* upstream changes to buck2's prelude
* remote execution

---

### Thank you

<div style="justify-content: center; display: flex; align-items: center">
<img alt="buck2" src="assets/buck2_logo.svg" style="max-height: 3em; vertical-align: middle; margin-right: .5em"></img>
<span style="padding-left: 1em; padding-right: 1em">➕</span>
<img alt="nix" src="assets/nix-snowflake-rainbow.svg" style="height: 3em; vertical-align: middle; margin-left: .5em">
<span style="padding-left: 1em; padding-right: 1em">🟰</span>
<svg xmlns="http://www.w3.org/2000/svg" style="color: yellow; max-height: 3em" viewBox="0 0 512 512"><!--!Font Awesome Free 6.6.0 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license/free Copyright 2024 Fonticons, Inc.--><path fill="currentColor" d="M464 256A208 208 0 1 0 48 256a208 208 0 1 0 416 0zM0 256a256 256 0 1 1 512 0A256 256 0 1 1 0 256zm177.6 62.1C192.8 334.5 218.8 352 256 352s63.2-17.5 78.4-33.9c9-9.7 24.2-10.4 33.9-1.4s10.4 24.2 1.4 33.9c-22 23.8-60 49.4-113.6 49.4s-91.7-25.5-113.6-49.4c-9-9.7-8.4-24.9 1.4-33.9s24.9-8.4 33.9 1.4zM144.4 208a32 32 0 1 1 64 0 32 32 0 1 1 -64 0zm192-32a32 32 0 1 1 0 64 32 32 0 1 1 0-64z"/></svg><span style="padding-left: 1em; padding-right: 1em">➕</span><svg style="color: red; max-height: 3em" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512"><!--!Font Awesome Free 6.6.0 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license/free Copyright 2024 Fonticons, Inc.--><path fill="currentColor" d="M320 96L192 96 144.6 24.9C137.5 14.2 145.1 0 157.9 0L354.1 0c12.8 0 20.4 14.2 13.3 24.9L320 96zM192 128l128 0c3.8 2.5 8.1 5.3 13 8.4C389.7 172.7 512 250.9 512 416c0 53-43 96-96 96L96 512c-53 0-96-43-96-96C0 250.9 122.3 172.7 179 136.4c0 0 0 0 0 0s0 0 0 0c4.8-3.1 9.2-5.9 13-8.4zm84 88c0-11-9-20-20-20s-20 9-20 20l0 14c-7.6 1.7-15.2 4.4-22.2 8.5c-13.9 8.3-25.9 22.8-25.8 43.9c.1 20.3 12 33.1 24.7 40.7c11 6.6 24.7 10.8 35.6 14l1.7 .5c12.6 3.8 21.8 6.8 28 10.7c5.1 3.2 5.8 5.4 5.9 8.2c.1 5-1.8 8-5.9 10.5c-5 3.1-12.9 5-21.4 4.7c-11.1-.4-21.5-3.9-35.1-8.5c-2.3-.8-4.7-1.6-7.2-2.4c-10.5-3.5-21.8 2.2-25.3 12.6s2.2 21.8 12.6 25.3c1.9 .6 4 1.3 6.1 2.1c0 0 0 0 0 0s0 0 0 0c8.3 2.9 17.9 6.2 28.2 8.4l0 14.6c0 11 9 20 20 20s20-9 20-20l0-13.8c8-1.7 16-4.5 23.2-9c14.3-8.9 25.1-24.1 24.8-45c-.3-20.3-11.7-33.4-24.6-41.6c-11.5-7.2-25.9-11.6-37.1-15c0 0 0 0 0 0l-.7-.2c-12.8-3.9-21.9-6.7-28.3-10.5c-5.2-3.1-5.3-4.9-5.3-6.7c0-3.7 1.4-6.5 6.2-9.3c5.4-3.2 13.6-5.1 21.5-5c9.6 .1 20.2 2.2 31.2 5.2c10.7 2.8 21.6-3.5 24.5-14.2s-3.5-21.6-14.2-24.5c-6.5-1.7-13.7-3.4-21.1-4.7l0-13.9z"/></svg>
</div>

<div style="position: relative">
<div style="transform: rotate(-28deg); position: absolute; top: 2em; right: 1em; font-size: larger">Questions?</div>
</div>

Notes: convinced that combining buck2 + nix is a match
