# RISC OS Filesystem In C example

This repository is a template for writing a RISC OS filing system module in C.
It provides the FileSwitch interfacing layer - the register-packing veneers and
the `fsentry_*` dispatch functions for `Open`, `GetBytes`, `PutBytes`, `Args`,
`Close`, `File`, `Func` and `GBPB` - so that a developer can concentrate on the
storage-specific logic of their own filing system rather than on the mechanics
of talking to FileSwitch.

The module builds as `DummyFS`: a skeleton filing system that answers every
FileSwitch entry point but implements none of the actual storage behaviour
(each case returns "does nothing"). It is a starting point to build a real
filing system from, not a working filing system in its own right.

Beneath the entry points the template is split into three layers - the
FileSwitch layer (`s/`, `c/_*`, `c/module`), a data layer (`c/fsdata`) that
understands the medium's structure, and a provider layer (`c/provider`) that
reads the medium - so that a filing system built from it can be moved between
storage backends, and tested without hardware, by swapping the provider alone.

## Origin

The code is a modernised version of the original Acorn "FS-in-C" example,
extracted from the Acorn NFS implementation and targeted at version 3 of the
Acorn DDE. It has been reformatted and updated to build with more modern
tooling:

* CMunge is used in place of CMHG.
* The interfacing veneers have been updated towards 32-bit compatibility.
* The build runs automatically in CI (see `.github/workflows/ci.yml`), which
  builds the module through [build.riscos.online](https://build.riscos.online).

## Repository layout

* `c/` - the C source for each `fsentry_*` entry point (`_args`, `_close`,
  `_file`, `_func`, `_gbpb`, `_getbytes`, `_open`, `_putbytes`), the module
  wrapper (`module`), shared static data (`statics`), the data layer
  (`fsdata`), the production provider skeleton (`provider`) and the
  in-memory test provider used only by the test build (`testprov`).
* `h/` - headers: the entry-point parameter structures (`fsentries`), the
  register-packing veneer declarations (`veneers`, `interface`), shared
  constants and information-word bits (`consts`), the provider interface
  (`provider`), the data-layer interface and descriptor types (`fsdata`),
  module-wide declarations (`dummyfs`, `modhead`) and statics (`statics`).
* `tests/` - `TestFS`, a BASIC program that loads the built modules and
  drives FileSwitch at them, checking that every operation reaches the
  filing system and that the no-medium state is reported cleanly.
* `s/` - the assembler interface layer that packs FileSwitch's register-based
  calling convention into the C parameter structures used by `h/fsentries`.
* `cmhg/` - the CMunge module header (`modhead`), describing the module's
  SWIs, commands and services to the build tools.
* `prminxml/fsinc.xml` - the "Making a Filing System for RISC OS in C" guide,
  written in PRM-in-XML and cross-linked to the relevant sections of the RISC
  OS Programmer's Reference Manual chapters on writing a filing system
  ([writefs.html](http://www.riscos.com/support/developers/prm/writefs.html),
  [writefsnew.html](http://www.riscos.com/support/developers/prm/writefsnew.html)).
  Build it to HTML with `riscos-prminxml -f html5+xml -O output prminxml/fsinc.xml`.

## Building

The module builds with `amu` in a RISC OS build environment:

```
riscos-amu -f Makefile,fe1
```

A test variant, `rm32/DummyFSTest`, which links the in-memory test provider
in place of the production one, is built with:

```
riscos-amu -f Makefile,fe1 ram TEST=1
```

## Testing

With both variants built, run the tests from the repository root on a RISC OS
that can register filing systems (Pyromaniac allows this by default):

```
riscos-run --command "Run tests.TestFS"
```

Each check prints `PASS` or `FAIL`, and the program ends with an error if any
check failed. The production module has no medium and reports "No medium
present" for every operation; the test module carries a medium and reports
"does nothing", which shows that the operation reached the entry point.

CI builds both variants and runs the tests automatically on every push, using
the same environment via `build.riscos.online`; see `.robuild.yaml` and
`.github/workflows/ci.yml` for the exact steps.
CI also renders `prminxml/fsinc.xml` to HTML and packages it as a
`Documentation-<version>.zip` archive, using the same versioning as the module
build. Both archives are uploaded as workflow artifacts on every push, and are
attached to the GitHub release created when a `v*` tag is pushed.

## Using this template

To build a real filing system from this template, work through
`prminxml/fsinc.xml` alongside the `c/_*` sources: each entry point's
`switch` statement has one case per FileSwitch reason code, documented with
the register/field usage FileSwitch expects on entry and exit, a `See:`
link to the matching section of the original PRM chapter, and a note of how
a read-only filing system should answer it. Replace `c/provider` with code
that reaches your storage, fill in `c/fsdata` with the code that reads your
medium's structures, and then replace the `err_DoesNothing` bodies with real
behaviour, one reason code at a time, extending `tests/TestFS` and the test
image in `c/testprov` as you go.

The filing system number, SWI chunk and error base in the template are
examples only; obtain your own allocations before releasing a filing system
built from it.
