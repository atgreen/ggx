How To Retarget the GNU Toolchain in 21 Patches
===============================================

Premable to the github Edition
------------------------------

The text below originally came from a series of daily blog posts I
wrote about porting the GNU Toolchain to a new target.  The original
blog site is down, so I have collected the contents of the articles
and have posted them here along with all source patches.  In
retrospect, some of the original thinking behind this project seems
naiive, however the end result was an interesting and useful project
that I hope others may learn from.

The source patches apply to 2008-era GCC and SRC trees for the GNU
Toolchain.  They may not be useful from a development perspective
today, but as a teaching tool they still have value.

The experimental 'ggx' target was eventually renamed to 'moxie', and
work continues here:
* http://moxielogic.org/blog
* http://moxielogic.org/wiki
* http://github.com/atgreen/moxiedev

I'd love to hear from people who found this useful or who have
questions.  I can be reached at green@moxielogic.com.

Happy Hacking!

**Anthony Green**

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Preamble: Top-Down ISA Evolution
---------------------------------

The design process of an Instruction Set Architecture (ISA) has always
struck me as being backwards. I imagine that they are mostly designed
by hardware engineers whose decisions are largely influenced by the
medium in which they work - hardware description languages. Every
decision is influenced to some degree by implementation difficulty or
limitations of the hardware.

Only the lucky compiler developers have a chance to review an ISA
before it is fixed in stone. The few times I've seen this happen it
has always been to everyone's benefit. Perhaps there are instructions
that the compiler could never use, or perhaps the compiler would
generate better code if only it had some overlooked addressing mode.

I've often wondered what would happen if we reversed this
process. What would an ISA look like that was solely influenced by the
needs of the compiler writer, without regard to the hardware side of
the house? Instead of designing from the hardware up, start from the
compiler and design top-down.

As an experiment, I've developed the start of a GNU tools port to a
new architecture that was wholly undefined at the start of the
project. There was no instruction set architecture definition, no
register file definition, no defined ABI or OS interfaces. Instead, I
defined these things on the fly, letting things evolve naturally based
on the needs of the compiler. The C compiler, assembler, linker,
binutils and simulator were developed concurrently. So, for instance,
when the compiler required an "add" instruction, each tool would be
taught about "add" and so on.

I'm going to blog a patch a day to show my progress. Each patch will
be small and buildable. I hope to show that you can get from nothing
to running real programs using this top-down ISA design method in a
surprisingly short amount of time.

A couple of caveats first... I am not an experienced compiler
writer. I've been involved in many new GNU ports over the years at
Cygnus and Red Hat, but mostly in a bizdev capacity. I have a good
idea how all the pieces go together, but this will be my first attempt
at it and I'm certain to botch some things. Second, I'm by no means an
expert in ISA design. I'm taking the lazy man's route to ISA design:

1. Try to build something with gcc
2. See what the compiler is complaining about
3. Implement it
4. Go to 1.

Every design decision is driven by whatever is easiest to
implement. What I expect I'll end up with is a simple although wildly
inefficient architecture. Then perhaps we can optimize this toy ISA
based on real-world code generation. At the end of the day, however,
my goal is for this to be a fun and interesting experiment.

I'll post my first patch later today. 

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 


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

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-01-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 2: BFD!
-------------

Today we'll build BFD, which is a library for reading and writing
object files. We're mostly concerned with ELF, but BFD handles a
number of other formats as well.

BFD is a backronym for Binary File Descriptor, but we all know it
originally stood for Big F'ing Deal. The BFD manual puts it thusly:

> The name came from a conversation David Wallace was having with Richard Stallman about the library: RMS said that it would be quite hard--David said "BFD". Stallman was right, but the name stuck.

David Henkel-Wallace (aka Gumby) was one of Cygnus' founders, and, at
least as I recall, his California license plate at one time was "GNU
BFD".

Now we have to make some real decisions about the ggx architecture,
but not very difficult ones. The most significant features we're
committing to in this patch are 32-bit words, 32-bit addresses and
8-bit bytes (see cpu-ggx.c). Two reasons for picking these values: 

1. the GNU tools are really good at targeting 32-bit systems and 
2. 8- and 16-bit systems won't run any interesting Free Software.

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

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-03-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

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

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-04-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

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

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-05-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

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

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-05-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

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

* 32-bit words
* 8-bit bytes
* 32-bit addresses
* von Neumann architecture

We've also defined a single instruction with a bogus single byte
encoding. Tomorrow we'll talk about instruction encodings and
implement some basic support for encoding and decoding instructions.

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-06-src.patch


- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

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

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-07-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 8: A First Real Instruction
---------------------------------

Yesterday we decided that the ggx core will have eight general purpose
registers. Let's name them $r0 to $r7.

