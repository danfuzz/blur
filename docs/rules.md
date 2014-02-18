Rule Semantics
==============

Outline
-------

Blur runs in two phases in sequence:

* Generate rules.

* Use those rules to attempt to satisfy targets named on the command-line.

The first phase is achieved by running a project-defined script called
`blur.sh`.

During the second phase, Blur will only ever attempt to run a given
rule at most once, even if it becomes active via several targets or
dependencies.


Satisfying Targets
------------------

The main operation of Blur is to "satisfy" the targets named on its
command-line. Each named target is satisfied, in order, as a separate
operation. (That is, Blur does not attempt to simultaneously satisfy
multiple targets, though that may happen as a byproduct of how rule
dependencies happen to be set up.)

In general, targets become satisfied by finding rules which name them
either as targets or as explicit named ids, and evaluating those rules.
Rules have "reqs" (requirements, a/k/a dependencies), and these reqs
have to themselves become satisfied as targets before a rule is allowed
to run any commands.

In more detail, satisfying a target takes place in the following manner:

* All rules are selected which name the target as `target` file (per se) or
  a named id.

* The selected rules are iterated, in the order that they were specified
  by the original rules generation script. For each rule that has not yet
  been run, its `req`s are iterated over, in order. Target satisfaction is
  run on each `req`. (That is, this procedure is invoked recursively on each
  `req`.)

* The selected rules are iterated, in the order that they were specified
  by the original rules generation script. Each rule that has not yet
  been run is run, per the description of `rule` above, and then it is
  noted to have run.

Per the above, once a rule has been run once, it will not be run again.


Blur.sh
-------

Rules are set up by writing a Bash script named `blur.sh`, and
placing it at the base directory of the project (or subproject) source.
Blur runs that file while `cd`ed to its directory. It is passed as arguments
any options not understood by Blur itself. The file is responsible for
writing a set of build rules to stdout.

The rules script is also provided with the following environement
variables:

* `PROJECT_DIR` &mdash; Full path to the directory in which the rules
  script is located.
* `PROJECT_NAME` &mdash; Final path component of `PROJECT_DIR`.
* `BASE_DIR` &mdash; Parent directory of `PROJECT_DIR`.

Within the rules script, in addition to all the usual shell facilities,
a number of utility functions are available, including most notably:

* `rule` &mdash; The primary way to emit rules. See below for details.
* `abs-path` &mdash; Get an absolute path for the given possibly-relative
  path.
* `quote` &mdash; Quote each of the arguments as shell-evaluable strings
  (protecting special characters, spaces, etc. from getting inadvertently
  interpreted).
* `unquote` &mdash; Undoes the operation of `quote`.


See `blur/blur-util.sh` for more details on the utility functions. (Yes,
this should be documented better.)


`rule` command
--------------

```
rule <type> [--<option> ...] [--] [<arg> ...]
```

Options and arguments vary with the type, except that every type accepts
these options:

* `--id=<name>` &mdash; Indicate that this rule can be required by using
  the indicated name.

* `--req=<target>` &mdash; Indicate that this rule requires the given target
  to be satisfied / built before it can proceed. `<target>` must be a file
  name (if non-absolute, relative to the directory being built).

* `--req-id=<id>` &mdash; Indicate that this rule requires the given named
  id to be satisfied / built before it can proceed.

* `--target=<file>` &mdash; Indicate that this rule produces the named target
  file.

* `--value=<string>` &mdash; Adds an arbitrary value passed to commands, when
  the targets are out-of-date. This is most useful within a req/target group.

* `--msg=<string>` &mdash; Causes the given message to be emitted when the
  rule is considered out-of-date.

* `--moot=<cmd>` &mdash; Runs the command when determining if the rule is
  out-of-date. If the command succeeds, then the rule is considered up-to-date,
  and so will not be run.

* `--assert=<cmd>` &mdash; Runs the command when determining if the rule is
  out-of-date. If the command fails, then the rule is considered to have
  failed.

* `--cmd=<cmd>` &mdash; Runs the command, when the rule is considered
  out-of-date. Array variables `STALE_TARGETS`, `NEW_REQS`, and `VALUES` are
  available to commands.

* `--build-in-dir=<dir>` &mdash; Runs Blur in the given directory, when the
  rule is considered out-of-date. Passes all stale targets to the command.
  (Yes, "recursive make considered harmful." There is probably a much
  better solution for the times when this option seems useful.)

* `(` &hellip; `)` &mdash; Can be used to group
  together related sets of reqs and targets. When used, the staleness of a
  target only depends on the reqs in its group. Within a group, the only
  valid options are `--req=`, `--target=`, and `--value=`. Furthermore,
  if a rule has any groups, then *all* `--req=`, `--target=`, and `--value=`
  options must be in some group or other.

* `--` &mdash; Indicates the end of options, unambiguously. It is good
  practice to always use a `--` option between options and other arguments,
  to avoid confusion for when an argument (such as a file whose name
  happens to start with two dashes) merely has the form of an option,
  without actually being one.

Some types accept one or more of these options:

* `--in-dir=<dir>` &mdash; Indicates that relative file name arguments
  should be taken to be in the given directory. By default, relative paths
  are taken to be relative to the source base directory (where the rules
  script is stored).

* `--out-dir=<dir>` &mdash; If the rule uses relative file names for both
  reqs and targets, this is how to specify the relative base for the
  targets.

When a rule is run, the following takes place, in this order:

* Rules for each of the reqs are run (recursively).
* Moots and asserts are run, in the order specified by the rule.
* Reqs that name files (as opposed to ones that are named ids) are checked
  for existence. It is a fatal error at this point if any file req doesn't
  exist. (Reqs get created by recursively attempting to satisfy them before
  rules are run. See below.)
* The timestamps of all reqs and targets are collected.
* If any target doesn't exist, or if any target is older than any req
  (limited to reqs in its group, if it is in a group), then the rule is
  considered out-of-date. Caveat: Timestamps on directories are ignored.
* If the rule has no targets at all, or has no reqs at all, it is always
  considered out-of-date.
* If the rule is considered out-of-date (per above), then its messages
  and commands are run, in the order specified by the rule.

The rule types are as follows:

* `body` &mdash; Catch-all. Has no semantics beyond the general rule
  semantics defined above.

* `copy --out-dir=<dir> [--in-dir=<dir>] [--chmod=<mode>] [--] <name> ...`
  &mdash; Copies files from a tree rooted in the indicated `in-dir` to a
  tree rooted in the indicated `out-dir` (which must be specified). Each of
  the `<name>`s must be a relative path within the directories. And each
  file copy operation is represented by its own rule. If `--chmod` is
  specified, each target is `chmod`ed to the indicated `mode` once copied.

* `mkdir [--] <name> ...` &mdash; Makes directories with the given names. Each
  directory creation is its own rule.

* `rm [--in-dir=<dir>] [--] <name> ...` &mdash; Removes files or directories
  with the given names. If `--in-dir` is specified, then relative paths are
  with respect to that directory (and not the source base directory).
  Each file removal is its own rule. Unless explicitly added, the emitted
  rules have no reqs and no targets. However, the rules are all automatically
  mooted with a check for the non-existence of the named files.
