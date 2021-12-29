# `opam-nix`

Turn opam-based OCaml projects into Nix derivations.


## TL;DR

### Templates

- `nix flake new my-package -t github:tweag/opam-nix#simple`

### Examples

- [Building a package from opam-repository](./examples/0install.nix)
- [Building a package from a custom github repository](./examples/opam2json.nix)
- [Building a static version of a package using compiler from nixpkgs](./examples/opam-ed.nix)
- [Building a static version of a package using the compiler from opam](./examples/opam2json-static.nix)
- [Building a GUI package](./examples/frama-c.nix)

All examples are checks and packages, so you can do e.g. `nix build
github:tweag/opam-nix#opam-ed` to try them out individually, or `nix
flake check github:tweag/opam-nix` to build them all.

## Public-facing API

Public-facing API is presented in the `lib` output of the flake. It is
mapped over the platforms, e.g. `lib.x86_64-linux` provides the
functions usable on x86_64-linux. Functions documented below reside in
those per-platform package sets, so if you want to use
e.g. `makeOpamRepo`, you'll have to use
`opam-nix.lib.x86_64-linux.makeOpamRepo`. All examples assume that the
relevant per-platform `lib` is in scope.

You can instantiate `opam.nix` yourself, by passing at least some
`pkgs` (containing `opam2json`), and optionally `opam-repository` for
use as the default repository (if you don't pass `opam-repository`,
`repos` argument becomes required everywhere).

### Package

`Derivation`

An `opam-nix` "Package" is just a nixpkgs-based derivation which has
some additional properties.

Its output contains an empty file `nix-support/is-opam-nix-package`,
and also it has a `nix-support/setup-hook` setting some internal
variables, `OCAMLPATH`, `CAML_LD_LIBRARY_PATH`, and other variables
exported by packages using `variables` in `<pkgname>.config` or
`setenv` in `<pkgname>.opam`.

The behaviour of the build script can be controlled using build-time
environment variables. If you want to set an opam environment variable
(be it for substitution or a package filter), you can do so by passing
it to `overrideAttrs` of the package with a special transformation
applied to the variable's name: replace `-` and `+` in the name with
underscores (`_`), replace all `:` (separators between package names
and their corresponding variables) with two underscores (`__`), and
prepend `opam__`. For example, if you want to get a package `cmdliner`
with `conf-g++:installed` set to `true`, do `cmdliner.overrideAttrs
(_: { opam__conf_g____installed = "true"; })`

If you wish to change the build in some other arbitrary way, do so as
you would with any nixpkgs package. You can override phases, but note
that `configurePhase` is special and should not be overriden (unless
you read `builder.nix` and understand what it is doing).

### Repository

`Path or Derivation`

A repository is understood in the same way as for opam itself. It is a
directory (or a derivation producing a directory) which contains at
least `repo` and `packages/`. Directories in `packages` must be
package names, directories in those must be of the format
`name.version`, and each such subdirectory must have at least an
`opam` file.

If a repository is a derivation, it may contain `passthru.sourceMap`,
which maps package names to their corresponding sources.

### `queryToScope`

```
{ repos = ?[Repository]
, pkgs = ?Nixpkgs
, overlays = ?[Overlay]
, env = ?{ ${var_name} = value : String; ... } }
→ Query
→ Scope
```

Turn a `Query : { ${package_name} = package_version : String or null;
... }` (an attribute set with attribute names being package names
which are desired to be in the scope and attribute palues being
versions), into a `Scope : { overrideScope' = (Scope → Scope → Scope)
→ Scope; ${package_name} = package : Package; ... }` (a nixpkgs
scope/package set). Note that the scope is self-referential,
i.e. packages in the set refer to other packages from that same
set. Also note that there can only be one version of each package in
the set, due to constraints in OCaml's way of
linking. `overrideScope'` can be used to apply overlays to the scope.

Special value of `null` can be passed as a version in the `Query` to
let opam figure out the latest possible version for the package.

The first argument allows providing custom repositories & top-level
nixpkgs, adding overlays and passing an environment to the resolver.

#### `repos`, `env` & version resolution

