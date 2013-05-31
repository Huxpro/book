# The Compilation Pipeline

The process of converting OCaml source code to executable binaries is done in
multiple steps.  Every stage generally checks and discards information from the
source code, until the final output is untyped and low-level assembly code.

Each of the compilation steps can be executed manually if you need to inspect
something to hunt down a bug or performance regression.  It's even possible to
compile OCaml to run efficiently on environments such as Javascript or the Java
Virtual Machine.

In this chapter, you'll learn:

* the compilation pipeline and what each stage represents.
* source preprocessing via `camlp4` and the intermediate forms.
* the bytecode `ocamlc` compiler and `ocamlrun` interpreter.
* the native code `ocamlopt` code generator.

The OCaml toolchain accepts textual source code as input.  Each source file is
a separate *compilation unit* that is compiled separately, and finally linked
together into an executable or library.  The compilation pipeline looks like
this:

```
    Source code
        |
        | parsing and preprocessing
        v
    Parsetree (untyped AST)
        |
        | syntax extensions
        v
    Camlp4 transformation (untyped AST)
        |
        | type inference and checking
        v
    Typedtree (type-annotated AST)
        |
        | pattern-matching compilation
        | elimination of modules and classes
        v
     Lambda
      /  \
     /    \ closure conversion, inlining, uncurrying,
    v      \  data representation strategy
 Bytecode   \
             \
            Cmm
             |
             | code generation
             v
        Assembly code
```

We'll now go through these stages and explain how the tools behind them
operate.

## Parsing and preprocessing with `camlp4`

