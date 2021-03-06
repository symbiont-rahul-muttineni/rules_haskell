# Bazel compatible `ghc-paths`

This package provides a Bazel compatible alternative to the [`ghc-paths`][ghc-paths-hackage] package.

## Usage

If you use `stack_snapshot` then you can use the `vendored_packages` attribute to replace the Hackage provided `ghc-paths` package by this Bazel compatible version:

``` python
stack_snapshot(
    name = "stackage",
    ...
    vendored_packages = {
        "ghc-paths": "@rules_haskell//tools/ghc-paths",
    },
```

If your target depends on files of the GHC installation at runtime, then you need to add the corresponding targets to your target's `data` attribute:

``` python
    data = [
        # If you use `GHC.Paths.ghc` or `GHC.Paths.ghc_pkg`
        "@rules_haskell//tools/ghc-paths:bin",
        # If you use `GHC.Paths.docdir`
        "@rules_haskell//tools/ghc-paths:docdir",
        # If you use `GHC.Paths.libdir`
        "@rules_haskell//tools/ghc-paths:libdir",
    ],
```

You can use the `add_data` rule to extend the runfiles of an existing target. This is useful for targets that you don't control directly. E.g. an executable target generated by `stack_snapshot`. E.g.

``` python
load("//tools/ghc-paths:defs.bzl", "add_data")

add_data(
    name = "ghcide",
    data = ["//tools/ghc-paths:libdir"],
    executable = "@ghcide-exe//ghcide",
)
```

## Rationale

Some packages, such as `doctest`, `ghcide`, or `proto-lens-protoc`, make use of the GHC API provided by the `ghc` package. The GHC API needs access to files included in the GHC installation to initialize a GHC session, in particular the library directory, i.e. `ghc --print-libdir`.

[`ghc-paths`][ghc-paths-hackage] is a convenient package that exposes the relevant filepaths as Haskell constants. It does so by querying the GHC installation at build time through a custom Cabal setup script and hard-coding the outcome into its sources. This assumes that paths to the GHC installation at build time will still be valid at runtime. This assumption is not generally valid.

In particular, when using a Bazel installed GHC bindist, i.e. `haskell_register_ghc_bindists`, Bazel will expose the GHC installation to build actions within the sandbox working directory. The path to this sandbox working directory will vary across build actions and will differ from the path at which it is available at runtime, e.g. `bazel run` or `bazel test`.

## Implementation

This Bazel compatible version of `ghc-paths` use two mechanisms to obtain the paths to the GHC installation.

1. It queries the following environment variables:
  - `RULES_HASKELL_GHC_PATH`
  - `RULES_HASKELL_GHC_PKG_PATH`
  - `RULES_HASKELL_LIBDIR_PATH`
  - `RULES_HASKELL_DOCDIR_PATH`
2. If the above environment variables are empty it queries the Bazel runfiles tree using `@rules_haskell//tools/runfiles`.

Note that both these operations are `IO` actions. However, `ghc-paths` exposes these paths as pure `String` constants. This Bazel compatible version of `ghc-paths` uses `unsafePerformIO` to provide the same API. The values are memoized at the first lookup, such that they do not change if the environment changes later on.

## Caveats

The implementation as described above means that the result is sensitive to the environment and has the same limitations as `@rules_haskell//tools/runfiles`. In particular, the runfiles location can depend on the value of `argv[0]` which can be modified by `System.Environment.withProgName` and `System.Environment.withArgs`, see [GHC #18418][ghc-18418].

[ghc-18418]: https://gitlab.haskell.org/ghc/ghc/-/issues/18418
[ghc-paths-hackage]: https://hackage.haskell.org/package/ghc-paths