It's time to add our first real instruction. We're going to implement
a "Load Immediate" instruction. This will extract an integer value
from the instruction stream and load it into a register. We'll use
syntax like this:

    ldi.l $r1, 0x555

The ".l" suffix means "long word" (4 bytes). So "ldi.l" is "Load
Immediate Long Word", and "ldi.l $r1, 0x555" will put the 4-byte value
0x555 into register $r1.

We're going to encode the "0x555" part in the instruction stream
immediately after the 16-bit instruction, so "ldi.l $r1, 0x555" is
really a 48-bit instruction.

Now we need to implement our first fixup. A fixup is a placeholder in
the instruction stream that gets populated with a value at the tail
end of assembly. In this case we need a fixup to tell the assembler
how to encode our 0x555 immediate value in the instruction
stream. Some fixups can't be resolved at assembly time, so the the
assembler turns them into "relocations" (or relocs).

Consider, for instance, the following instruction:

    _start:     ldi.l $r1, _start+0x10

We don't know where _start will end up in memory until link time, so
the assembler generates a reloc that gets computed at link time.

We don't need to worry too much about how this all works. However, we
do need to describe the basic relocs required by our port. In this
case, we simply need a 32-bit reloc that we'll call R_GGX_DIR32. This
patch contains all of the bfd changes necessary to implement 32-bit
relocations. Eventually we'll need a different one to fill the 12-bit
value part of FORM 2 instructions.

We also define a new instruction type, GGX_F1_A4, which is FORM 1
instruction using operand A followed by a 4-byte value. We teach
opcodes how to disassemble GGX_F1_A4 instructions, and we teach the
assembler how to parse and encode them.

Finally, we update our opcode table to include "ldi.l", defining it as
a GGX_F1_A4 instruction.

Here are our main tools in action: the assembler, linker and objdump:


    $ cat foo.s
            .global _start
    .text
    _start: ldi.l $r1, 0x555
            ldi.l $r2, 0x111+0x222
            ldi.l $r3, _start+0x10
    $ ggx-elf-as -o foo.o foo.s && ggx-elf-ld -o foo foo.o
    $ ggx-elf-objdump -d foo
    
    foo:     file format elf32-ggx
    
    Disassembly of section .text:
    
    00001054 <_start>:
        1054:       00 40 00 00     ldi.l   $r1, 0x555
        1058:       05 55
        105a:       00 80 00 00     ldi.l   $r2, 0x333
        105e:       03 33
        1060:       00 c0 00 00     ldi.l   $r3, 0x1064
        1064:       10 64 

Fantastico! Tomorrow we'll add one more unsolicited instruction, and
then we'll build our simulator...

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-08-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 9: Move It
----------------

We're adding one more instruction before we move on to the simulator:
register-to-register move.

    mov $rA, $rB

This simply copies the contents of register B into register A. The
patch would be tiny except that this is our first GGX_F1_AB
instruction (a FORM 1 instruction using only two operands) and we need
to tell opcodes and gas how to disassemble/assemble them.

The next two days are going to be really cool. Tomorrow we'll be
running code on a simulator, and then we'll start building our C
compiler!

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-09-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 10: Time to Simulate
--------------------------

There are many useful and interesting simulator infrastructures out
there. However, in the interest of simplicity, we'll use the sim
infrastructure associated with gdb in the src tree.

Today's patch provides a minimal simulator for ggx. Once applied, we
can start running code like so...


    $ cat foo.s
    .section .bss
            .long   0x00
            .global _start
    .text
    _start: ldi.l $r1, 0x555
            ldi.l $r2, 0x111+0x222
            ldi.l $r3, _start+0x10
            mov $r4, $r3
            bad
    $ ggx-elf-as -o foo.o foo.s && ggx-elf-ld -o foo foo.o
    $ ggx-elf-run -t foo
    # set pc to 0x1074
    # 0x00001074: $r1 = 0x555
    # 0x0000107a: $r2 = 0x333
    # 0x00001080: $r3 = 0x1084
    # 0x00001086: $r4 = $r3 (0x1084)
    program stopped with signal 4.

I had to manually create a .bss section because the simulator
currently requires one. Also note how execution was terminated. Since
we're simulating a processor and not just running a program, there's
no real way to end the simulation. I solved this problem by ending the
program with a "bad" instruction. The simulator currently turns that
into a simulated SIGILL and halts execution. Finally, the "-t" option
turns on instruction tracing, otherwise there would be no output.

The simulator itself is a very simple interpreter. It consists largely
of a loop within which we switch on the next instruction opcode. To
add a new instruction we simply add a new case to the switch
statement. The src/sim infrastructure handles pretty much everything
else.

This simulator uses gdb's standard interfaces, so it should link
cleanly into a future ggx-elf-gdb port. It should also be able to
integrate it into the board-level simulator sid should that be
desired.

