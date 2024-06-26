# sver
Semantic Version (SemVer) parsing & utility script/function library in pure bash

## Overview
**sver** is a self contained cli tool and function library implementing a
[Semantic Versioning 2](https://semver.org) compliant parser and utilities.
Written in optimized, portable, pure Bash (v3+) for simplicity & speed, and
can be even used as a function library in Busybox Dash/Ash.

### Features
- get or bump version identifiers (major, minor, patch, prerelease, build\_metadata)
- min/max/sort/filter lists of versions for valid versions with optional constraint matching
- version constraint evaluation using common constraint syntax with chaining (multiple constraints)
- precedence comparisons (sort/equals/greater\_than/less\_than/min/max) strictly implement SemVer spec (most SemVer sorts do not properly implement prerelease precedence comparisons)
- sort routine written with pure bash builtins and uses no subshells
- deconstruct SemVer identifiers and output in json or yaml
- Bash command line completion function & injector built in
- uses Bash primitives & builtins exclusively for speed & portability, minimum subshells, no looped subshells
- single small (< 20kB) script usable as a CLI or mixin Bash/Dash/Ash function library
- always formatted with [shfmt](https://github.com/patrickvane/shfmt)
- always checked with [shellcheck](https://github.com/koalaman/shellcheck) static analysis tool
- comprehensive [test](tests) coverage
- compatible with Bash v3 for us poor macOS users
- includes a GitHub Action

## Usage
### Installation
It is a self contained Bash script, so you can clone the repo and run directly.
However, here are some other convenient ways to install it.

#### asdf
A [PR](https://github.com/asdf-vm/asdf-plugins/pull/965) has been opened to add **sver**
into the asdf plugin registry; until that happens, you can manually specify the asdf
plugin repo.
```bash
asdf plugin add sver https://github.com/robzr/asdf-sver.git
asdf install sver latest
asdf global sver latest
```

#### curl
You can simply curl a version directly.
```bash
curl -LO https://github.com/robzr/sver/releases/download/v1.2.1/sver
```

#### Homebrew
A Homebrew tap is available.
```bash
brew tap robzr/sver
brew install sver
```
If we can get enough momentum for this project on GitHub to meet Homebrew
criteria for a core formula, it will be added! This requires more than 75 stars,
30 forks or 30 watchers.

### Command line completion
Command completion is available for Bash users. Simply add the following to your
`~/.bashrc`:
```
. /dev/stdin <<< "$(sver complete)"
```

### Command line usage
See `sver help` for documentation.
```text
sver v1.2.1 (https://github.com/robzr/sver) self contained cli tool and function
library implementing a Semantic Versioning 2 compliant parser and utilities.
Written in optimized, portable, pure Bash (v3)+ for simplicity & speed.

Usage: sver <command> [<sub_command>] [<version>] [(<constraint>|<option>|<version>) ...]

Commands:
  bump major <version>
  bump minor <version>
  bump patch <version>
  complete -- Bash command completion, use: . /dev/stdin <<< "$(sver complete)"
  constraint <version> <constraint(s)> -- version constraint evaluation - if
                              version matches constraint(s) ? exit 0 : exit 1
  equals <version1> <version2> -- version1 == version2 ? exit 0 : exit 1
  filter [constraint] -- filters stdin list, returns only valid SemVers
  greater_than <version1> <version2> -- version1 > version2 ? exit 0 : exit 1
  get major <version>
  get minor <version>
  get patch <version>
  get prerelease <version>
  get build_metadata <version>
  help
  json <version> -- displays JSON map of components
  less_than <version1> <version2> -- version1 < version2 ? 0 : exit 1
  max [constraint] -- returns max value from stdin list
  min [constraint] -- returns min value from stdin list
  sort [-r] [constraint] -- sorts stdin list of SemVers (-r for reverse sort)
  validate <version> -- version is valid ? exit 0 : exit 1
  version
  yaml <version> -- displays YAML map of components

Versions:
  Semantic Versioning 2 (https://semver.org) compliant versions, with an
  optional "v" prefix tolerated on input.

Constraints:
  Multiple comma-delimited constraints can be chained together (boolean AND) to
  form a single constraint expression. Commands that take a list of versions on
  stdin and take a constraint will filter the input for versions matching the
  constraint expression. Version substrings can be used, and are especially
  useful with the pessimistic constraint operator. Supported operators:
    = <version_substring> -- equal (default if no operator specified)
    > <version_substring> -- greater than
    >= <version_substring> -- greater than or equal to
    < <version_substring> -- less than
    <= <version_substring> -- less than or equal to
    ~> <version_substring> -- pessimistic constraint operator - least significant
       (rightmost) identifier specified in the constraint matched with >=, but 
       more significant (further left) identifiers must be equal
    !pre[release] -- does not contain a prerelease (ie: "stable")
    !bui[ild_metadata] -- does not contain build_metadata
  Examples: "~> v1.2, != 1.3", "> 1, <= v2.5, != v2.4.4", "v1, !pre"
```

### GitHub Action
A GitHub composite action is included in the repo, and each version of sver
is also embedded directly in the composite action, which makes it very quick
to load and run. Usage is simple, with the `command` input taking all commands
and arguments that the CLI does.
```yaml
- uses: robzr/sver@v1
  with:
    command: version
```
The `output` output will contain the result, regardless of type (string,
boolean, JSON, YAML).
```yaml
- id: sver
  uses: robzr/sver@v1
  with:
    command: version

- if: steps.sver.outputs.output == 'v1.2.3'
  ...
```
Commands that return a boolean will return a boolean-as-string.
```yaml
- id: sver
  uses: robzr/sver@v1
  with:
    command: constraint "${VERSION}" '~> v1.0, !pre'

- if: steps.sver.outputs.output == 'true'
  ...
```
For commands that take input on stdin, like `filter`, `max`, `min`, or `sort`,
the `input` input can be specified.
```yaml
- id: sver
  uses: robzr/sver@v1
  with:
    command: max
    input: |
      v1.0.1
      v2.2.2
      ...

- if: steps.sver.outputs.output == env.VERSION
  ...
```
The `input-command` input is also available, and will run the provided command,
and use the command's output as input for **sver**.
```yaml
- uses: robzr/sver@v1
  with:
    command: max '< v1, !pre'
    input-command: git tag -l
```

### Bash function library
To use **sver** as a Bash function library, source it with the `SVER_RUN=false`
variable set.
```bash
SVER_RUN=false . "$(command -v sver)"
```
As the CLI is simply a mapping to the function library, the commands and syntax
are the same, with the functions naming pattern `sver_<command>[_<subcommand>]`,
ie: `sver_version`, `sver_bump_major`, etc. To avoid the use of subshells and 
redirection for returning string values, functions generally return values via
the global `REPLY` variable. Note that this is a reuse of the default response
variable used by Bash (and Dash) `read`.

### Dash/Ash function library
The command line completion and CLI command-to-function mapping depends on the
Bash builtin `compgen`, which is not available in Dash (commonly used as the
default shell in Alpine Linux and other compact distributions that use Busybox).
Therefore, as the script is written, it will not function in Dash. However,
all of the core **sver** functions can be used, although the `sort` function
differs from the Bash implementation by using a shell call to Busybox `sort`
and does not fully sort prereleases according to SemVer spec. Every other
function is identical. To use **sver** in Dash, the Bash specific syntax needs
to first be stripped out, which can be done with the following command (even
with Busybox):
```
sed -n '/# bash-only-begin/,/# bash-only-end/!p' sver > sver.dash
. sver.dash
```

# License
Permissive [Creative Commons - CC BY 3.0](https://creativecommons.org/licenses/by/3.0/)
license - same as Semantic Versioning itself.
