# PMM - The Package Manager Manager

PMM is a module for CMake that manages... package managers.

## Wha- Why?

People hate installing new software. Especially when they already have a
perfectly working tool present. PMM uses the CMake scripting
language to manage external packaging tools. PMM will automatically
download, install, and control package managers from within your CMake
project.

(As you are reading this, only Conan and VCPKG are supported.)

## But This is Just *Another* Tool I have to Manage!

Never fear! PMM is the lowest-maintenance software you will ever use.

## How Do I Use PMM?

Using PMM is simple:

1. Download the `pmm.cmake` file (available at the top level of this
   respository), and place it at the top level of your repository
   (alongside your `CMakeLists.txt`).
2. In your `CMakeLists.txt`, add a line `include(pmm.cmake)`.
3. Call the `pmm()` CMake function.

That's it! The `pmm.cmake` file is just 23 significant lines. Take a look inside
if you doubt.

## Wait... It's Downloading a Bunch of Stuff!

Precisely! `pmm.cmake` is just a bootstrapper for the real PMM code, which
can be found in the `pmm/` directory in the repository. The content is
served over HTTPS from the `gh-pages` branch of the PMM repository, so it is all publicly visible.

## I Don't Want to Automatically Download and Run Code from the Internet

Great! I sympathize, but remember: If you run `apt`, `yum`, `pip`, or even
`conan`, you are automatically downloading and running code from the
internet. It's all about whose code you *trust*.

Even still, you can host the PMM code yourself: Download the `pmm/`
directory as you want it, and modify the `pmm.cmake` script to download
from your alternate location (eg, a corporate engineering intranet server).

## Will PMM Updates Silently Break my Build?

Nope. `pmm.cmake` will never automatically change the version of PMM that
it uses, and the files served will never be modified in-place: New versions
will be *added,* but old versions will remain unmodified.

PMM will notify you if a new version is available, but it won't be annoying
about it, and you can always disable this nagging by setting
`PMM_IGNORE_NEW_VERSION` before including `pmm.cmake`.

## How do I Change the PMM Version?

There are two ways:

1. Set `PMM_VERSION` before including the `pmm.cmake` script.
2. Modify the `PMM_VERSION_INIT` value at the top of `pmm.cmake`.

Prefer (1) for conditional/temporary version changes, and (2) for permanent
version changes.

## How do I Change the Download Location for PMM?

For permanent changes, set `PMM_URL` and/or `PMM_URL_BASE` in `pmm.cmake`.
For temporary changes, set `PMM_URL` before including `pmm.cmake`

# The `pmm()` Function

The only interface to PMM (after including `pmm.cmake`) is the `pmm()`
CMake function. Using it is very simple. At the time or writing, `pmm()`
only supports Conan and vcpkg, but other packaging solutions may be supported
in the future.

The `VERBOSE` and `DEBUG` options enable verbose and debug logging,
respectively. You may set `PMM_{DEBUG,VERBOSE}` before `include(pmm.cmake)` to
enable these options globally and see information about the PMM bootstrapping
process.

The `pmm()` signature:

```cmake
pmm(
    # Enable verbose logging
    [VERBOSE]
    # Enable debug logging (implies VERBOSE)
    [DEBUG]
    # Use Conan
    [CONAN
        # Set additional --setting flags
        [SETTINGS ...]
        # Set additional --option flags
        [OPTIONS ...]
        # Set the --build option. (Default is `missing`)
        [BUILD <policy>]
        # Ensure remotes are present before installing
        [REMOTES [<name>[::no_verify] <url> [...]]]
        # Enable the Bincrafters repository
        [BINCRAFTERS]
        # Enable the conan-community repository
        [COMMUNITY]
    ]
    # Use vcpkg
    [VCPKG
        # Specify the revision of vcpkg that you want to use (required)
        REVISION <rev>
        # Ensure the given packages are installed using vcpkg
        [REQUIRES [req [...]]]
        # Copy custom ports to the vcpkg ports directory
        [PORTS [dirpath [...]]]
    ]
    # Use CMakeCM
    [CMakeCM
        # Either use the latest release, or specify a specific base URL to
        # download from
        {ROLLING | FROM <base-url>}
    ]
    # Use dds
    [DDS
        # Specify a toolchain. Given as the --toolchain argument to `build-deps`.
        # If not specified, one will be generated automatically based on the
        # current CMake settings.
        [TOOLCHAIN <toolchain>]
        # Give a path to a catalog json5 file to import before doing dependency
        # resolution. If not provided, the user-local catalog will be used.
        [CATALOG <catalog-json-path>]
        # List of dependency files. Given as --deps to `build-deps`.
        [DEP_FILES [filepath [...]]]
        # List of direct dependency strings.
        [DEPENDS [dep [...]]]
        # Call 'import_packages()' on the caller's behalf. With 'AUTO', all
        # packages named with 'DEPENDS' will be imported automatically.
        # Packages from DEP_FILES will need to be named explicitly.
        [IMPORT < AUTO | [pkg-name [...]] >]
    ]
)
```