This is the largest patch of the bunch, mostly because there's a lot
of basic infrastructure we need to implement in order to get a working
sim.

To recap, now we have gas, ld, binutils and a simulator for the ggx
architecture which currently only has two real instructions (load
immediate and register-to-register move). Tomorrow we'll have our
first go at a C compiler. This is when things really start evolving
faster.

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-10-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 11: The Start of a C Compiler
-----------------------------------

We're building our compiler today...

First, check out the GCC tree:
$ svn -q checkout svn://gcc.gnu.org/svn/gcc/trunk gcc
Apply the attached patch, then configure and build like so...

    $ configure --target=ggx-elf --languages=c

This tells the build system to generate a C compiler for the ggx-elf
architecture.

Running "make" will build the compiler driver (gcc) and C compiler
(cc1) but fail during the libgcc build. libgcc is a runtime support
library and contains things like floating point emulation routines for
systems that don't have hardware floating point support.

The plan now is to get basic functionality working in cc1 manually,
then we'll build libgcc, then newlib and libgloss.

Here are some things we're changing in our architecture today...

We're renaming registers $r0..$r7 to $fp, $sp, $r0..$r5. $fp will be
the frame pointer, $sp will be the stack pointer, and $r0..$r5 remain
general purpose registers (src tree patch to follow). If you look
carefully, you'll see that I've also defined a $pc register, which is
the program counter. It's not a real register, but defining one for
the purposes of the compiler makes certain things easier.

We've also defined additional features of the ABI: 32-bit ints and
longs, 16-bit short, 8-bit char, chars are signed by default, 32-bit
floats, 64-bit doubles and long doubles, and 64-bit long longs. These
are typical settings for 32-bit word systems.

This patch also makes some initial decisions about the calling
convention. We don't have an over-abundance of registers, so for now
we're only going to pass the first two 32-bits of function arguments
in registers $r1 and $r2 and the rest will go on the stack. Integral
values smaller than 32-bits will be promoted to 32-bits. Scalar return
values will go in $r0 (and $r1 if required).

If you look at the patch, you'll see that a number of target macros
are set to abort(). My focus was to get to a cc1 that builds as
quickly as possible. We can fix up these aborting macros with real
definitions as required.

The compiler also provides a couple of aborting instruction
patterns. Even the most basic source file won't compile without these
definitions. However, we don't really need them for code generation
yet which is why they're aborting.

But now, finally.. 

    $ cat foo.c
    int myint;
    short myshort;
    double mydouble;
    $ ./cc1 -quiet foo.c
    $ ggx-elf-as foo.s -o foo.o && ggx-elf-nm foo.o
    00000008 C mydouble
    00000004 C myint
    00000002 C myshort
 
That's about all it will do for now. Anything else aborts during
compilation. The plan now is to try fix things up as we attempt
compile increasingly complex code.

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-11-gcc.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 12: Building and Running our First C Program
---------------------------------------------------

It's time to run our first program! Here's the C code: 

    void _start()
    {
      /* Initialize the stack and frame pointers. */
      asm ("ldi.l  $sp, 0x30000");
      asm ("ldi.l  $fp, 0x0");
    
      /* Call main()  */
      main();
    
      /* Terminate execution.  */
      asm ("bad");
    }
    
    int main()
    {
      return 0x555;
    }
    
    int x; /* because the sim requires a .bss section. */

We have no C runtime library support yet, so we have to do things
manually in our own hand-coded _start function. This would normally
initialize the stack and frame pointers, and then clear .bss before
calling main(). Unfortunately we don't have any instructions for
writing to memory yet, so we won't be clearing .bss. There's a "bad"
instruction after the call to main() in order to terminate the
simulator with SIGILL. I think there's a cleaner way to terminate
simulation, but it will have to wait 'til we're linking with libgloss.

This code requires two new ggx instructions: jsra and ret. jsra is
"Jump to SubRoutine at Absolute address" and ret is a "RETurn from
subroutine" instruction. jsra is a new instruction type:
GGX_F1_4. This is a FORM 1 instruction with no register operands
followed by a 4-byte value (the target function address). jsra
performs the following actions:


    push the return address (next instruction) on the stack
    push $fp on the stack
    set $pc to target address

ret does the opposite...

    pop the old frame pointer into $fp
    pop the return address into $pc

