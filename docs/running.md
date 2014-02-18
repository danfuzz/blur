Running
=======

Blur is invoked, unsurprisingly as:

```
blur [--<option> ...] [--] [<target> ...]
```

It takes the following options:

* `--in-dir=<dir>` &mdash; What directory to build in. Defaults to the
  current directory

* `--quiet` &mdash; Suppress all output other than errors.

* `--verbose=<level>` &mdash; Set output spew to be limited to the given
  level. `0` is the same as `--quiet`. `1` is the default. `2` through `4`
  also work and are (respectively) more and more spew-y.

* `--dump` &mdash; Causes all the rules to be dumped (printed) to stdout
  before proceeding with the main show.

* `--depth=<n>` &mdash; Sets up logging to be at the indicated depth. This
  is used when Blur recurses on itself, so that logging within inner
  invocations maintains consistent indentation with its outer environment.

Any other options get passed to the rule generation script (see <rules.md>).

The `<target>`s are which targets to build. Targets can be either file
names (if non-absolute, relative to the directory being built) or
named ids. If no target is specified, it defaults to `build`.
