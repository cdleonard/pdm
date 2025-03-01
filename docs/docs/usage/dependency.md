# Manage Dependencies

PDM provides a bunch of handful commands to help manage your project and dependencies.
The following examples are run on Ubuntu 18.04, a few changes must be done if you are using Windows.

## Initialize a project

```bash
mkdir pdm-test && cd pdm-test
pdm init
```

Answer several questions asked by PDM and a `pyproject.toml` will be created for you in the project root:

```toml
[project]
name = "pdm-test"
version = "0.0.0"
description = ""
authors = [
    {name = "Frost Ming", email = "mianghong@gmail.com"}
]
license = {text = "MIT"}
requires-python = ">=3.7"

dependencies = []
```

If `pyproject.toml` is already present, it will be updated with the metadata. The metadata format follows the
[PEP 621 specification](https://www.python.org/dev/peps/pep-0621/)

For details of the meaning of each field in `pyproject.toml`, please refer to [Project File](../pyproject/pep621.md).

## Add dependencies

```bash
pdm add requests
```

[`pdm add`](cli_reference.md#exec-0--add) can be followed by one or several dependencies, and the dependency specification is described in
[PEP 508](https://www.python.org/dev/peps/pep-0508/).

PDM also allows extra dependency groups by providing `-G/--group <name>` option, and those dependencies will go to
`[project.optional-dependencies.<name>]` table in the project file, respectively.

You can reference other optional groups in `optional-dependencies`, even before the package is uploaded:

```toml
[project]
name = "foo"
version = "0.1.0"

[project.optional-dependencies]
socks = ["pysocks"]
jwt = ["pyjwt"]
all = ["foo[socks,jwt]"]
```

After that, dependencies and sub-dependencies will be resolved properly and installed for you, you can view `pdm.lock` to see the resolved result of all dependencies.

### Local dependencies

Local packages can be added with their paths. The path can be a file or a directory:

```bash
pdm add ./sub-package
pdm add ./first-1.0.0-py2.py3-none-any.whl
```

The paths MUST start with a `.`, otherwise it will be recognized as a normal named requirement.

### VCS dependencies

You can also install from a git repository url or other version control systems. The following are supported:

- Git: `git`
- Mercurial: `hg`
- Subversion: `svn`
- Bazaar: `bzr`

The URL should be like: `{vcs}+{url}@{rev}`

Examples:

```bash
# Install pip repo on tag `22.0`
pdm add "git+https://github.com/pypa/pip.git@22.0"
# Provide credentials in the URL
pdm add "git+https://username:password@github.com/username/private-repo.git@master"
# Give a name to the dependency
pdm add "pip @ git+https://github.com/pypa/pip.git@22.0"
# Or use the #egg fragment
pdm add "git+https://github.com/pypa/pip.git@22.0#egg=pip"
# Install from a subdirectory
pdm add "git+https://github.com/owner/repo.git@master#egg=pkg&subdirectory=subpackage"
```

### Add development only dependencies

_New in 1.5.0_

PDM also supports defining groups of dependencies that are useful for development,
e.g. some for testing and others for linting. We usually don't want these dependencies appear in the distribution's metadata
so using `optional-dependencies` is probably not a good idea. We can define them as development dependencies:

```bash
pdm add -dG test pytest
```

This will result in a pyproject.toml as following:

```toml
[tool.pdm.dev-dependencies]
test = ["pytest"]
```

For backward-compatibility, if only `-d` or `--dev` is specified, dependencies will go to `dev` group under `[tool.pdm.dev-dependencies]` by default.

!!! NOTE
    The same group name MUST NOT appear in both `[tool.pdm.dev-dependencies]` and `[project.optional-dependencies]`.

### Editable dependencies

**Local directories** and **VCS dependencies** can be installed in [editable mode](https://pip.pypa.io/en/stable/cli/pip_install/#editable-installs). If you are familiar with `pip`, it is just like `pip install -e <package>`. **Editable packages are allowed only in development dependencies**:

!!! NOTE
    Editable installs are only allowed in the `dev` dependency group. Other groups, including the default, will fail with a `[PdmUsageError]`.

```bash
# A relative path to the directory
pdm add -e ./sub-package --dev
# A file URL to a local directory
pdm add -e file:///path/to/sub-package --dev
# A VCS URL
pdm add -e git+https://github.com/pallets/click.git@main#egg=click --dev
```

### Save version specifiers

If the package is given without a version specifier like `pdm add requests`. PDM provides three different behaviors of what version
specifier is saved for the dependency, which is given by `--save-<strategy>`(Assume `2.21.0` is the latest version that can be found
for the dependency):

- `minimum`: Save the minimum version specifier: `>=2.21.0` (default).
- `compatible`: Save the compatible version specifier: `>=2.21.0,<3.0.0`.
- `exact`: Save the exact version specifier: `==2.21.0`.
- `wildcard`: Don't constrain version and leave the specifier to be wildcard: `*`.

### Add prereleases

One can give `--pre/--prerelease` option to [`pdm add`](cli_reference.md#exec-0--add) so that prereleases are allowed to be pinned for the given packages.

## Update existing dependencies

To update all dependencies in the lock file:

```bash
pdm update
```

To update the specified package(s):

```bash
pdm update requests
```

To update multiple groups of dependencies:

```bash
pdm update -G security -G http
```

To update a given package in the specified group:

```bash
pdm update -G security cryptography
```

If the group is not given, PDM will search for the requirement in the default dependencies set and raises an error if none is found.

To update packages in development dependencies:

```bash
# Update all default + dev-dependencies
pdm update -d
# Update a package in the specified group of dev-dependencies
pdm update -dG test pytest
```

### About update strategy

Similarly, PDM also provides 2 different behaviors of updating dependencies and sub-dependencies，
which is given by `--update-<strategy>` option:

- `reuse`: Keep all locked dependencies except for those given in the command line (default).
- `eager`: Try to lock a newer version of the packages in command line and their recursive sub-dependencies and keep other dependencies as they are.
- `all`: Update all dependencies and sub-dependencies.

### Update packages to the versions that break the version specifiers

One can give `-u/--unconstrained` to tell PDM to ignore the version specifiers in the `pyproject.toml`.
This works similarly to the `yarn upgrade -L/--latest` command. Besides, [`pdm update`](cli_reference.md#exec-0--update) also supports the
`--pre/--prerelease` option.

## Remove existing dependencies

To remove existing dependencies from project file and the library directory:

```bash
# Remove requests from the default dependencies
pdm remove requests
# Remove h11 from the 'web' group of optional-dependencies
pdm remove -G web h11
# Remove pytest-cov from the `test` group of dev-dependencies
pdm remove -dG test pytest-cov
```

## Install the packages pinned in lock file

There are a few similar commands to do this job with slight differences:

- [`pdm sync`](cli_reference.md#exec-0--sync) installs packages from the lock file.
- [`pdm update`](cli_reference.md#exec-0--update) will update the lock file, then `sync`.
- [`pdm install`](cli_reference.md#exec-0--install) will check the project file for changes, update the lock file if needed, then `sync`.

`sync` also has a few options to manage installed packages:

- `--clean`: will remove packages no longer in the lockfile
- `--only-keep`: only selected packages (using options like `-G` or `--prod`) will be kept.

## Specify the lockfile to use

You can specify another lockfile than the default [`pdm lock`](cli_reference.md#exec-0--lock) by using the `-L/--lockfile <filepath>` option or the `PDM_LOCKFILE` environment variable.

### Select a subset of dependencies with CLI options

Say we have a project with following dependencies:

```toml
[project]  # This is production dependencies
dependencies = ["requests"]

[project.optional-dependencies]  # This is optional dependencies
extra1 = ["flask"]
extra2 = ["django"]

[tool.pdm.dev-dependencies]  # This is dev dependencies
dev1 = ["pytest"]
dev2 = ["mkdocs"]
```

| Command                         | What it does                                                         | Comments                  |
| ------------------------------- | -------------------------------------------------------------------- | ------------------------- |
| `pdm install`                   | install prod and dev deps (no optional)                              |                           |
| `pdm install -G extra1`         | install prod deps, dev deps, and "extra1" optional group             |                           |
| `pdm install -G dev1`           | install prod deps and only "dev1" dev group                          |                           |
| `pdm install -G:all`            | install prod deps, dev deps and "extra1", "extra2" optional groups   |                           |
| `pdm install -G extra1 -G dev1` | install prod deps, "extra1" optional group and only "dev1" dev group |                           |
| `pdm install --prod`            | install prod only                                                    |                           |
| `pdm install --prod -G extra1`  | install prod deps and "extra1" optional                              |                           |
| `pdm install --prod -G dev1`    | Fail, `--prod` can't be given with dev dependencies                  | Leave the `--prod` option |

**All** development dependencies are included as long as `--prod` is not passed and `-G` doesn't specify any dev groups.

Besides, if you don't want the root project to be installed, add `--no-self` option, and `--no-editable` can be used when you want all packages to be installed in non-editable versions. With `--no-editable` turn on, you can safely archive the whole `__pypackages__` and copy it to the target environment for deployment.

## Show what packages are installed

Similar to `pip list`, you can list all packages installed in the packages directory:

```bash
pdm list
```

Or show a dependency graph by:

```
$ pdm list --graph
tempenv 0.0.0
└── click 7.0 [ required: <7.0.0,>=6.7 ]
black 19.10b0
├── appdirs 1.4.3 [ required: Any ]
├── attrs 19.3.0 [ required: >=18.1.0 ]
├── click 7.0 [ required: >=6.5 ]
├── pathspec 0.7.0 [ required: <1,>=0.6 ]
├── regex 2020.2.20 [ required: Any ]
├── toml 0.10.0 [ required: >=0.9.4 ]
└── typed-ast 1.4.1 [ required: >=1.4.0 ]
bump2version 1.0.0
```

## Set PyPI index URL

You can specify a PyPI mirror URL by following commands:

```bash
pdm config pypi.url https://test.pypi.org/simple
```

## Allow prerelease versions to be installed

Include the following setting in `pyproject.toml` to enable:

```toml
[tool.pdm]
allow_prereleases = true
```

## Set acceptable format for locking or installing

If you want to control the format(binary/sdist) of the packages, you can set the env vars `PDM_NO_BINARY` and `PDM_ONLY_BINARY`.

Each env var is a comma-separated list of package name. You can set it to `:all:` to apply to all packages. For example:

```
# No binary for werkzeug will be locked nor used for installation
PDM_NO_BINARY=werkzeug pdm add flask
# Only binaries will be locked in the lock file
PDM_ONLY_BINARY=:all: pdm lock
# No binaries will be used for installation
PDM_NO_BINARY=:all: pdm install
```

## Solve the locking failure

If PDM is not able to find a resolution to satisfy the requirements, it will raise an error. For example,

```bash
pdm django==3.1.4 "asgiref<3"
...
🔒 Lock failed
Unable to find a resolution for asgiref because of the following conflicts:
  asgiref<3 (from project)
  asgiref<4,>=3.2.10 (from <Candidate django 3.1.4 from https://pypi.org/simple/django/>)
To fix this, you could loosen the dependency version constraints in pyproject.toml. If that is not possible, you could also override the resolved version in `[tool.pdm.resolution.overrides]` table.
```

You can either change to a lower version of `django` or remove the upper bound of `asgiref`. But if it is not eligible for your project,
you can tell PDM to forcedly resolve `asgiref` to a specific version by adding the following lines to `pyproject.toml`:

_New in version 1.12.0_

```toml
[tool.pdm.resolution.overrides]
asgiref = "3.2.10"  # exact version
urllib3 = ">=1.26.2"  # version range
pytz = "file:///${PROJECT_ROOT}/pytz-2020.9-py3-none-any.whl"  # absolute URL
```

Each entry of that table is a package name with the wanted version.
In this example, PDM will resolve the above packages into the given versions no matter whether there is any other resolution available.

!!! NOTE
    By using `[tool.pdm.resolution.overrides]` setting, you are at your own risk of any incompatibilities from that resolution. It can only be used if there is no valid resolution for your requirements and you know the specific version works.
    Most of the time, you can just add any transient constraints to the `dependencies` array.

## Environment variables expansion

For convenience, PDM supports environment variables expansion in the dependency specification under some circumstances:

- Environment variables in the URL auth part will be expanded: `https://${USERNAME}:${PASSWORD}/artifacts.io/Flask-1.1.2.tar.gz`.
  It is also okay to not give the auth part in the URL directly, PDM will ask for them when `-v/--verbose` is on.
- `${PROJECT_ROOT}` will be expanded with the absolute path of the project root, in POSIX style(i.e. forward slash `/`, even on Windows).
  For consistency, URLs that refer to a local path under `${PROJECT_ROOT}` must start with `file:///`(three slashes), e.g.
  `file:///${PROJECT_ROOT}/artifacts/Flask-1.1.2.tar.gz`.

Don't worry about credential leakage, the environment variables will be expanded when needed and kept untouched in the lock file.
