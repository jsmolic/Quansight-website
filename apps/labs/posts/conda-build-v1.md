---
title: 'Bringing v1 recipes to conda-build'
published: September 10, 2026
authors: [jakov-smolic]
description: "conda-build can now build v1 recipes, bringing a standardized, schema-validated recipe format to a familiar UX."
category: [Packaging]
featuredImage:
  src: 
  alt: ''
hero:
  imageSrc: 
  imageAlt: ''
---

Building conda packages can be slow. Anyone who has spent time creating or maintaining packages
for conda or conda-forge has encountered builds that can take a considerable amount of time.

Recent work on `conda-build` aims to improve that experience: `conda-build` now supports v1 recipes,
with builds powered by `rattler-build`.

## Motivation

`conda-build` has historically been the standard tool for building conda packages.
In 2025, conda-forge [adopted the v1 recipe format](https://conda-forge.org/blog/2025/02/27/conda-forge-v1-recipe-support/), which brought several improvements over the original one.

Initially, `conda-build` didn't support the v1 recipe format, only v0. In contrast to v1, the traditional v0 `meta.yaml` format supports arbitrary Jinja expressions anywhere in the file, which can introduce complexity, as parsing and processing these recipes often requires special handling.

The v1 recipe format uses pure YAML together with a standardized schema that only allows a subset of Jinja expressions in specific places only. This simplifies parsing and handling.

This enables features such as schema validation and editor autocomplete, while also making recipes easier to read, maintain, and update over time.

Additionally, this schema is approved by the conda steering council.


`rattler-build` is the primary tool for building v1 recipes. It is built on [Rattler](https://github.com/conda/rattler), a set of Rust crates that implement common conda package management functionality. It features faster solver times, IO and recipe parsing. Altogether, rattler-build feels faster than conda-build.


## Bringing rattler-build to conda-build

While trying to vring v1 support to conda-build, we considered different options, but ultimately settled for integrating the rattler-build primitives directly.

In order to bring v1 format recipes support to `conda-build`, we leveraged the python bindings of the `rattler-build` which were created by the [Prefix.dev](https://prefix.dev/) team in partnership with Anaconda.

The implementation aims to preserve the existing `conda-build` UX. The integration is
done entirely within the `conda-build` subcommands and is built into the existing command-line interface.

When `conda-build` is invoked, it determines the recipe format being used and dispatches
the build to the appropriate backend. 

<!-- Build configuration works in much the same way as it does for v0 recipes. Many existing
command-line options and configuration settings are translated into their rattler-build
equivalents and passed through py-rattler-build. -->

`py-rattler-build` provides Python APIs for parsing, rendering, building, and testing
packages. Integrating those APIs into `conda-build` required translating `conda-build`
configuration settings into their `rattler-build` equivalents and passing them through
`py-rattler-build`. 


```mermaid
flowchart LR
    A["conda build path/to/recipe"] --> B{"Recipe format?"}
    B -- "meta.yaml (v0)" --> C["conda-build backend"]
    B -- "recipe.yaml (v1)" --> D["py-rattler-build"]
    C --> E[".conda build output"]
    D --> E
```

<!-- To accomplish this, the integration relies on conda's existing configuration, which
combines settings from different sources such as `.condarc` files, environment variables
and command-line arguments. -->

The compatibility layer also handles other important aspects of the build such as:

- Surfacing errors through the existing `conda-build` interface
- Routing build output through conda's logging system
- Handling variant and channel configuration
- Gathering and reporting build results
- Reusing the existing package upload workflow

Support also extends beyond package builds, as v1 recipes now work with the
`conda render` and `conda debug` commands as well.

`py-rattler-build` provides a Python API for `rattler-build`, following the standard build workflow:

1. **Load the recipe.** `Stage0Recipe.from_file()` parses a `recipe.yaml` into a recipe object.
2. **Render it into variants.** `recipe.render()` takes a `VariantConfig`, which defines the values to build for (for example several Python or NumPy versions). It returns one `RenderedVariant` for each combination.
3. **Build each variant.** `run_build()` on a rendered variant builds it and returns a `BuildResult` with the package name, version, build time and the packages produced.
4. **Test the package.** `Package.from_file()` loads a built `.conda` file, and `run_tests()` runs the tests shipped with it. This also works on the `.conda` archives.


```mermaid
flowchart TD
    Start([recipe.yaml]) --> Load{"Recipe source"}
    Load["Stage0Recipe.from_file()"] --> Parsed["Stage0Recipe<br/>parsed recipe"]
    Parsed --> Render["recipe.render(VariantConfig)"]
    Render --> Variants["list of RenderedVariant<br/>one per variant combination"]
    Variants --> Loop[/"for each variant"/]
    Loop --> RunBuild["variant.run_build()"]
    RunBuild --> Result["BuildResult<br/>name, version, build_time, packages"]

    Result --> TestQ{"Test the package?"}
    TestQ -->|"No"| Done([Done])
    TestQ -->|"Yes"| LoadPkg["Package.from_file(.conda file)"]

    Existing[("Existing .conda package")] --> LoadPkg
    LoadPkg --> RunTests["pkg.run_tests()"]
    RunTests --> TestResults["Test results"]
    TestResults --> Done
```

## Configuration

conda-build builds its configuration from conda's context (`.condarc` files, environment variables and command-line arguments) and its own `Config` object. For v1 recipes, the compatibility layer in `conda_build/_rattler_build/compat.py` reads that same configuration and hands the relevant parts to `py-rattler-build`. The `rattler-build` side doesn't take a single configuration object. Different settings go to different places:

```mermaid
flowchart LR
    subgraph cb["conda-build"]
        V["Variant files<br/>conda_build_config.yaml, variants.yaml, -m files"]
        C["conda context and CLI<br/>channels, channel priority, build options"]
        P["Platforms<br/>CONDA_SUBDIR, target_platform"]
    end

    V -->|"from_conda_build_config() or from_file()<br/>then merge()"| VC["VariantConfig"]
    P --> RC["RenderConfig<br/>PlatformConfig, extra_context"]
    C --> TC["ToolConfiguration<br/>channel_priority"]
    C --> BA["run_build() arguments<br/>channels, output_dir, package_format, ..."]

    VC --> R["recipe.render()"]
    RC --> R
    R --> B["variant.run_build()"]
    TC --> B
    BA --> B
```

The `VariantConfig` is built only from variant files. Everything else goes into the render configuration, the tool configuration, or the build call itself.

### Variant files to `VariantConfig`

conda-build finds variant files with the same `find_config_files()` function it uses for v0 recipes, so discovery works the same way for both formats. For a single recipe, this includes a `conda_build_config.yaml` or `variants.yaml` next to the recipe, as well as files passed with `-m` or `--variant-config-files`. When you build several recipes at once, no recipe may have its own config file next to it. conda-build raises an error and asks you to build those recipes separately.

It then starts from an empty `VariantConfig()` and merges the files one at a time, in the order they are stacked. Each file is read in one of two ways, chosen by its name:

```python
if Path(variant).name == "conda_build_config.yaml":
    # legacy v0 format, with selector comments
    jinja_config = JinjaConfig(
        platform=PlatformConfig(
            build_platform=config.build_subdir,
            target_platform=config.host_subdir,
        )
    )
    variant_config = variant_config.merge(
        VariantConfig.from_conda_build_config(variant, jinja_config)
    )
else:
    # v1 format, plain YAML
    variant_config = variant_config.merge(VariantConfig.from_file(variant))
```

- **`conda_build_config.yaml` (v0 format):** `rattler-build` itself parses this file with `VariantConfig.from_conda_build_config()`. Selectors such as `# [linux]` need to know the platform, so conda-build supplies the build and host platforms from its own configuration.
- **Any other file name (v1 format):** the file is read as plain YAML with `VariantConfig.from_file()`. This is the case for `variants.yaml`.

Because the format is decided by file name, a v0-style file with a different name is read as v1 and may fail to load ([#6170](https://github.com/conda/conda-build/issues/6170)).

If a Jinja variable in the recipe is undefined during rendering, the error message points to missing or incorrect variant files as the likely cause.

### Platforms

Platforms are not part of the `VariantConfig`, but conda-build reads some of them from it. They are collected into a `PlatformConfig` that is passed to `RenderConfig`:

- **Build platform:** taken from conda-build's `build_subdir`, which can be overridden with the `CONDA_SUBDIR` environment variable.
- **Host and target platform:** both are read from the `target_platform` variant key, falling back to conda-build's host and target subdirectories if the key isn't set.
- **`noarch_build_platform`:** also read from the variant configuration and passed to the tool configuration.

These values must be single, not a list. If a variant file gives several values for one of them, conda-build raises an error.

### Channels

Channels come from conda's context. With `--override-channels`, only the channels passed with `-c` are used, and conda-build raises an error if there are none. conda-build expands multichannels such as `defaults` and `local`, as well as user-defined ones, into their subchannels, and `rattler-build` doesn't yet support multichannels natively. Because of that, channel priority across those subchannels may not be respected (there is a `TODO` about this in the source).

Channel priority is simplified to what `rattler-build` supports: it is `strict` if conda's channel priority is `strict`, and `disabled` otherwise. The same channels are used for the build, for `conda debug` and for running tests. For tests, conda-build also adds the output folder as a local `file://` channel, so tests can find the package that was just built.

### Other settings

The remaining command-line options are translated as follows:

| conda-build | Passed to `py-rattler-build` as |
| --- | --- |
| `--output-folder` (default: the build root) | `output_dir` |
| `--set-build-id` | `no_build_id` (inverted) |
| `--include-recipe` | `no_include_recipe` (inverted) |
| `--conda-pkg-format` | `package_format` (`conda` for the v2 format, otherwise `.tar.bz2`) |
| `--extra-meta` | `extra_context` of the render configuration |
| `--quiet` | whether `py-rattler-build` log messages are shown |

Some behavior stays in conda-build. Running tests is handled by conda-build rather than by `rattler-build`, so `--no-test` and `--build-only` simply skip that step. Uploads to anaconda.org use conda-build's existing upload logic and are skipped for `--build-only` and `--post`.

### Unsupported settings

The v1 path supports only a subset of conda-build's options. A validation step compares the command-line arguments and the `conda_build` section of `.condarc` against an allow-list for each command and raises an error that lists any unsupported ones. For example, `conda render` accepts only the recipe, the variant config files and the channel options.

### Differences to be aware of

- **Variant keys in build scripts:** conda-build scans v0 build scripts for variant keys they use. `rattler-build` does not. If a script needs a variant value that nothing else in the recipe uses, pass it in explicitly:

```yaml
  build:
    script:
      content: echo $MY_VARIANT
      env:
        MY_VARIANT: ${{ my_variant }}
```

- **Variant key names:** variant keys are matched against requirement names, and names containing `.` or `-` have known edge cases ([#6200](https://github.com/conda/conda-build/issues/6200)).


## Converting recipes to the v1 format


If you already maintain packages using the v0 `meta.yaml` format, moving to v1 does not require rewriting the recipe from scratch. The `conda-recipe-manager` project can convert a `meta.yaml` recipe into a v1 `recipe.yaml` with:

```shell
crm convert meta.yaml > recipe.yaml
```

Here is a small before and after showing the differences most often encountered during the conversion:

**v0 (`meta.yaml`)**

```yaml
{% set version = "1.2.3" %}

package:
  name: example
  version: {{ version }}

source:
  url: https://example.org/example-{{ version }}.tar.gz
  sha256: 0123456789abcdef...

build:
  number: 0
  skip: true  # [win]

requirements:
  host:
    - python
    - pip
  run:
    - python
    - numpy  # [linux]
```

**v1 (`recipe.yaml`)**

```yaml
schema_version: 1

context:
  version: "1.2.3"

package:
  name: example
  version: ${{ version }}

source:
  url: https://example.org/example-${{ version }}.tar.gz
  sha256: 0123456789abcdef...

build:
  number: 0
  skip: win

requirements:
  host:
    - python
    - pip
  run:
    - python
    - if: linux
      then: numpy
```

The two formats differ in more than syntax, a converted recipe should be treated as a starting point and be reviewd before relying on it. Selectors and conditional dependencies need extra attention, along with anything that used Jinja logic that v1 no longer allows.

Two resources are especially helpful for this review:

- The [rattler-build migration guide](https://rattler-build.prefix.dev/dev/converting_from_conda_build/) describes the differences between the formats and how to migrate recipes. Its section on [converting selectors](https://rattler-build.prefix.dev/dev/converting_from_conda_build/#converting-selectors) is a good place to start.
- The [conda-forge knowledge base](https://conda-forge.org/docs/maintainer/knowledge_base/) shows v0 and v1 versions of common patterns side by side, such as compiler, OpenMP and BLAS dependencies.

## How to try it?

If you already use `conda-build`, getting started with v1 recipes should
feel familiar.  They can be built using the same command:

```shell
conda build path/to/recipe
```

To convert existing meta.yaml recipes to the new format, you can use the
[conda-recipe-manager](https://github.com/conda/conda-recipe-manager) tool.

For more details on using v1 recipes with `conda-build`, see the
[conda-build v1 recipes documentation](https://docs.conda.io/projects/conda-build/en/latest/user-guide/v1-recipes.html).

## Conclusion

The conda packaging ecosystem is gradually moving toward the v1 recipe standard.
By supporting v1 recipes directly in `conda-build`, package maintainers now have
a more flexible migration path: recipes can be updated to the new format independently
of the build tooling they use.

Maintainers can continue using `conda-build`, start adopting the v1 format, and
benefit from the performance improvements provided by `rattler-build`.

For package maintainers who have spent too much time waiting for builds to finish,
that can be a meaningful improvement.