Pretty simple! Keep in mind, however, that this isn't nearly enough
support for proper function calls. We still need to save callee saved
registers on the stack and allocate space for local variables. Let's
do that later when we actually have instructions for writing to
memory. We'll just avoid using local variables for now.

    $ ./cc1 -quiet foo.c
    $ ggx-elf-as -o foo.o foo.s
    $ ggx-elf-ld -o foo foo.o
    $ ggx-elf-objdump -d foo
    foo:     file format elf32-ggx
    
    Disassembly of section .text:
    
    00001074 <_start>:
        1074:       00 40 00 03     ldi.l   $sp, 0x30000
        1078:       00 00
        107a:       00 00 00 00     ldi.l   $fp, 0x0
        107e:       00 00
        1080:       04 00 00 00     jsra    0x108a
        1084:       10 8a
        1086:       08 00           bad
        1088:       06 00           ret
    
    0000108a <main>:
        108a:       00 80 00 00     ldi.l   $r0, 0x555
        108e:       05 55
        1090:       06 00           ret
    $ ggx-elf-run -t foo
    # set pc to 0x1074
    # 0x00001074: $sp = 0x30000
    # 0x0000107a: $fp = 0x0
    # 0x00001080: jumping to subroutine 0x108a
    # 0x0000108a: $r0 = 0x555
    # 0x00001090: returning to 0x1086
    program stopped with signal 4.

See how we're returning 0x555 by loading it into $r0?

There are two patches today. The src tree patch does some register
renaming, and adds jsra and ret instruction support. The second patch
updates some of the target macros and adds a couple of instruction
patterns in the machine description for calls and returns.

Today's patch is available in the ggx patch archive.

More fun tomorrow....

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-12-src.patch
* http://github.com/atgreen/ggx/blob/master/ggx-12-gcc.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 13: Function Prologues and Epilogues
------------------------------------------

Today's patch introduces a first go at proper function prologues and
epilogues. The function prologue is responsible for saving
callee-saved registers and allocating stack space for local
variables. The function epilogue is responsible for restoring
callee-saved registers and executing a ret instruction.

I found this to be the toughest thing to get right so far. The trick I
learned is that you have to tell the compiler about registers that
don't really exist. In this case, we tell the compiler about an
argument pointer register and a second frame pointer. The ggx port
calls these "?ap" and "?fp". I prefixed them with ? because they won't
really exist in the architecture and we'll never generate code with
them. The compiler assumes that these are real registers until close
to the end of compilation when it has to replace ?fp and ?ap
references with $fp and $sp by computing the difference between
them. This description really doesn't do it justice. Just believe me
when I say it was tricky to get right.

We're introducing three new instructions in this patch: push, pop and
add.l. They look like this:

    push  $sp, $r2
    pop   $sp, $r2
    add.l $r1, $r2, $r3

The push instruction in this example will decrement $sp by four and
store $r2 at memory location $sp. (Note that the stack is growing
downwards, just like on most ports).

The pop instruction here will load $r2 with the 4-byte value stored at address $sp, and increment $sp by four.

And, finally, add.l adds the contents of $r2 to $r3 and saves the answer in $r1.

We use push and pop to save and restore callee-saved registers. The
add.l instruction was introduced to perform stack adjustments in the
prologue/epilogue.

We'll end up with push/pop pair for each callee-saved register. I
think we can do better some time down the road by using a single
instruction to push/pop all callee-saved registers in one go. The way
we do this by encoding a bitmask of the registers we need to save in
the push instruction (and similarly in the pop instruction). Let's
save that trick of later.

Now, given this C code:

    int foo(int, int);
    int main()
    {
      return foo (111, 222);
    }
    
We get:

    main:
            push   $sp, $r1
            ldi.l  $r0, 111
            ldi.l  $r1, 222
            jsra   foo
            ldi.l  $r5, -4
            add.l  $sp, $fp, $r5
            pop    $sp, $r1
            ret
    
