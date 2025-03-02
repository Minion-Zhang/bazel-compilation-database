Compilation database with Bazel ![Tests](https://github.com/grailbio/bazel-compilation-database/workflows/Tests/badge.svg?branch=master) ![Migration](https://github.com/grailbio/bazel-compilation-database/workflows/Migration/badge.svg?branch=master)
===============================

If you use [Bazel][bazel] and want to use libclang based editors and tools, you
can now generate [JSON compilation database][compdb] easily without using build
intercept hooks.  The advantage is that you can generate the database even if
your source code does not compile, and the generation process is much faster.

For more information on compilation database, [Guillaume Papin][sarcasm] has an
[excellent article][compdb2].

## How to Use

### Entire repo

Running generate.sh script from this project with current directory somewhere
in your bazel workspace will generate a compile_commands.json file in the
top-level directory of your workspace. You can even symlink the script to
somewhere in your PATH.

For example,
```sh
INSTALL_DIR="/usr/local/bin"
VERSION="0.4.5"

# Download and symlink.
(
  cd "${INSTALL_DIR}" \
  && curl -L "https://github.com/grailbio/bazel-compilation-database/archive/${VERSION}.tar.gz" | tar -xz \
  && ln -f -s "${INSTALL_DIR}/bazel-compilation-database-${VERSION}/generate.sh" bazel-compdb
)

bazel-compdb # This will generate compile_commands.json in your workspace root.

# You can tweak some behavior with flags:
# 1. To use the source dir instead of bazel-execroot for directory in which clang commands are run.
bazel-compdb -s

# 2. To use a list of regex to filter out the bazel targets.
# For example, to filter out any target contains 'algorithm', or starts with
# '//absl/memory', or ends with 'internal':
echo -e "algorithm\n^//absl/memory\ninternal$" > path_to_filters.txt
bazel-compdb -s -f path_to_filters.txt
```

### Selected targets

You can define a target of rule type `compilation_database` with the attribute
`targets` as a list of top-level `cc_.*` labels which you want to include in
your compilation database. You do not need to include targets that are
dependencies of your top-level targets. So these will mostly be targets of type
`cc_binary` and `cc_test`.

For example,

In your WORKSPACE file:
```python
# Change master to the git tag you want.
http_archive(
    name = "com_grail_bazel_compdb",
    strip_prefix = "bazel-compilation-database-master",
    urls = ["https://github.com/grailbio/bazel-compilation-database/archive/master.tar.gz"],
)
```

In your BUILD file located in any package:
```python
## Replace workspace_name and dir_path as per your setup.
load("@com_grail_bazel_compdb//:aspects.bzl", "compilation_database")

compilation_database(
    name = "example_compdb",
    targets = [
        "//a_cc_binary_label",
        "//a_cc_library_label",
    ],
    # [Optional]
    # If your exec root (value returned by `bazel info execution_root`)
    # is constant across your users, then you can supply the value here.
    # Otherwise, the default is `__EXEC_ROOT__` which you can replace in
    # the output file using `sed` or similar tool (see below).
    exec_root = "/path/to/bazel/exec_root",
)
```

Then, in your terminal (you can wrap this in a shell script and check it in your repo):
```
# Command to generate the compilation database file.
bazel build //path/to/pkg/dir:example_compdb

# Location of the compilation database file.
outfile="$(bazel info bazel-bin)/path/to/pkg/dir/compile_commands.json"

# Command to replace the marker for exec_root in the file.
execroot=$(bazel info execution_root)
sed -i.bak "s@__EXEC_ROOT__@${execroot}@" "${outfile}"

# The compilation database is now ready to use at this location.
echo "Compilation Database: ${outfile}"
```

### YouCompleteMe

If you want to use this project solely for semantic auto completion using
[ycmd][ycm] (YouCompleteMe) based editor plugins, then the easiest approach
is to install this project as a vim plugin with your favourite plugin manager.
The plugin will set `g:ycm_global_ycm_extra_conf` and instrument bazel with
the correct paths.
e.g. Using Plugged add the following to your vimrc.
```
Plug 'grailbio/bazel-compilation-database'
```

An alternative approach is to follow the instructions as above for making the
files available in this repo somewhere in the workspace, and then configure vim
to use the `.ycm_extra_conf.py` script that you just extracted. One way is to
make a symlink to the py script from the top of your workspace root. Another
way is to set the `ycm_global_ycm_extra_conf` variable in vim.

With both of these approaches, you don't have to maintain a separate
compile_commands.json file through a script and/or a `compilation_database`
target. Compile commands are fetched from bazel as the files are opened in your
editor.

## Contributing

Contributions are most welcome. Please submit a pull request giving the owners
of this github repo access to your branch for minor style related edits, etc.

## Known Issues

Please check open issues at the github repo.

We have tested only for C and C++ code, and with tools like
[YouCompleteMe][ycm], [rtags][rtags], and the [woboq code browser][woboq].

## Alternatives

1. [Kythe][kythe]: uses Bazel action listeners
1. [Bear][bear]: uses build intercept hooks

These approaches could be more accurate than the approach of this tool in some
rare cases, but need a more complicated setup and a full build every time you
refresh the database.

[bazel]: https://bazel.build/
[compdb]: https://clang.llvm.org/docs/JSONCompilationDatabase.html
[sarcasm]: https://github.com/Sarcasm
[compdb2]: https://sarcasm.github.io/notes/dev/compilation-database.html
[cla]: https://www.clahub.com/pages/why_cla
[ycm]: https://github.com/Valloric/YouCompleteMe
[rtags]: https://github.com/Andersbakken/rtags
[woboq]: https://github.com/woboq/woboq_codebrowser
[kythe]: https://github.com/google/kythe/blob/master/tools/cpp/generate_compilation_database.sh
[bear]: https://github.com/rizsotto/Bear
