Examples
========

Each of these is an example `blur.sh` file.

Just print a message
--------------------

Both of these print the usual greeting in response to a default build.
(`build` is the target satisfied by default, when no other target is
given to `blur`.)

This first one uses Blur's built-in (sorta) pretty printing facility, which
indents and (optionally) truncates output.

```shell
rule body \
    --id=build \
    --msg='Hello world!'
```

This second one just uses good ol' `echo`.

```shell
rule body \
    --id=build \
    --cmd='echo "Hello world!"'
```


Copying a set of files
----------------------

This defines a set of files, and makes a rule to copy them from the
source directory to an `out` directory. In a more sophisticated script,
the list of files might come from the output of a normal `find` command.

The id `copy-files` is defined on the copy, so that `blur copy-files`
can be used to activate the rules.

```shell
files=(
    foo.txt
    bar.txt
    blort/fizmo.txt
)

rule copy \
    --id=copy-files \
    --out-dir=out \
    -- "${files[@]}"
```


Compiling C code
----------------

One of the most common build operations is to transform one kind of
file into a different kind of file. Compiling C code is a classic
example of that.

As above, we just predefine the set of sources, however that's just
to make for easier exposition.

```shell
sources=(
    one.c
    two.c
    three.c
)

outDir='out/objects'

rule mkdir -- "${outDir}"

# Iterate over all the sources, producing a compilation rule for each. Each
# compilation rule depends on the output directory being created as well as
# on the corresponding source file.
for src in "${sources[@]}"; do
    outFile="${outDir}/${src/%.c/.o}"
    rule body \
        --id=build \
        --req="${outDir}" \
        --req="${src}" \
        --target="${outFile}" \
        --cmd="$(quote cc -o "${outFile}" "${src}")"
done
```

The call to `quote` ensures that wacky characters in the file names don't
confuse things.

**Note:** This example shows one case where Blur needs to improve.
It takes too much explicit scripting to produce a transformation rule such
as this.
