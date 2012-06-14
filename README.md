How To Retarget the GNU Toolchain in 21 Patches
===============================================

The text below originally came from a series of blog posts I wrote
about porting the GNU Toolchain to a new target.  The original blog
site is down, so I have collected the contents of the articles and
have posted them here, along with the patches.

[This is not complete yet]

Patch 1: Naming the Target
-------------------------

The GNU tools are maintained in two separate repositories: src and
gcc. The src tree contains the GNU binutils, gdb, instruction set
simulators, cgen, newlib (a C runtime library), winsup (cygwin runtime
support) and more. The gcc tree contains the GNU compiler collection
and related utilities. These trees are designed to be merged into a
single tree (sometimes called a Cygnus tree) so you can configure and
build the entire toolchain in one go.

Those of you interested in some history might want to read the mostly
accurate article and thread here:
http://www.sourceware.org/ml/gdb/2000-09/msg00009.html.

I'm not going to build everything from a merged tree. The unfortunate
truth is that shared top level files in the two trees often get out of
sync with one another. For instance, sometimes the trees depend on
incompatible versions of the autotools and keeping everything happy
together is more work that I care to do at this point.

Now that we've decided on two trees, I'm going to start in the src
tree.

The only thing we need to decide at this point is the architecture's
name. I'm calling this "ggx" for now.

The patch after the jump adds the top level configury for our port. It
just tells the build system to recognize our target name, and to not
configure/build any of the subdirectories requiring target specific
work.

Once patched, you should be able to configure and build the toolchain
like so:

    $ ./configure --target=ggx-elf
    $ make

The "ggx-elf" target simply tells the build system that we want a
toolchain to generate ELF object files for the ggx architecture.

The only thing this builds are some support libraries for the host
system. It's not much, but it's a start!


Patch 2: BFD!
-------------

Today we'll build BFD, which is a library for reading and writing
object files. We're mostly concerned with ELF, but BFD handles a
number of other formats as well.

BFD is a backronym for Binary File Descriptor, but we all know it
originally stood for Big F'ing Deal. The BFD manual puts it thusly:

    The name came from a conversation David Wallace was having with
    Richard Stallman about the library: RMS said that it would be
    quite hard--David said "BFD". Stallman was right, but the name
    stuck.

David Henkel-Wallace (aka Gumby) was one of Cygnus' founders, and, at
least as I recall, his California license plate at one time was "GNU
BFD".

Now we have to make some real decisions about the ggx architecture,
but not very difficult ones. The most significant features we're
committing to in this patch are 32-bit words, 32-bit addresses and
8-bit bytes (see cpu-ggx.c). Two reasons for picking these values: (1)
the GNU tools are really good at targeting 32-bit systems and (2) 8-
and 16-bit systems won't run any interesting Free Software.

Also note that we're defining the ELF machine number for ggx (0xFEED
in include/elf/common.h). This number is encoded in all ggx object
files and executables so other tools (objdump, gdb, etc) can identify
them as such. There's a standards body somewhere that maintains the
master list of ELF machine numbers. I just picked a random number
right now, but it's something you need to worry about if you start
working with other tool vendors for your processor (JTAG hardware
debuggers, for instance).

Apply this patch to your src tree, rebuild, and you'll have a
libbfd. There's really not much to it. Most of this patch is made up
of configury changes. Click on the link below to see the patch.

We're very close to having a working assembler for ggx. There's just
one more infrastructure step to take care of first.


Patch 3: Bad Instructions
-------------------------

Much like yesterday's BFD patch, the patch below is mostly
configury. It builds the opcodes library for our new target: the ggx
machine.

The opcodes library describes how instructions are encoded in memory
and how to disassemble them into text. It's used by tools like
objdump, gdb and gas.

And, finally, details of our future architecture are beginning to
emerge. This patch defines the first ggx instruction: "bad". This
instruction represents an illegal opcode, and should issue something
like an illegal instruction trap if we ever attempt to execute it.

There's no real reason to ever write a program with a "bad"
instruction, but there's no avoiding defining it either. We need well
defined behaviour when the core attempts to decode an undefined
instruction. In truth, there are many "bad" instruction variants: one
for each undefined opcode. It just so happens that they are all "bad"
right now!

We haven't written anything yet about how opcodes are encoded, or even
how wide the instructions are. If you look carefully at the
disassembler in ggx-dis.c you'll see that we're just pulling single
byte "instructions" out of memory and looking them up in an opcode
table. This is just scaffolding and will be replaced with a real
instruction decoder shortly.

Before we do that, we'll take our first major step tomorrow by
creating a working assembler.


Patch 4: Cooking with GAS
-------------------------

Today's patch to the src tree will let us build the GNU Assembler,
gas, for the ggx architecture. You may recall from yesterday's post
that we've defined our first and only instruction: "bad". This
assembler will let us create our first object file of bad code.

Two routines of note are md_begin(), which populates a hashtable with
all of our opcodes (only one so far), and md_assemble(), which parses
ggx assembly text to convert into binary.

The only thing you need to worry about in md_assemble() are your
machine's instructions. The rest of gas will handle things like
labels, data and pseudo-ops. Continuing with our temporary
scaffolding, gas is using a single byte encoding for our "bad"
instruction. We'll get around to defining a proper instruction
encoding in a couple of days.