The first thing the compiler does is to parse the input source code into
a more structured data type.  This immediately eliminates code which doesn't
match basic syntactic requirements.  The OCaml lexer and parser use the same
basic techniques described earlier in [xref](#parsing-with-ocamllex-and-menhir).

One powerful feature present in OCaml is the facility to dynamically extend the
syntax via the `camlp4` tool.  The compiler usually lexes the source code into
tokens, and then parses these into an Abstract Syntax Tree (AST) that represents the
parsed source code.

Camlp4 modules can extend the lexer with new keywords, and later transform these
keywords (or indeed, any portion of the input program) into conventional OCaml
code that can be understood by the rest of the compiler.  We've already seen
several examples of using `camlp4` within Core:

* **Fieldslib** to generates first-class values that represent fields of
  a record in [xref](#records).
* **Sexplib** to convert types to s-expressions in [xref](#data-serialization-with-s-expressions)
* **Bin_prot**: for efficient binary conversion in [xref](#fast-binary-serialization).

These all use a common `camlp4` library called `type_conv` to provide a common
extension point.  Type_conv defines a new keyword `with` that can appear after
a type definition, and passes on the type declaration to extensions.  The
type_conv extensions all generate boiler-plate code based on the type you
defined.  This approach avoids the inevitable performance hit of doing this
work dynamically, but also doesn't require a complex Just-In-Time (JIT) runtime
that is a source of unpredictable dynamic behaviour.

All `camlp4` modules accept an input AST and output a modified one.  This lets
you inspect the results of transformations at the source code level manually to
see exactly what's going on.  Let's look at a simple Core extension called
`pa_compare` for how to do this.

### Example: the `pa_compare` syntax transformer

OCaml provides a polymorphic comparison operator that inspects the runtime
representation of two values to see if they are equal.  As we noted in
[xref](#maps-and-hashtables), this is not as efficient or as safe as defining
explicit comparison functions between values.

The `pa_compare` syntax extension takes care of this boilerplate code
generation via `camlp4`. Try it out from `utop`:

```ocaml
# #require "comparelib.syntax" ;;

# type t = { foo: string; bar : t } ;;
type t = { foo : string; bar : t; }

# type t = { foo: string; bar: t } with compare ;;
type t = { foo : string; bar : t; }
val compare : t -> t -> int = <fun>
val compare_t : t -> t -> int = <fun>
```

The first type definition of `t` is a standard OCaml phrase and results in the
expected output.  The second one includes the `with compare` directive.  This
is intercepted by `comparelib` and turned into two new functions that are generated
from the type into the `compare` and `compare_t` functions.  How do we see what
these functions actually do?  You can't do this from `utop` directly, since it
embeds the `camlp4` compilation as an automated part of its operation.

Let's turn to the command-line to inspect the result of the `comparelib`
transformation instead.  Create a file that contains the type declaration from earlier:

```ocaml
(* comparelib_test.ml *)
type t = { foo: string; bar: t } with compare
```

Now create a shell script to run the `camlp4` tool manually.

```bash
#!/bin/sh
# camlp4_dump

OCAMLFIND="ocamlfind query -predicates syntax,preprocessor -r"
INCLUDE=`$OCAMLFIND -i-format comparelib.syntax`
ARCHIVES=`$OCAMLFIND -a-format comparelib.syntax`
camlp4o -printer o $INCLUDE $ARCHIVES $1
```

This shell script uses the `ocamlfind` package manager to list the include and
library paths required by the `comparelib` syntax extension.  The final
command invokes the `camlp4o` preprocessor directly and outputs the
resulting AST to standard output as textual source code.

```console
$ sh camlp4_dump comparelib_test.ml
type t = { foo : string; bar : t }

let _ = fun (_ : t) -> ()
  
let rec compare : t -> t -> int =
  fun a__001_ b__002_ ->
    if Pervasives.( == ) a__001_ b__002_
    then 0
    else
      (let ret =
         (Pervasives.compare : string -> string -> int) a__001_.foo
           b__002_.foo
       in
         if Pervasives.( <> ) ret 0
         then ret
         else compare a__001_.bar b__002_.bar)
  
let _ = compare
let compare_t = compare
let _ = compare_t
```

The result is the original type definition, and some automatically generated
code that implements an explicit comparison function for each field in the
record.  This generated code is then compiled as if you had typed it in
yourself.

Another useful feature of `type_conv` is that it can generate signatures too.
Copy the earlier type definition into a `comparelib_test.mli` and rerun the
camlp4 dumper script.

```console
$ ./camlp4_dump.sh test_comparelib.mli 
type t = { foo : string; bar : t }

val compare : t -> t -> int
```

The external signature generated by `comparelib` is much simpler than the
actual code.  Running `camlp4` directly on the original source code lets you
see these all these transformations precisely.

<note>
<title>Don't overdo the syntax extensions</title>

Syntax extensions are a very powerful extension mechanism that can completely
change your source code's layout and style.  Core includes a very conservative
set of extensions that minimise the syntax changes.  There are a number of
third-party libraries that perform much more wide-sweeping changes, such as
introducing whitespace-sensitive indentation or even building entirely new
languages.

While it's tempting to compress all your boiler-plate code into `camlp4`
extensions, it can make production source code much harder for other people to
read and review.  Core mainly focuses on type-driven code generation using the
`type_conv` extension, and doesn't fundamentally change the OCaml syntax.

Another thing to consider before deploying your own syntax extension is
compatibility with other syntax extensions.  Two separate extensions create a
grammar clash can lead to hard-to-reproduce bugs. That's why most of Core's
syntax extensions go through `type_conv`, which acts as a single point for
extending the grammar via the `with` keyword.

</note>

## The type checking phase

After obtaining a valid parsed AST, the compiler must then check that
the code obeys the rules of the static type system.  At this stage,
code that is syntactically correct but misuses values is rejected with
an explanation of the problem.  We're not going to delve into the details
of how type-checking works here (the rest of the book covers that), but
rather how it fits in with the rest of the compilation process.

Assuming that the source code is validly typed, the original AST is transformed
into a typed AST. This has the same broad structure of the untyped AST,
but syntactic phrases are replaced with typed variants instead.

You can explore the type checking process very simply.  Create a file with a
single type definition and value.

```ocaml
(* typedef.ml *)
type t = Foo | Bar
let v = Foo
```

Now run the compiler on this file to *infer* a default type for the compilation
unit.  This will run the type checking process on the compilation unit you
specify.

```console
$ ocamlc -i typedef.ml
type t = Foo | Bar
val v : t
```

The type checker has run and provided a default signature for the module.  It's
often useful to redirect this output to an `mli` file to give you a starting
signature to edit the external interface, without having to type it all in by
hand.

### Using `ocamlobjinfo` to inspect compilation units

Recall from [xref](#files-modules-and-programs) that `mli` files are optional.
If you don't supply an explicit signature, then the inferred output from the
module implementation is used instead.  The compiled version of the signature
is stored in a filename ending with `cmi`.

```console
$ ocamlc -c typedef.ml
$ ocamlobjinfo typedef.cmi
File typedef.cmi
Unit name: Typedef
Interfaces imported:
	559f8708a08ddf66822f08be4e9c3372	Typedef
	65014ccc4d9329a2666360e6af2d7352	Pervasives
```

The `ocamlobjinfo` command examines the compiled interface and displays what
other compilation units it depends on.  In this case, we don't use any external
modules other than `Pervasives`.  Every module depends on `Pervasives` by
default, unless you use the `-nopervasives` flag (this is an advanced use-case,
and you shouldn't need it in normal use).

The long alphanumeric identifier beside each module name is a hash calculated
from all the types and values exported from that compilation unit.  This is
used during type-checking and linking to check that all of the compilation
units have been compiled consistently against each other.  A difference in the
hashes means that a compilation unit with the same module name may have
conflicting type signatures in different modules.  This violates static type
safety if these conflicting instances are ever linked together, as every
well-typed value has precisely one type in OCaml.

This hash check ensures that separate compilation remains type safe all the way
up to the final link phase.  Each source file is type checked separately, but
the hash of the signature of any external modules is recorded with the compiled
signature.

### Examining the typed syntax tree

The compiler has a couple of advanced flags that can dump the raw output of the
internal AST representation.  You can't depend on these flags to give the same
output across compiler revisions, but they are a useful learning tool.

First, let's look at the untyped AST from our `typedef.ml`.

```console
$ ocamlc -dparsetree typedef.ml
[
  structure_item (typedef.ml[1,0+0]..[1,0+18])
    Pstr_type [
      "t" (typedef.ml[1,0+5]..[1,0+6])
        type_declaration (typedef.ml[1,0+5]..[1,0+18])
          ptype_params = []
          ptype_cstrs = []
          ptype_kind =
            Ptype_variant
              [
                (typedef.ml[1,0+9]..[1,0+12])
                  "Foo" (typedef.ml[1,0+9]..[1,0+12])
                  [] None
                (typedef.ml[1,0+15]..[1,0+18])
                  "Bar" (typedef.ml[1,0+15]..[1,0+18])
                  [] None
              ]
          ptype_private = Public
          ptype_manifest = None
    ]
  structure_item (typedef.ml[2,19+0]..[2,19+11])
    Pstr_value Nonrec [
      <def>
        pattern (typedef.ml[2,19+4]..[2,19+5])
          Ppat_var "v" (typedef.ml[2,19+4]..[2,19+5])
        expression (typedef.ml[2,19+8]..[2,19+11])
          Pexp_construct "Foo" (typedef.ml[2,19+8]..[2,19+11])
          None false
    ]
]
```

This is rather a lot of output for a simple two-line program, but also reveals
a lot about how the compiler works.  Each portion of the tree is decorated with
the precise location information (including the filename and character location
of the token).  This code hasn't been type checked yet, and so the raw tokens
are all included.  After type checking, the structure is much simpler.

```console
$ ocamlc -dtypedtree typedef.m
[
  structure_item (typedef.ml[1,0+0]..typedef.ml[1,0+18])
    Pstr_type [
      t/1008
        type_declaration (typedef.ml[1,0+5]..typedef.ml[1,0+18])
          ptype_params = []
          ptype_cstrs = []
          ptype_kind =
            Ptype_variant
              [
                "Foo/1009" []
                "Bar/1010" []
              ]
          ptype_private = Public
          ptype_manifest = None
    ]
  structure_item (typedef.ml[2,19+0]..typedef.ml[2,19+11])
    Pstr_value Nonrec [
      <def>
        pattern (typedef.ml[2,19+4]..typedef.ml[2,19+5])
          Ppat_var "v/1011"
        expression (typedef.ml[2,19+8]..typedef.ml[2,19+11])
          Pexp_construct "Foo" [] false
    ]
]
```

The typed AST is more explicit than the untyped syntax tree.  For instance, the
type declaration has been given a unique name (`t/1008`), as has the `v` value
(`v/1011`).

You'll never need to use this information in day-to-day development, but it's
always instructive to examine how the type checker folds in the source code
into a more compact form like this.

### The untyped lambda form

Once OCaml gets past the typed AST, it eliminates all the static type
information into a simpler intermediate *lambda form*.  This discards all the
modules and objects, replacing them direct references to values such as records
and function pointers instead.  Pattern matches are compiled into automata that
are highly optimized for the particular type involved.

It's possible to examine all this behaviour via another intermediate output
from the compiler.  Create a new `pattern.ml` file alongside the previous
`typedef.ml`.

```ocaml
(* pattern.ml *)
open Typedef
let _ =
  match v with
  | Foo -> "foo"
  | Bar -> "bar"
```

The lambda form is the first representation that discards the OCaml type
information and begins to look like the runtime memory model from
[xref](#memory-representation-of-values), and should be quite familiar to Lisp
aficionados.  To see it for `pattern.ml`, compile as usual but add the
`-dlambda` directive.

```console
$ ocamlc -dlambda -c pattern.ml 
(setglobal Pattern!
  (seq
    (let (match/1008 (field 0 (global Typedef!)))
      (if (!= match/1008 0) "bar" "foo"))
    (makeblock 0)))
```

It's not important to understand every detail of this internal form, but
some interesting points are:

* There are no mention of modules or types any more, and have been replaced by
  references to global values instead.
* The pattern match has turned into an integer comparison by checking 
  the header tag of `v`.  Recall that variants without parameters are stored 
  in memory as integers in the order which they appear.  The pattern matching 
  engine knows this, and has transformed the pattern into a single integer
  comparison.  If `v` has a tag of `0`, the function returns `"foo"`, and otherwise
  returns `"bar"`.
* Values are addressed directly by an index and `field` (`v` got assigned to `1008`
  during type checking).  The type safety checks earlier have ensured that these
  fields will always exist, and so the lambda form doesn't do any dynamic checks.
  However, unwise use of unsafe language features such as `Obj.magic` module can
  easily induce crashes at this level.

The lambda form is primarily a stepping-stone to the bytecode engine that we
cover next.  However, it's often easier to look at the textual output here than
wade through native assembly code from compiled executables.

### Bytecode with `ocamlc` and `ocamlrun`

After the lambda form has been generated, we are very close to having
executable code.  The OCaml tool-chain branches into two separate compilers at
this point.  We'll describe the the `ocamlc` bytecode compiler first, which consists
of two pieces:

* `ocamlc` compiles files into a simple bytecode that is a close mapping to the lambda form.
* `ocamlrun` is a portable interpreter that executes the bytecode.

`ocamlrun` is an interpreter that uses the OCaml stack and an accumulator to
store values, and only has seven registers in total (the program counter, stack
pointer, accumulator, exception and argument pointers, and environment and
global data).  It implements around 140 opcodes which form the OCaml program.
The full details of the opcodes are available
[online](http://cadmium.x9c.fr/distrib/caml-instructions.pdf).

The big advantage of using `ocamlc` is simplicity, portability and compilation
speed.  The mapping from the lambda form to bytecode is straightforward, and
this results in predictable (but slow) execution speed.

### Compiling and linking OCaml bytecode

`ocamlc` compiles individual `ml` files into bytecode `cmo` files.  These are
linked together with the OCaml standard library to produce an executable
program.  The order in which `.cmo` arguments are presented on the command line
defines the order in which compilation units are initialized at runtime (recall
that OCaml has no single `main` function like C does). 

A typical OCaml library consists of multiple modules (and hence multiple `cmo`
files).  `ocamlc` can combine these into a single `cma` bytecode archive by
using the `-a` flag. The objects in the library are linked as regular `cmo`
files in the order specified when the library file was built.  However, if an
object file within the library isn't referenced elsewhere in the program, it is
not linked unless the `-linkall` flag forces it to be included.  This behaviour
is analogous to how C handles object files and archives (`.o` and `.a`
respectively).

The bytecode runtime comprises three main parts: the bytecode interpreter,
garbage collector, and a set of C functions that implement the primitive
operations.  Bytecode instructions are provided to call these C functions.  The
OCaml linker produces bytecode for the standard runtime system by default, and
any custom C functions in your code (e.g. from C bindings) require the runtime
to dynamically load a shared library.

This can be specified as follows:

```console
$ ocamlc -a -o mylib.cma a.cmo b.cmo -dllib -lmylib
```

The `dllib` flag embeds the arguments in the archive file.  Any subsequent
packages including this archive will also include the extra linking directive.
This lets the `ocamlrun` runtime locate the extra symbols when it executes the
bytecode.

You can also generate a complete standalone executable that bundles the `ocamlrun`
interpreter with the bytecode in a single binary.  This is known as the "custom"
runtime mode, and can be run by:

```console
$ ocamlc -a -o mylib.cma -custom a.cmo b.cmo -cclib -lmylib
```

The custom mode is the most similar to native code compilation, as both
generate standalone executables.  There are quite a few other options available
for compiling bytecode (notably with shared libraries or building custom
runtimes).  Full details can be found in the
[manual](http://caml.inria.fr/pub/docs/manual-ocaml/manual022.html).

#### Embedding OCaml bytecode

A consequence of using the bytecode compiler is that the final link phase must
be performed by `ocamlc`.  However, you might sometimes want to embed your OCaml
code inside an existing C application.  OCaml also supports this mode of operation
via the `-output-obj` directive.

This mode causes `ocamlc` to output a C object file that containing the
bytecode for the OCaml part of the program, as well as a `caml_startup`
function.  The object file can then be linked with other C code using the
standard C compiler, or even turned in a standalone C shared library.

The bytecode runtime library is installed as `libcamlrun` in the standard OCaml
directory (obtained by `ocamlc -where`).  Creating an executable just requires
you to link the runtime library with the bytecode object file.  Here's a quick
example to show how it all fits together.

Create two OCaml source files that contain a single print line.

```console
$ cat embed_me1.ml 
let () = print_endline "hello embedded world 1"
$ cat embed_me2.ml 
let () = print_endline "hello embedded world 2"
```

Next, create a C file which will be your main entry point.

```c
/* main.c */
#include <stdio.h>
#include <caml/alloc.h>
#include <caml/mlvalues.h>
#include <caml/memory.h>
#include <caml/callback.h>

int 
main (int argc, char **argv)
{
  puts("Before calling OCaml");
  caml_startup (argv);
  puts("After calling OCaml");
  return 0;
}
```

Now compile the OCaml files into a standalone object file.

```console
$ ocamlc -output-obj -o embed_out.o embed_me1.ml embed_me2.ml
```

After this point, you no longer need the OCaml compiler, as `embed_out.o` has
all of the OCaml code compiled and linked into a single object file.  Compile
an output binary using gcc to test this out.

```console
$ gcc -Wall -I `ocamlc -where` -L `ocamlc -where` -lcamlrun -ltermcap \
  -o final_out embed_out.o main.c
$ ./final_out 
Before calling OCaml
hello embedded world 1
hello embedded world 2
After calling OCaml
```

Once inconvenience with `gcc` is that you need to specify the location
of the OCaml library directory.  The OCaml compiler can actually handle C
object and source files, and it adds the `-I` and `-L` directives for you.  You
can compile the previous object file with `ocamlc` to try this.

```console
$ ocamlc -o final_out2 embed_out.o main.c
$ ./final_out2
Before calling OCaml
hello embedded world 1
hello embedded world 2
After calling OCaml
```

You can also verify the system commands that `ocamlc` is invoking by adding
`-verbose` to the command line.  You can even obtain the source code to
the `-output-obj` result by specifying a `.c` output file extension instead
of the `.o` we used earlier.

```console
$ ocamlc -output-obj -o embed_out.c embed_me1.ml embed_me2.ml
$ cat embed_out.c
```

Embedding OCaml code like this lets you write OCaml code that interfaces with
any environment that works with a C compiler.   You can even cross back from the
C code into OCaml by using the `Callback` module to register named entry points
in the OCaml code.  This is explained in detail in the
[interfacing with C](http://caml.inria.fr/pub/docs/manual-ocaml/manual033.html#toc149)
section of the OCaml manual.

## Native code generation

The native code compiler is ultimately the tool that production OCaml code goes
through.  It compiles the lambda form into fast native code executables, with
cross-module inlining and code optimization passes that the bytecode
interpreter doesn't perform.  However, care is taken to ensure compatibility
with the bytecode runtime, and the same code should run identically when
compiled with either toolchain.

The `ocamlopt` command is the frontend to the native code compiler, and has a
very similar interface to `ocamlc`.  It also accepts `ml` and `mli` files, but
compiles them to:

* A `.o` file containing native object code.
* A `.cmx` file containing extra information for linking and cross-module optimization.
* A `.cmi` compiled interface file that is the same as the bytecode compiler.

When the compiler links modules together into an executable, it uses the
contents of the `cmx` files to perform cross-module inlining across compilation
units.  This can be a significant speedup for standard library functions that
are frequently used outside of their module.

Collections of `.cmx` and `.o` files can also be be linked into a `.cmxa`
archive by passing the `-a` flag to the compiler.  However, unlike the bytecode
version, you must keep the individual `cmx` files in the compiler search path
so that they are available for cross-module inlining.  If you don't do this,
the compilation will still succeed, but you will have missed out on an
important optimization and have slower binaries.

### Building debuggable libraries

The native code compiler builds executables that can be debugged using
conventional system debuggers such as GNU `gdb`.  You'll need to compile your
libraries with the `-g` option to add the debug information to the output, just
as you need to with C compilers.

TODO add example of gdb breakpoint use

#### Profiling native code libraries

TODOs

#### Embedding native code in libraries

The native code compiler also supports `output-obj` operation just like the
bytecode compiler.  The native code runtime is called `libasmrun.a`, and should
be linked instead of the bytecode `libcamlrun.a`.

Try this out using the same files from the bytecode embedding example earlier.

```console
$ ocamlopt -output-obj -o embed_native.o embed_me1.ml embed_me2.ml
$ gcc -Wall -I `ocamlc -where` -L `ocamlc -where` -lasmrun -ltermcap \
  -o final_out_native embed_native.o main.c
./final_out_native
Before calling OCaml
hello embedded world 1
hello embedded world 2
After calling OCaml
```

The `embed_native.o` is a standalone object file that has no further references
to OCaml code beyond the runtime library, just as with the bytecode runtime.

<tip>
<title>Activating the debug runtime</title>

Despite your best efforts, it is easy to introduce a bug into C bindings that
cause heap invariants to be violated.  OCaml includes a variant of the runtime
library `libasmrun.a` that is compiled with debugging symbols.  This is
available as `libasmrund.a` and includes extra memory integrity checks upon
every garbage collection cycle.  Running these often will abort the program
near the point of corruption and helps isolate the bug.

To use this debug library, just link with `-runtime-variant d` set:

```
$ ocamlopt -runtime-variant d -verbose -o hello hello.ml hello_stubs.c
$ ./hello 
### OCaml runtime: debug mode ###
Initial minor heap size: 2048k bytes
Initial major heap size: 992k bytes
Initial space overhead: 80%
Initial max overhead: 500%
Initial heap increment: 992k bytes
Initial allocation policy: 0
Hello OCaml World!
```

If you get an error that `libasmrund.a` is not found, then this is probably
because you're using OCaml 4.00 and not 4.01.  It's only installed by default
in the very latest version, which you should be using via the `4.01.0dev+trunk`
OPAM switch.

</tip>
