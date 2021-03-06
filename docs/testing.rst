*****************
Testing Cranelift
*****************

Cranelift is tested at multiple levels of abstraction and integration. When
possible, Rust unit tests are used to verify single functions and types. When
testing the interaction between compiler passes, file-level tests are
appropriate.

The top-level shell script :file:`test-all.sh` runs all of the tests in the
Cranelift repository.

Rust tests
==========

.. highlight:: rust

Rust and Cargo have good support for testing. Cranelift uses unit tests, doc
tests, and integration tests where appropriate.

Unit tests
----------

Unit test live in a ``tests`` sub-module of the code they are testing::

    pub fn add(x: u32, y: u32) -> u32 {
        x + y
    }

    #[cfg(test)]
    mod tests {
        use super::add;

        #[test]
        check_add() {
            assert_eq!(add(2, 2), 4);
        }
    }

Since sub-modules have access to non-public items in a Rust module, unit tests
can be used to test module-internal functions and types too.

Doc tests
---------

Documentation comments can contain code snippets which are also compiled and
tested::

    //! The `Flags` struct is immutable once it has been created. A `Builder` instance is used to
    //! create it.
    //!
    //! # Example
    //! ```
    //! use cranelift_codegen::settings::{self, Configurable};
    //!
    //! let mut b = settings::builder();
    //! b.set("opt_level", "fastest");
    //!
    //! let f = settings::Flags::new(&b);
    //! assert_eq!(f.opt_level(), settings::OptLevel::Fastest);
    //! ```

These tests are useful for demonstrating how to use an API, and running them
regularly makes sure that they stay up to date. Documentation tests are not
appropriate for lots of assertions; use unit tests for that.

Integration tests
-----------------

Integration tests are Rust source files that are compiled and linked
individually. They are used to exercise the external API of the crates under
test.

These tests are usually found in the :file:`tests` top-level directory where
they have access to all the crates in the Cranelift repository. The
:file:`lib/codegen` and :file:`lib/reader` crates have no external
dependencies, which can make testing tedious. Integration tests that don't need
to depend on other crates can be placed in :file:`lib/codegen/tests` and
:file:`lib/reader/tests`.

File tests
==========

.. highlight:: clif

Compilers work with large data structures representing programs, and it quickly
gets unwieldy to generate test data programmatically. File-level tests make it
easier to provide substantial input functions for the compiler tests.

File tests are :file:`*.clif` files in the :file:`filetests/` directory
hierarchy. Each file has a header describing what to test followed by a number
of input functions in the :doc:`Cranelift textual intermediate representation
<langref>`:

.. productionlist::
    test_file     : test_header `function_list`
    test_header   : test_commands (`isa_specs` | `settings`)
    test_commands : test_command { test_command }
    test_command  : "test" test_name { option } "\n"

The available test commands are described below.

Many test commands only make sense in the context of a target instruction set
architecture. These tests require one or more ISA specifications in the test
header:

.. productionlist::
    isa_specs     : { [`settings`] isa_spec }
    isa_spec      : "isa" isa_name { `option` } "\n"

The options given on the ``isa`` line modify the ISA-specific settings defined in
:file:`lib/codegen/meta-python/isa/*/settings.py`.

All types of tests allow shared Cranelift settings to be modified:

.. productionlist::
    settings      : { setting }
    setting       : "set" { option } "\n"
    option        : flag | setting "=" value

The shared settings available for all target ISAs are defined in
:file:`lib/codegen/meta-python/base/settings.py`.

The ``set`` lines apply settings cumulatively::

    test legalizer
    set opt_level=best
    set is_pic=1
    isa riscv64
    set is_pic=0
    isa riscv32 supports_m=false

    function %foo() {}

This example will run the legalizer test twice. Both runs will have
``opt_level=best``, but they will have different ``is_pic`` settings. The 32-bit
run will also have the RISC-V specific flag ``supports_m`` disabled.

The filetests are run automatically as part of `cargo test`, and they can
also be run manually with the `clif-util test` command.

Filecheck
---------

Many of the test commands described below use *filecheck* to verify their
output. Filecheck is a Rust implementation of the LLVM tool of the same name.
See the `documentation <https://docs.rs/filecheck/>`_ for details of its syntax.

Comments in :file:`.clif` files are associated with the entity they follow.
This typically means an instruction or the whole function. Those tests that
use filecheck will extract comments associated with each function (or its
entities) and scan them for filecheck directives. The test output for each
function is then matched against the filecheck directives for that function.

Comments appearing before the first function in a file apply to every function.
This is useful for defining common regular expression variables with the
``regex:`` directive, for example.

Note that LLVM's file tests don't separate filecheck directives by their
associated function. It verifies the concatenated output against all filecheck
directives in the test file. LLVM's :command:`FileCheck` command has a
``CHECK-LABEL:`` directive to help separate the output from different functions.
Cranelift's tests don't need this.

`test cat`
----------

This is one of the simplest file tests, used for testing the conversion to and
from textual IR. The ``test cat`` command simply parses each function and
converts it back to text again. The text of each function is then matched
against the associated filecheck directives.