You'll notice that we're saving and restoring the caller's $r1, but
not $r0. This is because the compiler knows that $r0 may be used to
return values and shouldn't assume it will survive function
calls. (Note: just as I was writing this, I realized that 64-bit
integrals may be returned in $r0 and $r1, so $r1 shouldn't be
callee-saved either. I'll fix this tomorrow.)

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-13-src.patch
* http://github.com/atgreen/ggx/blob/master/ggx-13-gcc.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 14: Loading and Storing
-----------------------------

It's time to implement loading and storing for our load-store
architecture! Today's patch introduces 4-byte memory loads and stores.

The compiler uses these to read and write global variables, as well as
to load and store local, temps and function arguments to and from the
stack.

We're introducing three new instructions in this patch: ld.l, lda.l,
st.l, and sta.l. They look like this:


    ld.l  $r1, ($r2)
    lda.l $r1, myvar
    st.l  ($r1), $r2
    sta.l myvar, $r2

The ld.l instruction loads a long word into $r1 from the memory
location stored in $r2. lda.l is a Load Absolute instruction, and
loads 4-bytes from the address "myvar" (the address is encoded in this
48-bit instruction). st.l and sta.l are the obvious "store" versions
of these instructions. Eventually we'll add .s and .b versions of
these instructions to load/store 16- and 8-bit values.

We had to add new instruction types to handle instructions with
indirect operands (in parenthesis). We also added our first custom
constraint ('W') in the compiler's machine description to implement
the simple patterns for indirect loads and stores. None of this was
very difficult.

This patch also includes a few little fixes. For instance, the jsra
instruction is now responsible for setting $fp to $sp and I fixed the
order in which we increment the stack pointer when pushing values in
jsra.

Now that we can load and store, we're able to compile and run programs
like this:


    int g;
    
    int add(int a, int b, int c, int d, int e, int f)
    {
      return a + b + c + d + e + f + g;
    }
    
    int main()
    {
      g = 7;
      return (add (1, 2, 3, 4, 5, 6));
    }

This is what the compiler now generates for main() (with annotations
by hand):

    main:
            # allocate stack space for outgoing function args
            ldi.l  $r5, -16
            add.l  $sp, $sp, $r5
            # store 7 in global variable 'g'
            ldi.l  $r0, 7
            sta.l  g, $r0
            # store arg 3 on stack
            ldi.l  $r0, 3
            st.l   ($sp), $r0
            # store arg 4 on stack
            ldi.l  $r0, 4
            add.l  $r1, $sp, $r0
            ldi.l  $r0, 4
            st.l   ($r1), $r0
            # store arg 5 on stack
            ldi.l  $r0, 8
            add.l  $r1, $sp, $r0
            ldi.l  $r0, 5
            st.l   ($r1), $r0
            # store arg 6 on stack
            ldi.l  $r0, 12
            add.l  $r1, $sp, $r0
            ldi.l  $r0, 6
            st.l   ($r1), $r0
            # load args 1 and 2 into registers
            ldi.l  $r0, 1
            ldi.l  $r1, 2
            # call the add() function and return the result
            jsra   add
            ret

While this certainly works (we can compile, assemble, link and then
run on our simulator), it's far too long. Almost all of this code is
dedicated to storing operands 3 through 6 to the stack (1 and 2 are
passed in registers $r0 and $r1). We can do better by introducing
indirect offset addressing. That's what we'll do tomorrow.

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-14-src.patch
* http://github.com/atgreen/ggx/blob/master/ggx-14-gcc.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 15: A Simpler Way to Load and Store
-----------------------------------------

Yesterday's patch introduced basic load and store instructions, but
with only two addressing modes: absolute and indirect. Storing a value
8-bytes into the stack looked like this:

    ldi.l  $r0, 8           # load offset into $r0
    add.l  $r1, $sp, $r0    # compute address
    ldi.l  $r0, 5           # load the value we want to save
    st.l   ($r1), $r0       # store it

We're going to introduce a new addressing mode to simplify this, so the code now looks like this:

    ldi.l  $r0, 5
    sto.l  8($sp), $r0

This makes "st.l ($sp), $r0" equivalent to "sto.l 0($sp), $r0".

My plan was to only add instructions as required by the compiler. I'm
going against plan here, as we don't absolutely need to add this new
instruction. However, I was having a tough time following all of the
loads and adds while debugging code generation and this makes the code
much more readable. I'll try not to do these kinds of premature
optimizations going forward, as I want to optimize the ISA and
encodings based on real-world code generation. You'll just have to
forgive me this one indulgence for now.

I'd also like to point out that I'm using different instruction names
for different addressing modes: st for store indirect, sta for store
absolute, and sto for store offset (similarly for ld, lda and
ldo). This is not strictly necessary. We should be able to overload
the name and distinguish between sto, st and sta based on instruction
operands. However, gas doesn't provide much infrastructure for fancy
instruction parsing, so I decided to go the easy way and give these
instructions different names.

Here's a side-by-side comparison with code we generated yesterday for
main()


    main:                           main:
            ldi.l  $r5, -16                 ldi.l  $r5, -16
            add.l  $sp, $sp, $r5            add.l  $sp, $sp, $r5
            ldi.l  $r0, 7                   ldi.l  $r0, 7
            sta.l  g, $r0                   sta.l  g, $r0
            ldi.l  $r0, 3                   ldi.l  $r0, 3
            st.l   ($sp), $r0               st.l   $(sp), $r0
            ldi.l  $r0, 4
            add.l  $r1, $sp, $r0
            ldi.l  $r0, 4                   ldi.l  $r0, 4
            st.l   ($r1), $r0               sto    4($sp), $r0
            ldi.l  $r0, 8
            add.l  $r1, $sp, $r0                   
            ldi.l  $r0, 5                   ldi.l  $r0, 5
            st.l   ($r1), $r0               sto.l   8($sp), $r0
            ldi.l  $r0, 12
            add.l  $r1, $sp, $r0
            ldi.l  $r0, 6                   ldi.l  $r0, 6
            st.l   ($r1), $r0               sto.l  12($sp), $r0
            ldi.l  $r0, 1                   ldi.l  $(r0), 1
            ldi.l  $r1, 2                   ldi.l  $(r1), 2
            jsra   add                      jsra   add
            ret                             ret

The new code, on the right, only saves us 6 instructions. It may not
seem like much, but it's enough to make a really big difference when
you're tracing through simulator output trying to debug code
generation errors.

We've got a small handful of instructions now, but only for
straight-line code. Tomorrow we'll add compare and branch
instructions, and soon we'll be building libgcc: GCC's runtime support
library.

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-15-src.patch
* http://github.com/atgreen/ggx/blob/master/ggx-15-gcc.patch


- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 16: if... then... else...
-------------------------------

Architectures with no if-then-else capability are pretty boring, so
today we're adding compare and branch instructions.

In addition to the new instructions, we'll be adding a condition code
register. This register holds a series of bits that will be set by our
compare instruction. The bits currently are:

> 00001: Greater Than, Signed
> 00010: Less Than, Signed
> 00100: Equal
> 01000: Greater Than, Unsigned
> 10000: Less Than, Unsigned

We could also probably benefit from a "Zero" bit, and we may want to
squeeze arithmetic overlow in there at some point as well, but this
collection of 5 bits is certainly good enough for now.

We're not currently exposing this register to the programmer. It's
just a bit of internal global state that's implemented in the
simulator. This may also change at some point.

GCC supports 10 conditional branch patterns: beq, bne, blt, bgt, bltu,
bgtu, bge, ble, bgeu, bleu. They all do the obvious things. For
instance, bleu is Branch if Less or Equal (Unsigned). I've implemented
all 10 as instructions even though this isn't really necessary. For
instance, bgt is essentially ble after you swap the operands. I didn't
want to get too clever at this point, and I still have plenty of
opcode space. Let's just keep this in our back pocket if we start
feeling opcode space pressure.

You'll also note that I encoded all of these branch instructions as
GGX_F1_AB4 GGX_F1_4. This means they have two no operands and are
followed by a 4-byte value (the target address). This is pretty
wasteful. We'll likely end up with a shorter PC-relative branch
instruction, but I want to actually measure things before I added this
complexity (for instance, how many bits are required to represent X%
of all PC-relative branches?).

Here's some sample source and output:

    int lessthan (int x, int y)
    {
      return (x < y);
    }
    
    lessthan:
            ldi.l  $r5, 0
            cmp    $r1, $r0
            bge    .L2
            ldi.l  $r5, 1
    .L2:
            mov    $r0, $r5
            ret

This patch also adds jsr and jmp instructions. They both use register
operands containing the targe t address. Both were pretty straight
forward to implement and required to get to this next step...

Running "make -k" in the libgcc directory results in 116 object files!

There should be 143, so we're very close. Of the ones that are failing, they all die like so:
    ../../../gcc/libgcc/../gcc/unwind-c.c: In function ‘__gcc_personality_sj0’:
    ../../../gcc/libgcc/../gcc/unwind-c.c:59: internal compiler error: in emit_move_multi_word, at expr.c:3247

This error is indicative of missing "mov" patterns for smaller data
sizes (short words and bytes). This is what we'll add tomorrow. Once
we've built libgcc, we'll move on the newlib and libgloss. We're very
close to running a real "Hello World" app!

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-16-src.patch
* http://github.com/atgreen/ggx/blob/master/ggx-16-gcc.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 17: Bytes, Shorts, and Some Cleanup
-----------------------------------------

We hit an important milestone today!...

Yesterday we tried building libgcc after adding compare and branch
instructions. The compiler built a few files but eventually failed
because it didn't know how to load and store values smaller than
4-bytes. Today we'll introduce patches for dealing with these 16-bit
(.s) and 8-bit (.b) values.

The new instructions are directly analogous to their ".l"
counterparts: ldi.b, ld.b, lda.b, st.b, sta.b, ldi.s, ld.s, lda.s,
st.s, and sta.s.

Once we add these instructions almost all of libgcc builds. The
compiler just asks for one more instruction before it'll finish the
job: an unconditional jump to an address stored in a register
(jmp). Ok, done!

Finally, all 143 object files build to create libgcc.a! [sound of
champagne bottle popping open]

Despite hitting this milestone, today's patches are a little boring,
so I've included a second patch to GCC.

One of the problems with coding by example is that you have to know
which examples to pick in order to be successful. Today's second GCC
patch fixes a poor choice I made in yesterday's compare and branch
patch.

I had implemented compare/branch using "cc0", a magic condition code
register that only the compiler knows about. GCC has plenty of special
case logic to deal with this cc0 register, and the GCC hackers hate
it. The preferred way to deal with this is to create a normal register
to hold condition codes (ggx's is called "?cc"). It still only exists
in the mind of the compiler (we'll never generate code referencing
it), but I'm told it greatly simplifies things for the compiler
developers. Does it change how the compiler optimizes? My
understanding is that the underlying logic is more sane, but the
output should be (almost?) identical. (comments welcome if I have this
wrong) Despite the strong desire to rid GCC of cc0 ports, I think most
a few of them still use cc0. Apparently, de-cc0'ing a mature port is a
lot of work (another good reason to get this right from day one!), so
I don't know if cc0 will be officially deprecated any time soon.

The second change in this patch replaces my 10 branch patterns in the
machine description with a single pattern that uses an iterator to
produce the 10 different patterns. It doesn't change the compiler's
behaviour, but it does let us rip out a bunch of repetitive code.

(Thanks to Hans-Peter and Paolo for their feedback and advice on this
last patch)

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-18-src.patch
* http://github.com/atgreen/ggx/blob/master/ggx-18-gcc.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 19: Hello World!
----------------------

This is it. Today we're building and running a "Hello World" C app on
the ggx simulator.

Yesterday we identified nine missing instructions required to link
hello.c to newlib. They were all arithmetic and logical operators and
were simple to implement in all of the tools.

Today's patch also includes a software interrupt instruction
(swi). This is the instruction that ggx code will use to talk to the
simulator in order to interface with the outside world. Consider, for
instance, the primitive function "write"

    int write (int fd, const void *buf, size_t len);

We implement a system call in libgloss like so:

    /*
     * Input:
     * $r0      -- File descriptor.
     * $r1      -- String to be printed.
     * -8($fp)  -- Length of the string.
     *
     * Output:
     * $r0    -- Length written or -1.
     * errno  -- Set if an error
     */

    write:
         swi     SYS_write /* SYS_write is a constant 5 */
         ret

Then the simulator's handling of the swi instruction includes a switch on the interrupt number:

      case 0x5: /* SYS_write */
        {
            char *str = &memory[cpu.asregs.regs[3]];
            /* String length is at 0x8($fp) */
            unsigned count, len = EXTRACT_WORD(&memory[cpu.asregs.regs[0] + 8]);
            count = len;
            while (count-- > 0)
                putchar (*(str++));
            cpu.asregs.regs[2] = len;
         }

It's not a perfect implementation (all output goes to stdout, and
there's no error checking), but it works for now.

Today's patch and tests really put the toolchain through its paces, as
we'll be simulating thousands of instructions just for hello.c! Our
simulator runs to date have been limited to a dozen or so
instructions, so this is a big jump...

    $ cat hello.c
    #include <stdio.h>
    
    int main()
    {
      puts ("Hello World!");
      return 0;
    }
    $ ggx-elf-gcc -o hello hello.c
    $ ggx-elf-run hello
    Hello World
    $ ggx-elf-run -v hello 
    ggx-elf-run hello
    Hello World!
    
    
    # instructions executed        2704

2704 instructions is a lot of code for just Hello World. Consider,
however, that this includes all of the system initialization code,
such as initializing the heap, allocating IO buffers with malloc(),
etc. There's a lot that has to happen before we get to printing our
greeting.

To be honest, I skipped a step in there. It's the one where running
"hello" fails in many interesting ways and you have to debug simulator
traces. There's no avoiding this step. Just think of it as a rite of
passage.

In my case, I tracked it down to bad relocation generation in the
assembler and was able to fix it thanks again to #gcc IRC
folks. Today's patch includes this fix.

When I started this series I wrote that I would show how go from
nothing to running real programs on a new architecture by posting
daily patches. I think we're there, so I'm going to slow down the
blogging effort. But that's not to say that I'm done with ggx. There's
still plenty to do. Here are some ideas for work items: 

### gcc testsuite

GCC comes with a huge suite of tests to exercise code generation. The
next obvious step in the development of this toolchain is to start
whacking away at bugs identified by the testsuite. I've had a quick
crack at it, and a couple of obvious things need fixing. For instance,
there's an off-by-one error in ABI handling for vararg
functions. There's also no support for trampolines, which are needed
for GCC's nested functions extension. Apart from that, the results
look pretty good so far.  

### broader language support

Exception handling is big issue here. The compiler needs to be taught
how to deal with C++ and Java exceptions. libffi is also needed for
Java.  

### gdb support

This is a tricky one for ggx. If we were working from a pre-defined
ISA, then I would go straight to gdb next. Stepping through code with
gdb is much more productive than reading instruction traces from the
simulator. But we're not working from a pre-defined ISA with fixed
instruction encodings. We're just at a first draft of the ISA and plan
to make many changes.

Unfortunately, it looks like a lot of what goes into making gdb work
is hard coding recognition of instruction sequences for things like
like function prologues. I don't want to have to hack gdb every time I
tweak the ISA, so I'll leave this 'til much later in the game.  ISA
tweaking

This is where the game really begins. Check out this chart, which
shows the static frequency of instruction types used in our hello
application:

[chart appears to be lost. looking for copy]

The most frequently used instruction type is GGX_F1_A4, which is a
16-bit instruction with one operand followed by a 32-bit value. These
are all of the "load immediate" and "load absolute" instructions. What
we'll want to do is understand everything about these instructions:
how and why they're used, and if there's anything we can do to
eliminate them or encode them more efficiently. We already know
there's 6-bits of waste in the first 16-bits of GGX_F1_A4 because
we're not using operands B and C. Perhaps we can use those 6-bits to
hold a small constant value. That would turn some number of those
48-bit load-immediate instructions into 16-bit instructions. There
will be many opportunities for improvements like this, and it should
be interesting to see how dense we can make our code.  

### Run Linux!

Only half joking here. Truth be told, it builds, but we're a long way
from booting...

[chart appears to be lost. looking for copy]

It's interesting to see that the mix of instructions is completely
different in the kernel. The 4th and 5th most frequently used
instruction types are swapped. This demonstrates how it will be
important to have a good mix of programs at hand while we're tweaking
the ISA.

I hope some people have found this series interesting. I'm interested
in hearing feedback if you are so inspired.

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-19-src.patch
* http://github.com/atgreen/ggx/blob/master/ggx-19-gcc.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 20: The GCC Testsuite and Cooperative Multitasking
--------------------------------------------------------

There's good reason to implement support for the "write" system call
right away. It's the bare minimum required to run any of the
"c-torture/execute" tests. This is the group of tests in GCC's
extensive testsuite which require execution on the target platform.

I let the gcc execution tests run last night, and this is what I got:

                    === gcc Summary ===
    
    # of expected passes            11945
    # of unexpected failures        910
    # of unresolved testcases       129
     
Not bad for a first run! Lots of failing tests were timing out. I
think there's a bug in how we compile long long and long double
support which is creating an infinite loop in libgcc. Or it could be a
bug in the simulator. One never knows...

I did find a couple of simple sim bugs, which are fixed in today's
patch.

I also implemented setjmp and longjmp support in newlib, and then
tested with Cheap Threads.

Cheap Threads is a simple C library providing cooperative multitasking
using setjmp/longjmp.

This is first code I've downloaded from the net which built and ran
without a hitch:

    $ ggx-elf-gcc -DCT_RETURN -c *.c
    $ ggx-elf-gcc -o demo1 demo1.o cterror.o  ctsched.o ctassert.o ctalloc.o  ctmemory.o ctsubscr.o
    $ ggx-elf-run -v demo1
    ggx-elf-run demo1
    
    Cheap threads demo #1
    Hello from thread A
    This is coming from thread B
    Hello from thread A
    This is coming from thread B
    Hello from thread A
    This is coming from thread B
    Hello from thread A
    This is coming from thread B
    Hello from thread A
    This is coming from thread B
    This is coming from thread B
    This is coming from thread B
    This is coming from thread B
    
    Maximum memory pools registered: 0
    Total memory allocations: 2
    Maximum outstanding memory allocations: 2
    
    
    # instructions executed       44631

44k instructions! This is most we've run so far...

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-20-src.patch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

Patch 21: Trampolines, Benchmarks, and Rebasing Patches
-------------------------------------------------------

Somewhere along the line I botched one of my gcc patches and people
have been complaining that they don't all apply nicely. Well today I'm
posting two new big patches for src and gcc. These patches apply
against the latest src and gcc development sources and contain all of
the work to date.

The patches also include stdarg and trampoline support, my lazy gdb
port, and a few simulator fixes (like sta.b should actually store a
byte, not a word!). This brings the gcc testsuite's unexpected error
count down to 315. Looking good!

A quick scan of the remaining test failures tells me that something is
amiss with passing and returning aggregate types.

I've also been thinking about how to measure results of ISA
changes. In a perfect world, EEMBC benchmarks would be freely
available for all to use. EEMBC provides a terrific collection of
benchmarks that are representative of different industries, and are
all designed to build and run on bare metal embedded targets.

There's a Free work-alike to EEMBC called MiBench. The problem with
the MiBench benchmarks is that they aren't designed to run on bare
metal systems. For instance, they use file I/O to read and write
benchmark data. So right now I think I have three options:

* Modify MiBench to work more like EEMBC and run on bare metal.
* Enhance my libgloss stubs to fake file I/O by using the host filesystem.
* Port a simple OS like eCos to ggx, and then run that on qemu (using src/sim as the simulation core).

I don't know which way to go yet. Suggestions?

Patches:
* http://github.com/atgreen/ggx/blob/master/ggx-21-src.patch
* http://github.com/atgreen/ggx/blob/master/ggx-21-gcc.patch


FIN.


The Adventure Continues at http://github.com/atgreen/moxiedev !