## `CONAN` PMM mode

In `CONAN` mode, PMM will find, obtain, and use Conan to manage project
packages.

PMM will always use the `cmake` Conan generator, and will define imported
targets for consumption (Equivalent of `conan_basic_setup(TARGETS)`). It will
also set `CMAKE_PREFIX_PATH` and `CMAKE_MODULE_PATH` for you to use
`find_package()` and `include()` against the installed dependencies.

> **NOTE:** No other CMake variables from regular Conan usage are defined.

`CONAN` mode requires a `conanfile.txt` or `conanfile.py` in your project
source directory. It will run `conan install` against this file to obtain
dependencies for your project.

The nitty-gritty of how PMM finds/obtains Conan:

1. Check for the `CONAN_EXECUTABLE` variable. If found, it is used.
2. Try to find a `conan` executable. Searches:
    1. Any `pyenv` versions in the user home directory
    2. `~/.local/bin` for user-mode install binaries
    3. `C:/Python{36,27,}/Scripts` for Conan installations
    4. Anything else on `PATH`
3. If still no Conan, attempts to obtain one automatically, trying first
   Python 3, then Python 2:
    1. Check for a `venv` or `virtualenv` executable Python module.
    2. Create a user-local virtualenv.
    3. Installs Conan *within the created virtualenv* and uses Conan from there.

### PMM Will Not do _Everything_ for You

While PMM will ensure that Conan has been executed for you as part of your
configure stage, it is up to you to provide a Conanfile that Conan can consume
to get your dependency information.

You will still need to read the Conan documentation to understand the basics of
how to declare and consume your dependencies.


## `VCPKG` PMM mode

In `VCPKG` mode, PMM will download the vcpkg repository at the given
`REVISION`, build the `vcpkg` tool, and manage the package installation in a
use-local data directory.

`REVISION` should be a git tree-ish (A revision number (preferred), branch, or
tag) that you could `git checkout` from the vcpkg repository. PMM will download
the specified commit from GitHub and build the `vcpkg` command line tool from
source. **You will need `std::filesystem` or `std::experimental::filesystem`
support from your compiler and standard library.**

If you want to copy custom ports to the vcpkg ports folder, you can define
`PORTS` with a list of folders to copy over.

`REQUIRES` is a list of packages that you would like to install using the
`vcpkg` command line tool.

When using PMM, you do not need to use the `vcpkg.cmake` CMake toolchain
file: PMM will take care of this aspect for you.

After calling `pmm(VCPKG)`, all you need to do is `find_package()` the
packages that you want to use.


## `CMakeCM` PMM mode

If `CMakeCM` is provided, PMM will download and make available the [CMake
Community Modules](https://github.com/vector-of-bool/CMakeCM) for you project.

Once the `pmm()` function is run, you may `include` or `find_package` any of the
modules provided by `CMakeCM`.

You must also specify either `ROLLING` or `FROM <base-url>` to use CMakeCM with
PMM:

- If you specify `ROLLING`, PMM will download the latest version of the CMakeCM
  module index every time you configure (with a few minutes of cooldown).
- If you specify `FROM`, the module index will only be obtained from the given
  base URL. **Note:** This URL *is not* the URL of a `CMakeCM.cmake` file: It
  is a url that *prefixes* the `CMakeCM.cmake` module URL.


## `DDS` PMM mode

With `DDS`, PMM will automatically download and use `dds` to install
dependencies. This will result in the generation of an `INDEX.lmi` file within
the build directory that can be imported using the `libman` CMake module's
`import_packages` function (currently available via CMakeCM).

Calling `pmm(DDS)` multiple times is allowed: Each call will *append* to the
set of installed dependencies rather than override it.

The current compile flags, definitions, and include directories will be used to
generate a toolchain file automatically if one is not provided.

The value of `CMAKE_CXX_COMPILER_LAUNCHER` will be given as the compiler
launcher in the generated toolchain file. This can be override with
`PMM_DDS_COMPILER_LAUNCHER`, including setting an empty string `""` to disable
it completely.

**NOTE**: `dds` support is still very experimental, and `dds` itself is very
new at the time of this writing. Refer to [the `dds`
documentation](https://vector-of-bool.github.io/docs/dds/) for information
about using `dds`.


# Helper Commands

The `pmm.cmake` script can be run with `-P` to access a set of utility
subcommands and options. See `cmake -P pmm.cmake /Help` for options.
Additionally, after PMM has run for the first time, it will generate a `sh` and
`bat` script that can be used to access the same set of options without needing
`cmake -P pmm.cmake`:

Get help with the `/Help` option:

```sh
> pmm-cli.sh /Help
```

As an example, you can rebuild a Conan package with this command:

```sh
> pmm-cli.sh /Conan /Rebuild <package name> /BuildType <Release or Debug>
```
