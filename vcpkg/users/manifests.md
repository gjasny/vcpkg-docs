---
title: Manifest mode
description: Use vcpkg in Manifest mode to configure libraries on a per project basis.
ms.date: 11/30/2022
---
# Manifest mode

Check out the [manifest cmake example](../examples/manifest-mode-cmake.md) for an example project using CMake and manifest mode.

## Simple Example Manifest

```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg/master/scripts/vcpkg.schema.json",
  "name": "my-application",
  "version": "0.15.2",
  "dependencies": [
    "boost-system",
    {
      "name": "cpprestsdk",
      "default-features": false
    },
    "libxml2",
    "yajl"
  ]
}
```

## Manifest syntax reference

A manifest is a JSON-formatted file named `vcpkg.json` which lies at the root of your project.
It contains all the information a person needs to know to get dependencies for your project,
as well as all the metadata about your project that a person who depends on you might be interested in.

Manifests follow strict JSON: they can't contain C++-style comments (`//`) nor trailing commas. However
you can use field names that start with `$` to write your comments in any object that has a well-defined set of keys.
These comment fields are not allowed in any objects which permit user-defined keys (such as `"features"`).

The latest JSON Schema is available at [vcpkg.schema.json](https://raw.githubusercontent.com/microsoft/vcpkg/master/scripts/vcpkg.schema.json). IDEs with JSON Schema support such as Visual Studio and Visual Studio Code can use this file to provide IntelliSense and syntax checking. For most IDEs, you should set `"$schema"` in your `vcpkg.json` to this URL (like the above example).

Each manifest contains a top level object with the following fields:

### <a id="name"></a> `"name"`

This is the name of your project! It must be formatted in a way that vcpkg understands - in other words,
it must be lowercase alphabetic characters, digits, and hyphens, and it must not start nor end with a hyphen.
For example, `Boost.Asio` might be given the name `boost-asio`.

### Version fields

There are four version field options, depending on how the port orders its
releases.

- [`"version"`](versioning.md#version) - Generic, dot-separated numeric
  sequence.
- [`"version-semver"`](versioning.md#version-semver) - [Semantic Version
  2.0.0](https://semver.org/#semantic-versioning-specification-semver)
- [`"version-date"`](versioning.md#version-date) - Used for packages which do
  not have numeric releases (for example, Live-at-HEAD). Matches `YYYY-MM-DD`
  with optional trailing dot-separated numeric sequence.
- [`"version-string"`](versioning.md#version-string) - Used for packages that
  don't have orderable versions. This should be rarely used, however all ports
  created before the other version fields were introduced use this scheme.

Additionally, the optional `"port-version"` field is used to indicate revisions
to the port with the same upstream source version. For pure consumers, this
field should not be used.

See [versioning](versioning.md#version-schemes) for more details.

### <a id="description"></a> `"description"`

This is where you describe your project. Give it a good description to help in searching for it!
This can be a single string, or it can be an array of strings;
in the latter case, the first string is treated as a summary,
while the remaining strings are treated as the full description.

### <a id="builtin-baseline"></a> `"builtin-baseline"`

This field indicates the commit of vcpkg which provides global minimum version
information for your manifest.

It is required for top-level manifest files using versioning without a specified [`"default-registry"`](registries.md#configuration-default-registry). It has the same semantic as defining your default registry to be:

```json
{
  "default-registry": {
    "kind": "builtin",
    "baseline": "<value>"
  }
}
```

See [versioning](versioning.md#baselines) for more semantic details.

### <a id="dependencies"></a> `"dependencies"`

This field lists all the dependencies you'll need to build your library (as well as any your dependents might need,
if they were to use you). It's an array of strings and objects:

- A string dependency (e.g., `"dependencies": [ "zlib" ]`) is the simplest way one can depend on a library;
  it means you don't depend on a single version, and don't need to write down any more information.
- On the other hand, an object dependency (e.g., `"dependencies": [ { "name": "zlib" } ]`)
  allows you to add that extra information.

#### Example

```json
"dependencies": [
  {
    "name": "arrow",
    "default-features": false,
    "features": [ "json" ]
  },
  "boost-asio",
  "openssl",
  {
    "name": "picosha2",
    "platform": "!windows"
  }
]
```

#### <a name="dependencies-name"></a> `"name"` field

The name of the dependency. This follows the same restrictions as the [`"name"`](#name) property for a project.

#### <a name="dependencies-default-features"></a> <a name="dependencies-features"></a> `"features"` and `"default-features"` fields

`"features"` is an array of feature names which tell you the set of features that the
dependencies need to have at a minimum,
while `"default-features"` is a boolean that tells vcpkg whether or not to
install the features the package author thinks should be "most common for most people to use".

For example, `ffmpeg` is a library which supports many, many audio and video codecs;
however, for your specific project, you may only need mp3 encoding.
Then, you might just ask for:

```json
{
  "name": "ffmpeg",
  "default-features": false,
  "features": [ "mp3lame" ]
}
```

#### <a name="host"></a> `"host"` field

A boolean indicating that the dependency must be built for the [host triplet](host-dependencies.md) instead of the current port's triplet. Defaults to `false`.

Any dependency that provides tools or scripts which should be "executed" during a build (such as buildsystems, code generators, or helpers) should be marked as `"host": true`. This enables correct cross-compilation in cases that the target is not executable -- such as when compiling for `arm64-android`.

See [Host dependencies](host-dependencies.md) for more information.

#### <a name="platform"></a> `"platform"` field

The `"platform"` field limits the platforms where the dependency is required.

Some predefined values are computed from the [triplet settings](triplets.md):

- `x64` - `VCPKG_TARGET_ARCHITECTURE` == `"x64"`
- `x86` - `VCPKG_TARGET_ARCHITECTURE` == `"x86"`
- `arm` - `VCPKG_TARGET_ARCHITECTURE` == `"arm"` or `VCPKG_TARGET_ARCHITECTURE` == `"arm64"`
- `arm64` - `VCPKG_TARGET_ARCHITECTURE` == `"arm64"`
- `wasm32` - `VCPKG_TARGET_ARCHITECTURE` == `"wasm32"`
- `windows` - `VCPKG_CMAKE_SYSTEM_NAME` == `""` or `VCPKG_CMAKE_SYSTEM_NAME` == `"WindowsStore"` or `VCPKG_CMAKE_SYSTEM_NAME` == `"MinGW"`
- `mingw` - `VCPKG_CMAKE_SYSTEM_NAME` == `"MinGW"`
- `uwp` - `VCPKG_CMAKE_SYSTEM_NAME` == `"WindowsStore"`
- `linux` - `VCPKG_CMAKE_SYSTEM_NAME` == `"Linux"`
- `osx` - `VCPKG_CMAKE_SYSTEM_NAME` == `"Darwin"`
- `ios` - `VCPKG_CMAKE_SYSTEM_NAME` == `"iOS"`
- `freebsd` - `VCPKG_CMAKE_SYSTEM_NAME` == `"FreeBSD"`
- `openbsd` - `VCPKG_CMAKE_SYSTEM_NAME` == `"OpenBSD"`
- `android` - `VCPKG_CMAKE_SYSTEM_NAME` == `"Android"`
- `emscripten` - `VCPKG_CMAKE_SYSTEM_NAME` == `"Emscripten"`
- `static` - `VCPKG_LIBRARY_LINKAGE` == `"static"`
- `static-crt` - `VCPKG_CRT_LINKAGE` == `"static"`

Cross-compiling can be detected with the `native` value:

- `native` - `TARGET_TRIPLET` == `HOST_TRIPLET`

Expressions can also be combined using logical operators and grouping:

- `!<expr>`, `not <expr>` - negation
- `<expr>|<expr>`, `<expr>||<expr>`, `<expr>,<expr>`, `<expr> or <expr>` - logical OR
- `<expr>&<expr>`, `<expr>&&<expr>`, `<expr> and <expr>` - logical AND
- `(<expr>)` - grouping/precedence

##### Examples

- **Needs `picosha2` for sha256 on non-Windows, but get it from the OS on Windows (BCrypt)**

```jsonc
{
  "name": "picosha2",
  "platform": "!windows"
}
```

- **Require zlib on arm64 Windows and amd64 Linux**

```jsonc
{
  "name": "zlib",
  "platform": "(windows & arm64) | (linux & x64)"
}
```

#### <a name="version-gt"></a> `"version>="` field

A minimum version constraint on the dependency.

This field specifies the minimum version of the dependency, optionally using a
`#N` suffix to denote port-version if non-zero.

For more information on versioning semantics, see [Versioning](versioning.md#version).

### <a name="overrides"></a> `"overrides"`

This field pins exact versions for individual dependencies.

`"overrides"` from transitive manifests (i.e. from dependencies) are ignored.

See also [versioning](versioning.md#overrides) for more semantic details.

#### Example

```json
  "overrides": [
    {
      "name": "arrow", "version": "1.2.3", "port-version": 7
    }
  ]
```

### <a name="supports"></a> `"supports"`

If your project doesn't support common platforms, you can tell your users this with the `"supports"` field. It uses the same platform expressions as [`"platform"`](#platform), from dependencies, as well as the `"supports"` field of features.

For example, if your library doesn't support linux, you might write `{ "supports": "!linux" }`.

### <a name="default-features"></a> <a name="features"></a> `"features"` and `"default-features"`

The `"features"` field defines _your_ project's optional features, that others may either depend on or not. It's an object, where the keys are the names of the features, and the values are objects describing the feature. `"description"` is required, and acts exactly like the [`"description"`](#description) field on the global package, and `"dependencies"` are optional, and again act exactly like the [`"dependencies"`](#dependencies) field on the global package. There's also the `"supports"` field, which again acts exactly like the [`"supports"`](#supports) field on the global package.

You also have control over which features are default, if a person doesn't ask for anything specific, and that's the `"default-features"` field, which is an array of feature names.

#### Example

```json
{
  "name": "libdb",
  "version": "1.0.0",
  "description": [
    "An example database library.",
    "Optionally can build with CBOR, JSON, or CSV as backends."
  ],
  "$default-features-explanation": "Users using this library transitively will get all backends automatically",
  "default-features": [ "cbor", "csv", "json" ],
  "features": {
    "cbor": {
      "description": "The CBOR backend",
      "dependencies": [
        {
          "$explanation": [
            "This is how you tell vcpkg that the cbor feature depends on the json feature of this package"
          ],
          "name": "libdb",
          "default-features": false,
          "features": [ "json" ]
        }
      ]
    },
    "csv": {
      "description": "The CSV backend",
      "dependencies": [
        "fast-cpp-csv-parser"
      ]
    },
    "json": {
      "description": "The JSON backend",
      "dependencies": [
        "jsoncons"
      ]
    }
  }
}
```

### `"vcpkg-configuration"`

Allows to embed vcpkg configuration properties inside the `vcpkg.json` file. Everything inside the `vcpkg-configuration` property is treated as if it were defined in a `vcpkg-configuration.json` file. For more details, see the [`vcpkg-configuration.json`](registries.md) documentation.

Having a `vcpkg-configuration` defined in `vcpkg.json` while also having a `vcpkg-configuration.json` file is not allowed and will result in the vcpkg command terminating with an error message.

#### Example

```json
  "name": "test",
  "version": "1.0.0",
  "dependencies": [ "beison", "zlib" ],
  "vcpkg-configuration": {
    "registries": [
      {
        "kind": "git",
        "baseline": "dacf4de488094a384ca2c202b923ccc097956e0c",
        "repository": "https://github.com/northwindtraders/vcpkg-registry",
        "packages": [ "beicode", "beison" ]
      }
    ],
    "overlay-ports": [ "./my-ports/fmt", 
                       "./team-ports"
    ]
  }
```
