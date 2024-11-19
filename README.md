# vs-msvc-component-install-action

A reusable action which installs a specific version of the VC component for Visual Studio.

Consider using the action provided by [thepwrtank18/install-vs-components](https://github.com/thepwrtank18/install-vs-components) for general use-cases.
The action in this repository was designed only for use with vcpkg and has the following features.

- Display the installation logs to avoid silent failures.
- Cache the install based on an expected version of the compiler toolset.
- Fail if an expected toolset is not present once the VS installer has closed.

This action depends on the following publicly-available actions.

- [actions/cache@v4](https://github.com/actions/cache)

## Usage

Specify a version to install using the `version` input.

```yaml
- name: Install MSVC
  if: ${{ runner.os = 'Windows' }}
  uses: cosmopetrich/vs-msvc-component-install-action@v1
  with:
    version: "14.36-17.6"
```

The version should be [a valid version of the VC x86_64 component](https://learn.microsoft.com/en-us/visualstudio/install/workload-component-id-vs-build-tools?view=vs-2022)
in the form ww.xx.yy.zz. Hyphens or periods are fine, and the leading "v" can be left in or omitted.
For example, the following would all install `Microsoft.VisualStudio.Component.VC.14.36.17.6.x86.x64`.

- v14.36-17.6
- 14.36-17.6
- 14.36.17.6

It is recommended that

## Advanced usage

### Toolset version

Specify a `toolset-version` to tell the action what version of the toolset should be installed by the component.

```yaml
- name: Install MSVC
  if: ${{ runner.os = 'Windows' }}
  uses: cosmopetrich/vs-msvc-component-install-action@v1
  with:
    version: "14.36-17.6"
    toolset-version: "14.36.32532"
```

This will enable the following features:

- Avoid needlessly running the installer if the toolset version is already present.
- Store the toolset version in the Actions cache (see below).
- Raise an error if the correct toolset version is not present after running the install.

Note that this must the version of the underlying toolset which will be different to the version of the compiler component.
Just specifying `toolset-version` is not enough to install the appropriate component, and `version` must always be specified.

### Modifying caching behaviour

Caching can be disabled by setting `cache-save`.

```yaml
- name: Install MSVC
  if: ${{ runner.os = 'Windows' }}
  uses: cosmopetrich/vs-msvc-component-install-action@v1
  with:
    version: "14.36-17.6"
    cache-save: false
    toolset-version: "14.36.32532"
```

And the cache key can be altered with `cache-key-prefix`.

```yaml
- name: Install MSVC
  if: ${{ runner.os = 'Windows' }}
  uses: cosmopetrich/vs-msvc-component-install-action@v1
  with:
    version: "14.36-17.6"
    cache-key-prefix: some-custom-prefix-
    toolset-version: "14.36.32532"
```

See also [the docs on cache scoping](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows#matching-a-cache-key) and also [actions/cache#79](https://github.com/actions/cache/issues/79).

And note the following.

- The `toolset-version` will automatically be included in the final key name.
- Caching is only ever enabled if `toolset-version` is specified.
- As of 2024-11 a single cache entry is around 400MB.
- The action will always attempt to _retrieve_ from the cache even if saving is disabled.
- Only the toolset folder will be cached. This should be sufficient for vcpkg builds but may not be suitable for other workloads.
- A `toolset-version` which was already present in the base image and was not installed by the action will not be saved.

## Misc notes

### Diagnosing failures

The VS installer is not well designed for headless usage, and it might apepar to finish successfully even if it hasn't installed anything.
It may also mark the actual cause of failure as a 'warning'. Try searching for `Shutting down the application with exit code` in the output.
The real cause of the failure will probably be a few lines before that.

### Finding the correct toolset version

Unfortunately it doesn't seem to be trivial to figure out which MSVC component installs which versions of the toolset.

The quickest way is probably to install VS on your local machine and look at what folders exist in `%PROGRAMFILES%\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC`
before and after installing some different component versions. If you can't install locally then a slower alternative would be to
set an arbtirary `version` for this action and then check the available version lists that are printed at the start and end of the action's output.