Example::

    function %r1() -> i32, f32 {
    ebb1:
        v10 = iconst.i32 3
        v20 = f32const 0.0
        return v10, v20
    }
    ; sameln: function %r1() -> i32, f32 {
    ; nextln: ebb0:
    ; nextln:     v10 = iconst.i32 3
    ; nextln:     v20 = f32const 0.0
    ; nextln:     return v10, v20
    ; nextln: }

`test verifier`
---------------

Run each function through the IR verifier and check that it produces the
expected error messages.

Expected error messages are indicated with an ``error:`` directive *on the
instruction that produces the verifier error*. Both the error message and
reported location of the error is verified::

    test verifier

    function %test(i32) {
        ebb0(v0: i32):
            jump ebb1       ; error: terminator
            return
    }

This example test passes if the verifier fails with an error message containing
the sub-string ``"terminator"`` *and* the error is reported for the ``jump``
instruction.

If a function contains no ``error:`` annotations, the test passes if the
function verifies correctly.

`test print-cfg`
----------------

Print the control flow graph of each function as a Graphviz graph, and run
filecheck over the result. See also the :command:`clif-util print-cfg`
command::

    ; For testing cfg generation. This code is nonsense.
    test print-cfg
    test verifier

    function %nonsense(i32, i32) -> f32 {
    ; check: digraph %nonsense {
    ; regex: I=\binst\d+\b
    ; check: label="{ebb0 | <$(BRZ=$I)>brz ebb2 | <$(JUMP=$I)>jump ebb1}"]

    ebb0(v0: i32, v1: i32):
        brz v1, ebb2            ; unordered: ebb0:$BRZ -> ebb2
        v2 = iconst.i32 0
        jump ebb1(v2)           ; unordered: ebb0:$JUMP -> ebb1

    ebb1(v5: i32):
        return v0

    ebb2:
        v100 = f32const 0.0
        return v100
    }

`test domtree`
--------------

Compute the dominator tree of each function and validate it against the
``dominates:`` annotations::

    test domtree

    function %test(i32) {
        ebb0(v0: i32):
            jump ebb1     ; dominates: ebb1
        ebb1:
            brz v0, ebb3  ; dominates: ebb3
            jump ebb2     ; dominates: ebb2
        ebb2:
            jump ebb3
        ebb3:
            return
    }

Every reachable extended basic block except for the entry block has an
*immediate dominator* which is a jump or branch instruction. This test passes
if the ``dominates:`` annotations on the immediate dominator instructions are
both correct and complete.

This test also sends the computed CFG post-order through filecheck.

`test legalizer`
----------------

Legalize each function for the specified target ISA and run the resulting
function through filecheck. This test command can be used to validate the
encodings selected for legal instructions as well as the instruction
transformations performed by the legalizer.

`test regalloc`
---------------

Test the register allocator.

First, each function is legalized for the specified target ISA. This is
required for register allocation since the instruction encodings provide
register class constraints to the register allocator.

Second, the register allocator is run on the function, inserting spill code and
assigning registers and stack slots to all values.

The resulting function is then run through filecheck.

`test binemit`
--------------

Test the emission of binary machine code.

The functions must contains instructions that are annotated with both encodings
and value locations (registers or stack slots). For instructions that are
annotated with a `bin:` directive, the emitted hexadecimal machine code for
that instruction is compared to the directive::

    test binemit
    isa riscv

    function %int32() {
    ebb0:
        [-,%x5]             v0 = iconst.i32 1
        [-,%x6]             v1 = iconst.i32 2
        [R#0c,%x7]          v10 = iadd v0, v1       ; bin: 006283b3
        [R#200c,%x8]        v11 = isub v0, v1       ; bin: 40628433
        return
    }

If any instructions are unencoded (indicated with a `[-]` encoding field), they
will be encoded using the same mechanism as the legalizer uses. However,
illegal instructions for the ISA won't be expanded into other instruction
sequences. Instead the test will fail.

Value locations must be present if they are required to compute the binary
bits. Missing value locations will cause the test to crash.

`test simple-gvn`
-----------------

Test the simple GVN pass.

The simple GVN pass is run on each function, and then results are run
through filecheck.

`test licm`
-----------------

Test the LICM pass.

The LICM pass is run on each function, and then results are run
through filecheck.

`test dce`
-----------------

Test the DCE pass.

The DCE pass is run on each function, and then results are run
through filecheck.

`test shrink`
-----------------

Test the instruction shrinking pass.

The shrink pass is run on each function, and then results are run
through filecheck.

`test preopt`
-----------------

Test the preopt pass.

The preopt pass is run on each function, and then results are run
through filecheck.

`test postopt`
-----------------

Test the postopt pass.

The postopt pass is run on each function, and then results are run
through filecheck.

`test compile`
--------------

Test the whole code generation pipeline.

Each function is passed through the full ``Context::compile()`` function
which is normally used to compile code. This type of test often depends
on assertions or verifier errors, but it is also possible to use
filecheck directives which will be matched against the final form of the
Cranelift IR right before binary machine code emission.
