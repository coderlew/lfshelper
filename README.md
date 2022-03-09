# LFS build helper

Provides a python3 command line tool which can generate shell scripts to build a LFS system.

The generated build scripts do *not* entirely automate an LFS build. E.g. most of the preparation steps and LFS configuration after the system was built is not covered by the build scripts.

The package was tested by successfully building a LFS 11.1 system. However, no thorough testing was done e.g. on different host systems and/or with different python3 versions. So, use at your own risk!

## Installation
Build the lfshelper package:
```sh
cd lfshelper
python setup.py sdist
```
The source package to install with pip can be found in the dist dircetory.

Create a python virtual environment on the host system and install the lfshelper package inside it. The lxml package is a dependency and should be installed along the lfshelper package as well.
```sh
python -m venv lfsenv
source lfsenv/bin/activate
pip install lfshelper-<VERSION>.tar.gz
deactivate
```
## Usage
Follow the instructions in the LFS book to prepare the LFS build. At some point all source packages are downloaded and placed in a *source* directory on the LFS partition.

Now, add the LFS-BOOK-<LFS.VERSION>-NOCHUNKS.html file and the build_db.json file corresponding to the LFS book version to this *source* directory.

Activate the python environment where you installed the lfshelper package, and from within the *source* directory run:

```sh
lfsbuild
```
This creates several shell scripts in the *source* directory containing the commands to build the LFS system. After creating the scripts, deactivate the environment and continue with the preparation of the LFS build.

A list of all options of the lfsbuild script is produced with the help option:
```sh
lfsbuild --help
```

## Build database
In a first pass, the lfsbuild script parses the LFS book html file and builds an internal data structure representing the sequence of chapters & sections. This includes the commands to build the LFS system given in the LFS book sections.

In a second pass, the build scripts are generated from the internal data structure. Each extracted section is identified by the id given in the LFS-BOOK-<LFS.VERSION>-NOCHUNKS.html file. Only commands from sections that are found in the *build_db.json* file are copied into the build scripts.

A basic entry representing a source package in the *build_db.json* file looks like this:

```json
"ch-tools-m4": {
    "package_base": "m4",
    "part": ["tool1"]
}
```
The lookup key is the chapter/section id as used in the html "NOCHUNK" version of the LFS book.

*package_base*: Base name of the package without version number.

*part*: Part of the LFS book where this section belongs to. Recognized values are: tool1, tool2, system.

Sections in the book which contain build commands, but are not related to a source package should use "nonpackage" as value of the *package_base* field.

```json
"ch-tools-kernfs": {
    "package_base": "nonpackage",
    "part": ["tool1"]
}
```
A *newfile* field with a value of *true* forces the lfsbuild script to close the currently generated build file and open a new one. Commands of this and the following sections are written to the new file.

Whenever the lfsbuild script encounters a call to *bash* or *logout* within the section build commands, a new file is generated automatically, i.e. the *newfile* field is implied in these cases.

```json
"ch-tools-changingowner": {
    "package_base": "nonpackage",
    "part": ["tool1"],
    "newfile": true
}
```
The *changdir* field allows to adapt to sections where additional build directories are used. The value of this field is the number of directory levels that are created beneath the actual package directory level, e.g. 1 means, that the build commands in this section create one additional directory level below the package directory.
```json
"ch-tools-binutils-pass1": {
    "package_base": "binutils",
    "changedir": 1,
    "part": ["tool1"]
}
```

The *append* field allows to append commands to a section.
```json
"ch-tools-util-linux": {
    "package_base": "util",
    "part": ["tool2"],
    "append": [
        "rm -rf /usr/share/{info,man,doc}/*",
        "find /usr/{lib,libexec} -name \\*.la -delete"
    ]
}
```

With the *replace* field, commands can be removed or modified. The value of a replacement entry is an array where the first element is the pattern that is going to be replace if found, and the second entry is the replacement string. If the replacement string contains %s, the value of the variable with the name given as third array element is substituted in the replacement.
```json
 "ch-system-glibc": {
    "package_base": "glibc",
    "replace": {
        "0": ["make localedata/install-locales", ""],
        "1": ["tzselect", ""],
        "2": ["zic -d $ZONEINFO -p America/New_York", "zic -d $ZONEINFO -p %s", "timezone"],
        "3": ["ln -sfv /usr/share/zoneinfo/\n /etc/localtime", "ln -sfv /usr/share/zoneinfo/%s /etc/localtime", "timezone"]
}
```
The *message* field allows to echo messages commands after section commands section.
```json
"ch-system-shadow": {
        "package_base": "shadow",
        "part": ["system"],
        "replace": {
            "0": ["sed -i 's:DICTPATH.*:DICTPATH\\t/lib/cracklib/pw_dict:' etc/login.defs", ""],
            "1": ["passwd root", ""]
        },
        "message": [
            "-> Now issue: passwd root",
            "-> Then enter root password and continue with next build script"
        ]
    }
```

## Building the LFS system
To actually build the LFS system, run the generated shell scripts in the sequence given by the "_part\<n>" suffix from within the *source* directory, e.g. like so:
```sh
time bash lfs_build_part1.sh 2>&1 | tee lfs_build_part1.log
```

Depending on the included sections the build scripts need to be run as user *lfs* or *root*. Please refer to the LFS book for details.