As an example, here's some ggx assembly source code, which we'll
assemble and examine with nm.

    $ cat foo.s
    .data
                    .global foo
    foo:            .long 0x123
    bar:            .long 0x456
    .text
                    .global main
    main:           bad
                    bad
                    bad
    
    $ ggx-elf-as foo.s
    $ nm a.out
    00000004 d bar
    00000000 D foo
    00000000 T main

My x86 Fedora nm doesn't know about the ggx architecture, but it does
understand ELF so it's able to examine the symbols and dump them
correctly. The host's objdump is similarly able to dump section
information but, of course, it can't disassemble ggx instructions. We
need a ggx-elf-objdump for that, and that's what we'll build tomorrow.


Patch 5: binutils
-----------------

Today's tiny patch simply turns on the binutils directory in our src
tree. This gives us the following fully functional tools:
ggx-elf-addr2line, ggx-elf-ar, ggx-elf-as, ggx-elf-c++filt,
ggx-elf-nm, ggx-elf-objcopy, ggx-elf-objdump, ggx-elf-ranlib,
ggx-elf-readelf, ggx-elf-size, ggx-elf-strings and ggx-elf-strip.

Now we can disassemble the code we assembled yesterday:

    $ ggx-elf-objdump -sd a.out
    
    a.out:     file format elf32-ggx
    
    Contents of section .text:
     0000 000000                               ...             
    Contents of section .data:
     0000 00000123 00000456                    ...#...V       
    Disassembly of section .text:
    
    00000000 <main>:
       0:   00              bad
            ...

Tomorrow we'll create a linker. We still only support one instruction
("bad"), but all of the toolchain infrastructure is starting to take
shape.


Patch 6: The Linker
-------------------

Today's patch enables the GNU linker, ld.

The only architectural element we're committing to in this patch is
the basic memory architecture. We'll be using a standard von Neumann
architecture with shared text and data memory. There are some GNU
tools ports to Harvard architectures, with their separate text and
data memories, but they have to play tricks in order to deal with
limitations of the tools. GDB, for instance, was never designed to
deal with situations where the instruction at address 0x1000 would be
different from the data at 0x1000. Our primary design goal is to make
the tools simple, which means we'll be going with the basic von
Neumann.

This patch includes a simple linker script template that tells the
linker where to place sections in memory. The linker script also
defines a few important symbols for things like the base of the C
runtime stack and the location and extent of the .bss section. These
will be important when we write our startup code and have to
initialize the stack pointer and clear .bss.

So now we have an assembler, linker and binutils for the ggx
architecture, but we've only defined a few things about it:

- 32-bit words
- 8-bit bytes
- 32-bit addresses
- von Neumann architecture

We've also defined a single instruction with a bogus single byte
encoding. Tomorrow we'll talk about instruction encodings and
implement some basic support for encoding and decoding instructions.


Patch 7: Instruction Encodings
------------------------------

This is where things get interesting....

What kind of machine do we want? Here's what I have in mind:

1. Load-store architecture. Operations will either work on registers
and immediates or perform loads/stores to memory. This is simple and
GCC is well suited for this kind of architecture.

2. Variable width instructions. We're not building a high-performance
RISC engine. We're just building an architecture that is simple to
implement good GCC support for. Let's ignore any non-essential
complexity and allow for variable width-instructions. So, for
instance, loading a 32-bit immediate value into a register should just
be a single instruction (> 32-bits).

3. If we're going to go with variable width instructions, let's make
code-density a secondary goal. Given this, let's default to 16-bit
instructions as the smallest instruction.

Designing instruction encodings is tricky business. I fully expect
this initial attempt to change once we see the kind of output GCC
gives us. But, for now, here's a comment from the patch describing my
first stab at two classes of instruction encodings..
 
    /*
      The ggx processor's 16-bit instructions come in two forms:
    
      FORM 1 instructions start with a 0 bit...
    
        0ooooooaaabbbccc
        0              F
     
       oooooo - FORM 1 opcode number
       aaa    - operand A
       bbb    - operand B
       ccc    - operand C
    
      FORM 2 instructions start with a 1 bit...
    
        1ooovvvvvvvvvvvv
        0              F
     
       ooo          - FORM 2 opcode number
       vvvvvvvvvvvv - 12-bit immediate value
    */
    
Allowing for three operands lets us perform operations like A = B + C
as one instruction (where reg ister A != register B != register
C). This limits us to 3-bit operands if we want 6-bits for the
opcode. Three bit operands naturally allows for 8 general purpose
registers. This puts the ggx architecture into the "register poor"
bucket. However, it's a start, and perhaps it will change as we evolve
the architecture.

At this point I don't know if the FORM 2 instruction makes sense. Just
think of it as a place holder for now. It demonstrates one way to
divide our 16-bit instruction space into different groups of
instruction encodings.

The following patch to the src tree modifies the opcodes library to
understand FORM 1 and FORM 2 instructions. ggx_opc_info_t is a new
type describing each instruction. We also define a few macros
describing instruction types. For instance, GGX_F1_AB is a FORM 1
instruction that only uses operands A and B, and GGX_F1_NARG is a FORM
1 instruction that uses no operands. Classifying instructions using
these values will simplify disassembly. The print_insn_ggx() function
now includes support for disassembling generic GGX_F1_NARG
instructions (of which "bad" is now a member). We'll add more
instructions types as our port develops.

When I started this series I mentioned that we would only add
instructions as required by the compiler. However, that's not quite
true. We're going to add two basic instruction prior to building the
compiler so we can assemble and run our first program on a
simulator. Our first good (or, at least, non-"bad") instruction will
be "load immediate". We'll do that tomorrow.