Versions are resolved using upstream opam. The passed repositories
(`repos`, containing `opam-repository` by default) are merged and then
`opam admin list --resolve` is called on the resulting
directory. Package versions from earlier repositories take precedence
over package versions from later repositories. `env` allows to pass
additional "environment" to `opam admin list`, affecting its version
resolution decisions. See [`man
opam-admin`](https://opam.ocaml.org/doc/man/opam-admin.html) for
further information about the environment.

When a repository in `repos` is a derivation and contains
`passthru.sourceMap`, sources for packages taken from that repository
are taken from that source map.

#### `pkgs` & `overlays`

By default, `pkgs` match the `pkgs` argument to `opam.nix`, which, in
turn, is the `nixpkgs` input of the flake. `overlays` default to
`defaultOverlay` and `staticOverlay` in case the passed nixpkgs appear
to be targeting static building.

#### Examples

Build a package from `opam-repository`, using all sane defaults:

```nix
(queryToScope { } { opam-ed = null; }).opam-ed
```

Build a specific version of the package, overriding some dependencies:

```nix
let
  scope = queryToScope { } { opam-ed = "0.3"; };
  overlay = self: super: {
    opam-file-format = super.opam-file-format.overrideAttrs
      (oa: { opam__ocaml__native = "true"; });
  };
in (scope.overrideScope' overlay).opam-ed
```

Pass static nixpkgs (to get statically linked libraries and
executables) and a local opam-repository:

```nix
let
  my-nixpkgs = import ../nixpkgs { };
  my-opam-repository = import ../opam-repository { };
  scope = queryToScope {
    pkgs = my-nixpkgs.pkgsStatic;
    repos = [ my-opam-repository ];
  } { opam-ed = null; };

in scope.opam-ed
```

### `makeOpamRepo`

`Path → Derivation`

Traverse a directory, looking for `opam` files and collecting them
into a repository in a format understood by `opam`. The resulting
derivation will also provide `passthru.sourceMap`, which is a map from
package names to package sources taken from the original `Path`.

Packages for which the version can not be inferred get `local` as
their version.

Note that all `opam` files in this directory will be evaluated using
`importOpam`, to get their corresponding package names and versions.

#### Examples

Build a package from a local directory, which depends on packages from opam-repository:

```nix
let
  # You can reuse opam-repository from inputs of opam-nix's flake
  repos = [ (makeOpamRepo ./.) opam-repository ];
  scope = queryToScope { inherit repos; } { my-package = null; };
in scope.my-package
```

### `listRepo`

`Repository → {${package_name} → [version : String]}`

Produce a mapping from package names to lists of versions (sorted
older-to-newer) for an opam repository.

### `opamImport`

`{ repos?, pkgs? } → Path → Scope`

Import an opam switch, similarly to `opam import`, and provide a
package combining all the packages installed in that switch. `repos`,
`pkgs` and `Scope` are understood identically to `queryToScope`,
except no version resolution is performed.

### `opam2nix`

`{ src = Path; opamFile = ?Path; name = ?String; version = ?String; } → Dependencies → Package`

Produce a callPackage-able `Package` from an opam file. This should be
called using `callPackage` from a `Scope`. Note that you are
responsible to ensure that the package versions in `Scope` are
consistent with package versions required by the package. May be
useful in conjunction with `opamImport`.

#### Examples

```nix
let
  scope = opamImport ./src/opam.export;
  pkg = opam2nix ./my-package.opam;
in scope.callPackage pkg {}
```

### `defaultOverlay`, `staticOverlay`

`Scope → Scope → Scope`

Overlays for the `Scope`'s. Contain enough to build the
examples. Apply with `overrideScope'`.

### `fromOpam` / `importOpam`

`fromOpam : String -> {...}`

`importOpam : Path -> {...}`

Generate a nix attribute set from the opam file. This is just a Nix
representation of the JSON produced by `opam2json`.

### Lower-level functions

`joinRepos : [Repository] → Repository`

`opamList : Repository → Env → Query → [String]`

`opamListToQuery : [String] → Query`

`queryToDefs : [Repository] → Query → Defs`

`defsToScope : Nixpkgs → Defs → Scope`

`applyOverlays : [Overlay] -> Scope -> Scope`

`opamList` resolves package versions using the repo (first argument)
and environment (second argument). Note that it accepts only one
repo. If you want to pass multiple repositories, merge them together
yourself with `joinRepos`. The result of `opamList` is a list of
strings, each containing a package name and a package version. Use
`opamListToQuery` to turn this list into a "`Query`" (but with all the
versions specified).

`queryToDefs` takes a query (with all the version specified,
e.g. produced by `opamListToQuery` or by reading the `installed`
section of `opam.export` file) and produces an attribute set of
package definitions (using `importOpam`).

`defsToScope` takes a nixpkgs instantiataion and an attribute set of
definitions (as produced by `queryToDefs`) and produces a `Scope`.

`applyOverlays` applies a list of overlays to a scope.

#### `Defs` (set of package definitions)

The attribute set of package definitions has package names as
attribute names, and package definitions as nix attrsets. These are
basically the nix representation of
[opam2json](https://github.com/tweag/opam2json/) output. The format of
opam files is described here:
https://opam.ocaml.org/doc/Manual.html#opam .

### Examples

Build a local package, using an exported opam switch and some
"vendored" dependencies, and applying some local overlay on top.

```nix
let
  pkgs = import <nixpkgs> { };

  repos = [
    (opam-nix.makeOpamRepo ./src) # "Pin" vendored packages
    inputs.opam-repository
  ];

  export =
    opam-nix.opamListToQuery (opam-nix.fromOPAM ./src/opam.export).installed;

  vendored-packages = {
    "my-vendored-package" = "local";
    "my-other-vendored-package" = "v1.2.3";
    "my-package" = "local"; # Note: you can't use null here!
  };

  myOverlay = import ./overlay.nix;

  scope = applyOverlays [ defaultOverlay myOverlay ]
    (defsToScope pkgs (queryToDefs repos (export // vendored-packages)));
in scope.my-package
```

## Using without flakes

This is a flake. It can be used without flakes enabled, using the
provided [default.nix](default.nix) file. To use it, fetch the
repository somehow (e.g. using `fetchTarball
https://github.com/tweag/opam-nix/archive/master.tar.gz`), import it
and then use the `lib.${builtins.currentPlatform}` attribute.

